# Plan de Desarrollo 04 — Vertical Gimnasios y Estudios

> Cuotas, morosidad y cupos por clase para gimnasios y estudios boutique (yoga, pilates, crossfit) de Córdoba, Argentina.
> **Audiencia**: IA/dev sin contexto. Prerrequisito: motor de `01-NUCLEO-MOTOR.md` implementado hasta Fase 5, y paquete `core-payments` del plan `02-VERTICAL-CANCHAS.md`.
>
> **Dependencia de core-payments**: este vertical REUTILIZA el módulo de Mercado Pago definido en el plan 02. Si al implementar este plan el 02 no está construido, alcanza con su interfaz mínima: creación de preferencia de pago Checkout Pro (`createPaymentPreference()`), webhook idempotente de notificaciones de MP, y registro de cada pago en la tabla `payments`. Este documento asume esa interfaz y no la redefine.

## 1. Qué se construye (resumen ejecutivo)

El dolor de este rubro NO son los turnos: es la **cobranza**. Cuotas vencidas controladas en Excel y persecución manual de morosos por WhatsApp. Secundariamente, cupos por clase anotados en papel. El producto reduce la morosidad de forma medible — esa es la métrica de venta.

1. **Membresías y cuotas**: planes mensuales (X clases/semana o ilimitado), membresías por socio con día de vencimiento, registro de pagos por Mercado Pago (link en el WhatsApp) **y** en efectivo en mostrador — ambos caminos son obligatorios: en Argentina mucha gente paga efectivo.
2. **Escalera de cobranza automática por WhatsApp**: aviso N días antes del vencimiento con link de pago, aviso al vencer, aviso a los N días de vencida. Configurable por tenant. Pago acreditado por webhook → membresía renovada sola + mensaje de confirmación. Cero persecución manual.
3. **Cupos por clase**: clases recurrentes semanales con capacidad; el socio reserva su lugar por WhatsApp con el bot del motor. Solo reservan miembros con membresía activa. Lista de espera cuando la clase se llena.
4. **Check-in de mostrador**: pantalla del panel que busca al socio por nombre/teléfono y muestra verde (al día) / rojo (vencida). Nada de hardware.
5. **Métricas de morosidad**: tasa de morosidad mes a mes, recupero por avisos automáticos (pagos dentro de 72h de un aviso), ocupación por clase.

**NO se construye**: rutinas de entrenamiento, seguimiento físico/antropometría, app del socio, facturación AFIP, integración con molinetes o cualquier hardware de acceso, planes anuales/semestrales (solo mensual en MVP), débito automático de tarjeta (el link de MP por WhatsApp cumple ese rol).

## 2. Delta sobre el motor

| Módulo del motor | Uso en este vertical |
|---|---|
| `core-db` (tenants, users, customers, staff, message_log, conversations) | Se reusa tal cual. `customers` = socios. `staff` = profesores. `tenants.vertical = 'gym'` |
| `core-whatsapp` | Se reusa tal cual (3 plantillas nuevas a aprobar, §4) |
| `core-bot` (FSM) | Se extiende: flujo nuevo de reserva de clase + flujo de consulta de cuota |
| `core-scheduling` | NO se usa para clases (las clases tienen horario fijo y capacidad, no slots por duración). Queda disponible si el gym ofrece turnos 1:1, fuera del MVP |
| `core-jobs` (Trigger.dev) | Se extiende: jobs de la escalera de cobranza + materialización de clases + lista de espera |
| `core-payments` (plan 02) | Se reusa tal cual: preferencia Checkout Pro, webhook idempotente, tabla `payments` |
| `appointments` / `reminders` | NO se usan en este vertical (no hay turnos individuales en el MVP) |
| **100% nuevo** | Paquete `core-memberships`: planes, membresías, ciclo de cuotas, escalera de cobranza. Módulo de clases con cupos y lista de espera. Pantalla de check-in. Métricas de morosidad |

`core-memberships` sigue el principio rector del motor: la lógica de ciclo de vida de la cuota (vencimientos, proración, pausas, transiciones de estado) es pura y testeable sin I/O; los efectos (enviar plantilla, crear preferencia MP) los ejecuta la capa exterior.

## 3. Modelo de datos (delta)

Mismas convenciones del motor: PK UUID v7, `tenant_id` NOT NULL + índice, `created_at`/`updated_at` timestamptz, centavos ARS integer, E.164.

```
plans                                -- planes de membresía del gym
  id, tenant_id, name,               -- "Pilates 2x semana", "Libre"
  price_cents int,                   -- precio mensual
  weekly_classes int nullable,       -- clases por semana; NULL = ilimitado
  active bool, sort_order

memberships
  id, tenant_id, customer_id, plan_id,
  status ('active'|'overdue'|'paused'|'cancelled'),
  billing_day int (1-28),            -- día de vencimiento mensual; se fija al alta
  current_period_end date,           -- hasta cuándo está paga
  paused_at date nullable, pause_reason text nullable,
  cancelled_at timestamptz nullable
  -- UNIQUE parcial: un customer no puede tener 2 memberships en estado active/overdue/paused

membership_payments                  -- cada cuota cobrada, por cualquier vía
  id, tenant_id, membership_id,
  amount_cents int,
  period_start date, period_end date,   -- qué mes cubre
  method ('mp'|'cash'),
  payment_id nullable FK → payments,    -- de core-payments; NOT NULL si method='mp'
  registered_by nullable FK → users,    -- quién lo cargó; NOT NULL si method='cash'
  paid_at timestamptz
  -- UNIQUE (membership_id, period_start): una cuota no se cobra dos veces

dunning_notices                      -- cada aviso de la escalera enviado (para medir recupero)
  id, tenant_id, membership_id,
  kind ('pre_due'|'due'|'overdue'),
  sent_at timestamptz, trigger_run_id,
  mp_preference_id text nullable     -- preferencia generada para el link de este aviso

class_sessions                       -- clase recurrente semanal (la definición, no la instancia)
  id, tenant_id, discipline text,    -- "Yoga", "WOD", "Pilates reformer"
  staff_id FK → staff,               -- profesor
  weekday int (0-6), start_time time, duration_min int,
  capacity int, active bool

class_occurrences                    -- instancia concreta de una clase en una fecha
  id, tenant_id, class_session_id, date date,
  starts_at timestamptz,             -- desnormalizado para queries
  capacity_override int nullable,    -- ajuste puntual del día
  cancelled bool default false
  -- UNIQUE (class_session_id, date). Materializadas por job con 7 días de anticipación

class_bookings
  id, tenant_id, class_occurrence_id, customer_id,
  status ('booked'|'cancelled'|'attended'|'no_show'),
  source ('bot'|'panel'), cancelled_at timestamptz nullable
  -- UNIQUE parcial (class_occurrence_id, customer_id) WHERE status='booked'
  -- El cupo se valida en transacción: COUNT(booked) < capacity con lock sobre la occurrence

class_waitlist
  id, tenant_id, class_occurrence_id, customer_id,
  position int, status ('waiting'|'offered'|'converted'|'expired'),
  offered_at timestamptz nullable, offer_expires_at timestamptz nullable
```

Reglas de integridad clave:
- `memberships.status` se deriva SIEMPRE de `current_period_end` por el job diario (§6) — nunca se edita a mano salvo pausa/cancelación desde el panel.
- Registrar un `membership_payment` (cualquier método) → recalcular `current_period_end` y status en la misma transacción.
- Cancelar una `class_occurrence` → cancelar sus bookings y avisar por WhatsApp a los anotados (plantilla de texto dentro de ventana si está abierta; si no, plantilla `clase_cancelada` — ver §10, en MVP el aviso lo manda el gym a mano y el panel lista los teléfonos).

## 4. Flujos de WhatsApp (delta)

### Plantillas nuevas (categoría utility, enviar a aprobación en Fase 1 — tardan 24-48h)

**`cuota_por_vencer`** — aviso pre-vencimiento (N días antes, default 3):
> Hola {{1}}! Te recordamos que tu cuota de {{2}} ({{3}}) vence el {{4}}. Podés pagarla ahora desde este link y te queda renovada al toque 💪
- Variables: {{1}} nombre, {{2}} nombre del gym, {{3}} nombre del plan, {{4}} fecha de vencimiento.
- Botones: [Pagar ahora] (URL → link Checkout Pro) · [Ya pagué en el gym] (quick-reply).

**`cuota_vencida`** — el día del vencimiento sin pago registrado:
> Hola {{1}}! Tu cuota de {{2}} venció hoy. Para seguir reservando tus clases sin drama, podés pagarla desde acá 👇
- Variables: {{1}} nombre, {{2}} nombre del gym.
- Botones: [Pagar ahora] (URL) · [Ya pagué en el gym] (quick-reply).

**`cuota_recuperacion`** — a los N días de vencida (default 7):
> Hola {{1}}! Hace {{2}} días que venció tu cuota de {{3}}. Te extrañamos en las clases 🙂 Si querés seguir entrenando, regularizala desde este link o pasá por recepción.
- Variables: {{1}} nombre, {{2}} días vencida, {{3}} nombre del gym.
- Botones: [Pagar ahora] (URL) · [Hablar con el gym] (quick-reply).

Quick-replies: "Ya pagué en el gym" → crea alerta en el panel para que recepción registre el pago en efectivo (NO renueva sola la membresía); "Hablar con el gym" → handoff del motor (bot silenciado 4h, alerta en panel).

### Mensajes de sesión (dentro de ventana, texto libre del bot)

- Confirmación de pago MP acreditado: "¡Listo, {nombre}! Recibimos tu pago 🎉 Tu cuota queda al día hasta el {fecha}. Nos vemos en clase 💪"
- Confirmación de pago efectivo registrado en panel (si hay ventana abierta; si no, no se envía nada): mismo texto.

### Cambios a la FSM (`core-bot`)

Menú principal del vertical gym: **[Reservar clase | Mis reservas | Mi cuota]** (reemplaza al menú de salones vía config por vertical).

Flujo C — Reserva de clase (estados nuevos):
```
awaiting_discipline:  lista de disciplinas con clases activas
awaiting_day:         lista de próximos 7 días con occurrences de esa disciplina
awaiting_class_time:  lista de horarios del día con "(quedan N lugares)" o "(completa — lista de espera)"
confirming_class:     resumen + botones [Confirmar | Cambiar | Cancelar]
done:                 crea class_booking → "✅ ¡Anotado/a! {disciplina} el {día} a las {hora} con {profe}."
```
Guardas del flujo:
- Al entrar al flujo, si `membership.status != 'active'`: comportamiento según `settings.booking_when_overdue` (§6.3) — bloquear con mensaje + link de pago, o dejar pasar con aviso.
- Clase llena → botón [Anotarme en lista de espera] → crea `class_waitlist` con `position` siguiente → "Quedaste {N}º en la lista de espera. Si se libera un lugar te avisamos por acá."
- Carrera de cupo: si al confirmar la clase se llenó, "Uy, se acaba de llenar 😞" + ofrecer lista de espera.
- "Mis reservas": lista de bookings futuros con botón cancelar cada uno. Cancelar → libera cupo → dispara oferta a lista de espera si aplica (§6.5).
- "Mi cuota": estado de la membresía, fecha de vencimiento y, si está vencida o por vencer, botón [Pagar ahora] con link MP fresco.

Oferta de lista de espera (mensaje de sesión si hay ventana; si no, plantilla `clase_liberada` — redactarla y aprobarla en Fase 1 junto a las otras):
> Se liberó un lugar en {{1}} el {{2}} a las {{3}}! Sos el/la primero/a de la lista. ¿Lo querés? Tenés {{4}} para confirmar.
- Botones: [Lo quiero] · [No, gracias].

## 5. Panel web (delta)

1. **Check-in** (nueva, home del vertical): buscador grande por nombre o teléfono, pensado para mostrador con apuro. Resultado: tarjeta del socio con semáforo **verde** (al día) / **rojo** (vencida) / amarillo (pausada), plan, vencimiento, y acciones de un toque: [Registrar pago en efectivo] y [Marcar asistencia a la clase de ahora] (si hay una occurrence en curso ±30 min). Sin hardware, sin molinetes.
2. **Socios** (extiende Clientes del motor): columna estado de membresía + filtro "vencidas". Detalle: plan, historial de `membership_payments`, historial de avisos enviados (`dunning_notices`), botones pausar/reanudar/cancelar membresía, asignar/cambiar plan.
3. **Planes** (nueva): ABM (nombre, precio, clases por semana o ilimitado, activo, orden).
4. **Clases** (nueva): grilla semanal de `class_sessions` (ABM: disciplina, profesor, día/hora, duración, capacidad). Vista del día con occurrences: anotados/capacidad, lista de espera, marcar asistencia/no-show por socio, cancelar la clase del día.
5. **Cobranza** (nueva): lista de membresías vencidas ordenada por días de atraso, con último aviso enviado y botón [Registrar pago efectivo]. Alertas pendientes de "Ya pagué en el gym". Configuración de la escalera (días y on/off por escalón).
6. **Métricas** (extiende la del motor): tasa de morosidad mes a mes (vencidas / activas+vencidas al cierre de cada mes), **recupero por avisos** (pagos dentro de 72h de un `dunning_notice`, en $ y en cantidad — la métrica de venta), ocupación por clase (asistencias / capacidad, por class_session).

## 6. Lógica de negocio específica

**6.1 Ciclo de la cuota.** Job diario (Trigger.dev, 08:00 hora del tenant) por cada membresía no pausada/cancelada: si `current_period_end < hoy` → status `overdue`. La escalera evalúa contra `billing_day` y días de atraso: pre_due a `pre_due_days` antes, due el día del vencimiento, overdue a `overdue_days` después — cada escalón se envía una sola vez por período (dedupe por `dunning_notices` kind+período) y solo si la cuota sigue impaga. Cada aviso con botón de pago genera su preferencia MP en el momento (con `external_reference = membership_id + period_start`) y la guarda en `dunning_notices.mp_preference_id`. `customers.opted_out` se respeta SIEMPRE: sin avisos, el socio aparece en Cobranza para gestión manual.

**6.2 Proración de alta a mitad de mes.** Al dar de alta una membresía el día D con `billing_day` B: la primera cuota cubre desde D hasta el próximo B y se cobra proporcional: `round(price_cents × días_cubiertos / 30)`, mínimo 25% del plan. Desde ahí, cuotas completas de B a B. Regla fija: B se elige al alta (default: el día del alta, cap a 28) y no se recalcula. La función `prorate(price_cents, from, billingDay)` vive en `core-memberships` con tests de bordes (alta el mismo día B, alta el 29-31, febrero).

**6.3 Membresía vencida y reservas.** Configurable por tenant (`booking_when_overdue`): `'block'` (default) — el bot no deja reservar y responde "Tu cuota está vencida 😕 Regularizala desde este link y reservás al toque 👇" con botón de pago; `'warn'` — deja reservar pero antepone "Ojo: tu cuota venció el {fecha}. Podés ponerte al día acá 👇". En ambos casos las reservas YA hechas no se cancelan al vencer la cuota — el gym decide en el mostrador (el check-in la muestra roja).

**6.4 Pausas (vacaciones/lesión).** Solo desde el panel. Pausar → status `paused`, se detiene la escalera y se bloquean reservas nuevas (bot: "Tu membresía está pausada. Pasá por el gym para reactivarla."). Reanudar → `current_period_end` se extiende por los días pausados (la pausa no consume cuota) y status recalculado. Máximo 1 pausa activa; duración libre (el panel muestra hace cuánto está pausada).

**6.5 Lista de espera.** Cuando se libera un cupo (cancelación por bot o panel) y faltan menos de `waitlist_window_hours` (default 12) para la clase: ofrecer al primero `waiting` con deadline `waitlist_offer_minutes` (default 60, cap a "hasta la hora de la clase"). [Lo quiero] → booking creado, `converted`; [No] o timeout (job programado) → `expired`, se ofrece al siguiente. Si falta MÁS que la ventana, el cupo queda libre para reserva normal (sin oferta dirigida). Solo un `offered` por occurrence a la vez.

**6.6 Conciliación efectivo vs MP.** Dos caminos, una sola verdad: `membership_payments`. Efectivo: recepción lo registra desde Check-in o Cobranza → fila con `method='cash'` y `registered_by`. MP: el webhook de core-payments acredita → fila con `method='mp'` y `payment_id`. El UNIQUE `(membership_id, period_start)` impide el doble cobro del período: si entra el webhook de MP y el período ya está pago en efectivo (pagó en mostrador con el link viejo abierto), el pago MP queda registrado en `payments` pero NO genera `membership_payment`; se crea alerta en el panel "pago duplicado — devolver por MP" (la devolución es manual en MVP). El quick-reply "Ya pagué en el gym" jamás acredita: solo alerta.

**6.7 Idempotencia del webhook MP.** core-payments ya dedupea notificaciones por id de pago de MP. Este vertical agrega su propia capa: el handler que convierte `payment` aprobado → renovación resuelve la membresía por `external_reference` y hace INSERT de `membership_payment` + UPDATE de `current_period_end` (+1 mes desde el máximo entre hoy y el vencimiento — pagar tarde no regala días, pagar temprano no los quita) en una transacción; el UNIQUE de período hace el reintento inocuo. El mensaje de confirmación se envía solo si el INSERT insertó.

**6.8 Asistencia y no-show de clase.** El profe o recepción marca asistencia desde la pantalla Clases (o Check-in). Job 2h después del fin de la occurrence: bookings aún `booked` → `no_show`. No hay penalidad automática en MVP (las métricas de ocupación lo exponen; penalidades en §10).

## 7. Fases de implementación

Prerrequisito verificado antes de la Fase 1: motor en producción (plan 01 Fase 5) y `core-payments` disponible (plan 02; si no existe, construir primero su interfaz mínima — preferencia Checkout Pro + webhook + tabla `payments` — estimando +1 semana NO incluida acá).

### Fase 1 — Membresías y pagos en panel (1,5 semanas)
Schema completo de §3 con migraciones y seed gym (1 tenant `vertical='gym'`, 3 planes, 10 socios, 6 class_sessions). `core-memberships` con ciclo de estados y proración (tests de §6.2). Pantallas Planes, Socios y registro de pago en efectivo. **Día 1: redactar y enviar a aprobación las 4 plantillas de §4** (cuota_por_vencer, cuota_vencida, cuota_recuperacion, clase_liberada).
✅ *Criterios*: alta de socio con plan y proración correcta demostrada por test + UI; registrar pago efectivo → `current_period_end` avanza y el doble registro del mismo período es imposible (test de la constraint); job diario marca `overdue` a la vencida (test con fecha simulada); plantillas en estado "en revisión" o aprobadas en Meta.

### Fase 2 — Escalera de cobranza + pago MP (1,5 semanas) ← **entregable que vende**
Jobs de la escalera completos con dedupe por período. Generación de preferencia MP por aviso. Webhook → renovación automática + confirmación (§6.7). Quick-replies "Ya pagué en el gym" y handoff. Pantalla Cobranza. Flujo "Mi cuota" en el bot.
✅ *Criterios*: e2e demostrable con fechas simuladas — membresía que vence en 3 días → llega `cuota_por_vencer` con link → pago sandbox MP → webhook acredita → membresía renovada sola + mensaje de confirmación, todo sin tocar el panel; reenvío del webhook no duplica nada (test); vencida sin pago recibe `cuota_vencida` el día D y `cuota_recuperacion` el D+7, una sola vez cada una; opt-out no recibe nada y aparece en Cobranza.

### Fase 3 — Clases y cupos por WhatsApp (1,5 semanas)
ABM de Clases + job de materialización de occurrences (7 días). Flujo C completo en `core-bot` con tests de FSM (guarda de membresía en ambos modos de §6.3, clase llena, carrera de cupo, cancelación). "Mis reservas". Lista de espera completa (§6.5) con job de expiración de ofertas.
✅ *Criterios*: reserva de clase de punta a punta por WhatsApp en <45s; socio `overdue` bloqueado o avisado según config (ambos modos testeados); clase llena → lista de espera → cancelación dentro de la ventana → oferta al 1º → timeout → oferta al 2º (e2e demostrable); imposible superar `capacity` bajo concurrencia (test de la transacción con lock).

### Fase 4 — Check-in, métricas y producción (1 semana)
Pantalla Check-in con semáforo y acciones de un toque. Asistencia/no-show (§6.8). Métricas de §5.6 (morosidad m/m, recupero 72h, ocupación). Alerta de pago duplicado (§6.6). Revisión de seguridad (RLS en tablas nuevas, links MP con external_reference verificado). Runbook de onboarding de un gimnasio nuevo: crear tenant, conectar WhatsApp, cargar planes/clases/socios (import CSV simple desde su Excel: nombre, teléfono, plan, vencimiento), activar escalera.
✅ *Criterios*: check-in busca y muestra semáforo en <2s con 500 socios cargados; las 3 métricas calculan correcto contra un dataset de prueba conocido (test); el recupero atribuye un pago al aviso solo si llegó dentro de las 72h (test de borde); runbook ejecutado completo con un tenant nuevo de prueba, Excel real importado incluido.

**Total estimado: 5,5 semanas** sobre el motor terminado (+1 semana si hay que construir la interfaz mínima de core-payments).

## 8. Configuración del tenant

`tenants.settings` para `vertical = 'gym'`:

```ts
const gymSettingsSchema = z.object({
  dunning: z.object({
    preDueDays: z.number().int().min(1).max(10).default(3),
    dueEnabled: z.boolean().default(true),
    overdueDays: z.number().int().min(1).max(30).default(7),
    enabled: z.boolean().default(true),          // apagado general de la escalera
  }),
  bookingWhenOverdue: z.enum(['block', 'warn']).default('block'),
  waitlist: z.object({
    windowHours: z.number().int().min(1).max(48).default(12),
    offerMinutes: z.number().int().min(15).max(240).default(60),
  }),
  classBookingOpenDays: z.number().int().min(1).max(14).default(7), // anticipación máxima de reserva
});
```

## 9. Puntos a verificar al implementar

- Interfaz real de `core-payments` si el plan 02 ya se implementó (nombres de funciones, shape de `payments`, manejo de `external_reference`) — este documento asumió la interfaz mínima del encabezado.
- Botones de URL dinámica en plantillas de Meta: hoy el botón URL acepta un sufijo variable (`https://.../pay/{{1}}`); confirmar límites vigentes y que el dominio corto propio esté configurado.
- Tarifas vigentes de plantillas utility en Argentina y de Checkout Pro (comisión MP) para el pricing del producto.
- Política de Meta sobre frecuencia de plantillas a un mismo número (la escalera manda hasta 3 por mes por socio — hoy holgado, confirmar).

## 10. Extensiones previstas (no implementar)

Débito automático con tarjeta (suscripciones MP), penalidad por no-show reiterado (configurable, ej. pierde prioridad de reserva), plantilla `clase_cancelada` con aviso masivo a anotados, packs de clases sueltas (clipcard) además del plan mensual, multi-sucursal, reportes exportables para el contador, campañas de recuperación de ex-socios. El diseño no los bloquea: `membership_payments.method` es extensible, `class_bookings.status` ya distingue no_show, y la escalera es genérica por `kind`.
