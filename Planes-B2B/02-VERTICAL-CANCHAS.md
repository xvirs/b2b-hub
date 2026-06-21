# Plan de Desarrollo 02 — Reservas de Canchas (Pádel / Fútbol 5) con Señas por Mercado Pago

> Sistema de reservas de canchas por WhatsApp con seña anticipada online, para complejos deportivos de Córdoba. El dueño deja de contestar "¿está libre la cancha a las 21?" y los no-shows del horario pico desaparecen porque sin seña no hay reserva.
> **Audiencia**: IA/dev sin contexto. Prerrequisito: motor de `01-NUCLEO-MOTOR.md` implementado hasta Fase 5 (completo). Hereda stack y convenciones de `00-INDICE.md` §Convenciones — acá se documenta solo el DELTA.

## 1. Qué se construye (resumen ejecutivo)

- El **complejo** (tenant) administra canchas, precios por franja horaria y reservas desde el panel del motor.
- El **cliente final** reserva por WhatsApp: deporte → duración → día → hora/cancha → paga la seña por Mercado Pago (Checkout Pro) → reserva confirmada. Sin seña pagada en 30 min, la reserva expira y el slot se libera.
- **Turnos fijos semanales** ("los martes 21h cancha 2 para Juan") con generación rolling de las próximas N semanas.
- **Lista de espera** para horarios pico: ante una cancelación, el sistema ofrece el slot por WhatsApp a los anotados, en orden, con 15 min para aceptar cada uno.
- Recordatorio **3h antes** del partido (configurable), en lugar del 24h+2h de salones.
- Módulo de pagos como paquete **`core-payments`** reutilizable por otros verticales (gimnasios, etc.).

**NO se construye**: facturación AFIP, split de pago entre jugadores, pago del saldo restante online (el saldo se cobra en el mostrador), torneos/ligas, ranking de jugadores, alquiler de paletas/pecheras como ítems, app móvil, multi-sucursal, devoluciones parciales.

## 2. Delta sobre el motor

| Módulo del motor | Uso en este vertical |
|---|---|
| `core-db` | Se EXTIENDE: tablas nuevas de §3, columnas nuevas en `appointments` |
| `core-whatsapp` | Se reusa tal cual; plantillas nuevas de §4 |
| `core-bot` (FSM) | Se EXTIENDE: flujo de reserva nuevo (deporte/duración en vez de servicio/staff) + estados de pago y lista de espera |
| `core-scheduling` | Se EXTIENDE: el recurso reservable pasa de `staff` a `court`; slots discretizados cada 30 min |
| `core-jobs` | Se EXTIENDE: jobs de expiración de pago, generación rolling de fijos, cascada de lista de espera |
| Panel web | Se EXTIENDE: pantallas de §5; la agenda muestra columnas por cancha |
| `core-payments` | **100% NUEVO** — paquete agnóstico del vertical (Checkout Pro, webhook, refunds, idempotencia) |

Decisión estructural: **`staff` no se usa en este vertical** (queda vacía). El recurso reservable es `courts`. Los `services` del motor tampoco se usan: el "servicio" acá es la **duración de alquiler** (60/90/120 min) con precio por franja, modelado en `court_pricing`. `appointments` se reusa con columnas nuevas (no se crea tabla `bookings` aparte: los recordatorios, el message_log y la agenda del motor ya cuelgan de `appointments`).

## 3. Modelo de datos (delta)

Mismas convenciones del motor: UUID v7, `tenant_id` NOT NULL + índice, centavos ARS, E.164, timestamptz.

```
courts                              -- reemplaza el rol de staff
  id, tenant_id, name,              -- "Cancha 1", "Cancha Central"
  sport ('padel'|'futbol5'),
  indoor bool default false, active bool, sort_order

court_pricing                       -- precio por cancha + duración + franja
  id, tenant_id, court_id,
  duration_min int,                 -- 60 | 90 | 120 (las que el tenant active)
  band ('day'|'peak'),              -- franja diurna vs nocturna/pico (límite en settings)
  price_cents int
  -- UNIQUE (court_id, duration_min, band)

appointments                        -- columnas NUEVAS sobre la tabla del motor
  + court_id uuid FK courts,        -- staff_id y service_id quedan NULL en este vertical
  + duration_min int,
  + price_cents int,                -- precio total congelado al reservar
  + deposit_cents int,              -- seña congelada al reservar
  + payment_id uuid FK payments nullable,
  + recurring_booking_id uuid FK recurring_bookings nullable,
  + hold_expires_at timestamptz nullable   -- solo en estado awaiting_payment
  -- status agrega valor 'awaiting_payment' al enum del motor
  -- la EXCLUDE constraint de solapamiento pasa a ser por court_id,
  --   con status NOT IN ('cancelled') — awaiting_payment SÍ bloquea (retención temporal)

payments                            -- paquete core-payments, agnóstico del vertical
  id, tenant_id, provider ('mercadopago'),
  amount_cents int, currency ('ARS'),
  status ('pending'|'approved'|'rejected'|'expired'|'refunded'|'forfeited'),
  external_reference,               -- = appointments.id (cómo MP nos devuelve la referencia)
  mp_preference_id, mp_payment_id nullable, mp_init_point text,  -- URL de pago
  refund_id nullable, paid_at nullable, refunded_at nullable

payment_events                      -- log crudo de webhooks de MP, base de la idempotencia
  id, tenant_id nullable, payment_id nullable,
  provider_event_id,                -- UNIQUE (provider, provider_event_id)
  type, payload jsonb, processed_at nullable

recurring_bookings                  -- turnos fijos semanales
  id, tenant_id, customer_id, court_id,
  weekday int (0-6), start_time time, duration_min int,
  price_cents int,                  -- precio pactado (puede diferir del de lista)
  status ('active'|'ended'),
  ends_on date nullable,            -- null = sin fin definido
  generated_until date              -- hasta qué fecha hay appointments generados

waitlist
  id, tenant_id, customer_id,
  sport, date, time_from time, time_to time,     -- "me anoto para pádel el jueves entre 20 y 23"
  status ('waiting'|'offered'|'accepted'|'expired_offer'|'cancelled'|'fulfilled'),
  offered_appointment_id nullable, offer_expires_at nullable,
  created_at  -- el orden de la cola es created_at ASC
```

Integridad clave: al pasar un appointment a `cancelled` o expirar un `awaiting_payment` → cancelar reminders pendientes (regla del motor) **y disparar la cascada de lista de espera** (§6.5). `reminders.kind` agrega valor `'3h'`.

## 4. Flujos de WhatsApp (delta)

### Plantillas nuevas a aprobar (categoría utility, crear al inicio de la Fase 2)

- `sena_pendiente`: "Hola {{1}}! Reservamos {{2}} el {{3}} a las {{4}} ({{5}} min). Para confirmar, pagá la seña de ${{6}} acá: {{7}} ⏰ Tenés {{8}} minutos, después el horario se libera." (sin botones; {{7}} = link de Checkout Pro)
- `reserva_confirmada`: "Listo {{1}}! Seña recibida ✅ Te esperamos en {{2}} el {{3}} a las {{4}}, {{5}}. Saldo a abonar en el complejo: ${{6}}."
- `recordatorio_partido`: "Hola {{1}}! Hoy a las {{2}} tenés {{3}} en {{4}} ({{5}}). Te esperamos 💪" + botón quick-reply: «Cancelar reserva»
- `slot_liberado`: "Hola {{1}}! Se liberó {{2}} el {{3}} a las {{4}} ({{5}} min, ${{6}}). Estás en la lista de espera: ¿lo querés?" + botones: «Sí, lo quiero» / «No, gracias» — "Tenés 15 minutos antes de que pase al siguiente."
- `fijo_por_vencer`: "Hola {{1}}! Tu turno fijo de los {{2}} a las {{3}} en {{4}} termina el {{5}}. Avisanos si querés renovarlo 🙌"

### FSM — flujo de reserva (reemplaza al Flujo B del motor para `vertical = 'cancha'`)

```
idle → menú principal [Reservar cancha | Mis reservas | Lista de espera | Hablar con el complejo]
awaiting_sport:     botones con los deportes que tengan canchas activas ("¿Qué jugás? 🎾⚽")
                    (si el tenant tiene un solo deporte, se saltea)
awaiting_duration:  botones con las duraciones activas: "60 min" / "90 min" / "120 min"
awaiting_day:       lista: próximos 7 días con al menos un slot libre
awaiting_slot:      lista: horarios libres del día; cada ítem = hora + cancha + precio
                    ("21:00 — Cancha 2 — $24.000"); máx 10, paginar con "Ver más horarios"
confirming:         resumen + seña + botones [Confirmar y pagar seña | Cambiar | Salir]
awaiting_payment:   crea appointment (status awaiting_payment, hold 30 min), crea preference
                    de MP y manda el link: "Te paso el link para la seña de $12.000 👉 {link}
                    El horario queda retenido 30 minutos."
                    → webhook aprueba → confirmed + plantilla reserva_confirmada
                    → expira → "Se venció el tiempo de pago y el horario se liberó 😞
                       Si todavía querés jugar, arrancá de nuevo cuando quieras."
```

Reglas: el estado conversacional `awaiting_payment` NO bloquea al cliente — cualquier mensaje suyo responde con el link de pago vigente y el tiempo restante. "Mis reservas" lista las futuras con botón de cancelar (aplica política de seña de §6.4, avisando ANTES de confirmar: "Si cancelás ahora, la seña no se devuelve. ¿Confirmás?"). "Lista de espera": deporte → día → franja horaria (botones "19-21" / "21-23" / "Cualquiera") → alta en `waitlist`.

## 5. Panel web (delta)

1. **Agenda** (modificada): columnas por **cancha** en vez de profesional; color extra para `awaiting_payment` (naranja, con countdown). Crear reserva manual permite marcar "seña cobrada en mostrador" (crea `payments` manual aprobado) o "sin seña".
2. **Canchas** (nueva, reemplaza "Equipo"): ABM de canchas + editor de la grilla de precios (`court_pricing`): matriz duración × franja por cancha, con copiar precios entre canchas.
3. **Turnos fijos** (nueva): ABM de `recurring_bookings`, vista de próximas generaciones, conflictos detectados, fecha de fin.
4. **Lista de espera** (nueva): cola por día con estado de cada oferta; acción manual "ofrecer ahora".
5. **Pagos** (nueva): listado de `payments` con filtros, totales de señas del mes, botón de reembolso manual.
6. **Configuración** (extendida): franja pico (hora inicio/fin), % de seña, minutos de hold, política de devolución, horas del recordatorio, duraciones activas.

## 6. Lógica de negocio específica

### 6.1 Precios por franja
- El tenant define UNA franja pico por día de semana en settings (`peak`, default 18:00–23:59 todos los días). Todo lo demás es `day`. Las canchas abren/cierran según `open_time`/`close_time` de settings (default 09:00–00:00).
- El precio de una reserva se resuelve por la **hora de inicio**: si arranca dentro de la franja pico, paga precio `peak` completo aunque termine fuera (decidido — sin prorrateo).
- `price_cents` y `deposit_cents` se **congelan en el appointment** al reservar: un cambio de precios posterior no afecta reservas existentes.
- Seña = `deposit_pct` (default 50%) sobre el precio, redondeada hacia arriba al múltiplo de $100 (10.000 centavos).
- Slots discretizados cada **30 min** (no 15 como salones). `getAvailableSlots` se generaliza: el recurso es `court` filtrado por `sport`, la duración viene del flujo, los `schedule_blocks` del motor se reusan con `court_id` en lugar de `staff_id` (renombrar la columna a `resource_id` en una migración del motor está PROHIBIDO — se agrega `court_id nullable` a `schedule_blocks`).

### 6.2 Retención temporal del slot durante el pago
- Al confirmar en el bot: se crea el appointment en `awaiting_payment` con `hold_expires_at = now() + payment_window_min` (default 30). La EXCLUDE constraint lo incluye, así que **el slot queda retenido**: nadie más puede tomarlo mientras el hold viva.
- Job en Trigger.dev programado a `hold_expires_at`: si el payment sigue sin aprobar → appointment `cancelled` (cancelled_by `system`), payment `expired`, y la preferencia de MP se invalida vía `expiration_date_to`. El slot vuelve a estar libre.
- Máximo **1 reserva `awaiting_payment` simultánea por cliente** (anti-acaparamiento): si intenta otra, el bot le recuerda el pago pendiente.

### 6.3 Webhook de Mercado Pago (en `core-payments`)
- Endpoint `POST /api/webhooks/mercadopago`. Responder 200 en <5s y **encolar**; nunca procesar inline.
- Validar firma `x-signature` (HMAC con el secret del webhook de MP) antes de encolar.
- **Idempotencia**: insertar en `payment_events` con UNIQUE `(provider, provider_event_id)`; violación de unique = evento duplicado, descartar silenciosamente. MP reintenta y duplica — esto es la primera barrera; la segunda es que las transiciones de estado de `payments` son monótonas (un `approved` nunca retrocede).
- **Nunca confiar en el body del webhook**: el job toma el `payment.id` notificado y consulta `GET /v1/payments/{id}` a la API de MP; el estado real sale de ahí. Mapeo: `approved` → payment `approved` + appointment `confirmed` + plantilla `reserva_confirmada` + programar recordatorio 3h; `rejected`/`cancelled` → payment `rejected`, el hold sigue vivo hasta expirar (el cliente puede reintentar con el mismo link); `pending`/`in_process` → no hacer nada (esperar evento siguiente).
- **Carrera pago-tardío**: si llega un `approved` con el appointment ya expirado → si el slot sigue libre, revivirlo a `confirmed` (caso feliz); si ya lo tomó otro, **refund automático total** vía `POST /v1/payments/{id}/refunds` + mensaje: "Uy, el pago entró tarde y el horario ya se ocupó 😞 Te devolvimos la seña completa. ¿Buscamos otro horario?".

### 6.4 Política de devolución de seña por cancelación (configurable por tenant)
- Setting `refund_cutoff_hours` (default 24). Cliente cancela con **más** de N horas de anticipación → refund total automático por API de MP, payment `refunded`. Con **menos** de N horas (o no-show) → el tenant retiene la seña, payment `forfeited`.
- El bot SIEMPRE anticipa la consecuencia antes de confirmar la cancelación. El panel permite refund manual discrecional (override del dueño) sobre cualquier payment `approved`/`forfeited` no vencido.
- Cancelación por parte del **complejo** (lluvia, mantenimiento) → refund total siempre, sin importar la anticipación, + disparo de lista de espera NO (el slot queda bloqueado con un `schedule_block`).

### 6.5 Lista de espera
- Se dispara cuando un appointment `confirmed` pasa a `cancelled` y su horario de inicio cae dentro de una franja pico futura (>2h de anticipación; con menos, no da el tiempo de la cascada).
- Cascada: tomar de `waitlist` los `waiting` cuyo (sport, date, franja) matchee, orden `created_at ASC`. Al primero: plantilla `slot_liberado`, status `offered`, `offer_expires_at = now() + 15 min`, job de expiración.
- «Sí, lo quiero» → entra al flujo de pago normal (§6.2): appointment `awaiting_payment` + link de seña; waitlist `accepted` (y `fulfilled` cuando paga). «No, gracias» o expiración → siguiente de la cola. Cola vacía → fin, el slot queda libre para el público.
- Mientras hay una oferta activa, el slot está retenido por el `offered` (se chequea antes de ofrecer y el appointment `awaiting_payment` del aceptante lo retiene después). Si el aceptante no paga, la cascada CONTINÚA con el siguiente.

### 6.6 Turnos fijos semanales
- Job diario (03:00 del tenant): para cada `recurring_bookings` activo, generar appointments `confirmed` (source `recurring`, **sin seña** — ver abajo) hasta `today + rolling_weeks` (default 4), avanzando `generated_until`. Conflicto con una reserva existente → NO generar esa ocurrencia y alertar en el panel.
- Los fijos **no pagan seña** por default (relación de confianza, pagan en el complejo); configurable con `recurring_requires_deposit` — si está activo, cada ocurrencia nace `awaiting_payment` con hold de 48h y link enviado 7 días antes.
- `ends_on` definido y a ≤14 días → plantilla `fijo_por_vencer` al cliente (una sola vez) + alerta en panel. Cancelar UNA ocurrencia no toca la recurrencia; terminar la recurrencia cancela las ocurrencias futuras ya generadas (las pagadas se reembolsan según §6.4).
- El recordatorio 3h aplica también a los fijos.

### 6.7 Recordatorios
- `reminder_hours` del tenant pasa a default `{3}` para este vertical. Resto de la mecánica = motor (Fase 3 de 01). El botón «Cancelar reserva» del recordatorio aplica §6.4 con confirmación previa.

## 7. Fases de implementación

Prerrequisito: motor completo (01 Fases 0-5). **Total estimado: 4-6 semanas.**

### Fase C0 — Schema + canchas en panel (3-4 días)
Migraciones de §3 completas, seed (1 complejo demo, 3 canchas pádel + 2 F5, grilla de precios day/peak), pantalla Canchas con grilla de precios, agenda por canchas con creación manual (sin pagos aún), settings del vertical (§8) con su Zod schema.
✅ *Criterios*: migraciones aplican sobre una DB del motor sin romper salones; agenda muestra columnas por cancha; reserva manual valida solapamiento por cancha (test de la EXCLUDE nueva); grilla de precios editable desde el celular.

### Fase C1 — Disponibilidad + bot sin pagos (1 semana)
`core-scheduling` generalizado a courts (slots de 30 min, filtro por deporte y duración, tests de bordes: cancha ocupada parcial, duración que no entra, franja que cruza medianoche). FSM nueva completa de §4 detrás del vertical, terminando en reserva `confirmed` directa (flag `payments_enabled = false`). Recordatorio 3h.
✅ *Criterios*: reserva completa por WhatsApp deporte→hora en <60s; tests FSM cubren paginación de slots, carrera de slot robado, TTL; recordatorio 3h llega y el botón cancelar funciona.

### Fase C2 — `core-payments` + Mercado Pago (1,5-2 semanas) ← **corazón del vertical**
Paquete `core-payments`: creación de preference (Checkout Pro con `external_reference`, `expiration_date_to`, `back_urls`, `notification_url`), webhook con firma+idempotencia+encolado, consulta de payment, refunds. Integración: hold de 30 min, job de expiración, plantillas `sena_pendiente`/`reserva_confirmada`, flujo §6.2-6.4 completo incluida la carrera de pago tardío. Pantalla Pagos. Todo contra el **sandbox de MP** (cuentas de prueba comprador/vendedor).
✅ *Criterios*: e2e en sandbox — reservar por WhatsApp → pagar el link con tarjeta de prueba → llega `reserva_confirmada` y el panel muestra verde en <30s del pago; no pagar → a los 30 min el slot reaparece libre en el bot; webhook duplicado no produce doble transición (test); cancelar con >24h → refund visible en la cuenta de prueba; cancelar con <24h → payment `forfeited` y el bot lo avisó antes; firma inválida → 401.

### Fase C3 — Turnos fijos + lista de espera (1-1,5 semanas)
`recurring_bookings` con job de generación rolling, conflictos, vencimiento y pantalla. `waitlist` con cascada de ofertas (plantilla `slot_liberado`, ventana 15 min, jobs de expiración) y pantalla.
✅ *Criterios*: crear fijo → aparecen 4 ocurrencias futuras; el job diario genera la 5ta al avanzar la fecha (test con clock simulado); conflicto genera alerta y no pisa; cancelar una reserva pico con 2 anotados → oferta al 1ro, expira a los 15 min, oferta al 2do, acepta, paga seña, queda `confirmed` (e2e demostrado en sandbox).

### Fase C4 — Pulido + producción (1 semana)
Métricas del vertical (ocupación por franja, señas cobradas/perdidas/devueltas del mes, tasa de no-show), credenciales de MP **por tenant** (cifradas en DB como las de WhatsApp, vía OAuth de MP o access token manual — decidir según §9), modo producción de MP, revisión de seguridad (firma de webhook, RLS en tablas nuevas), runbook de onboarding de un complejo nuevo (alta WhatsApp + alta MP + carga de canchas/precios paso a paso).
✅ *Criterios*: complejo real con número y cuenta de MP reales procesando una seña de verdad; checklist de seguridad pasada; runbook escrito y probado con un onboarding completo.

## 8. Configuración del tenant

`tenants.settings` para `vertical = 'cancha'`:

```ts
const canchaSettingsSchema = z.object({
  openTime: z.string().default('09:00'),          // HH:mm hora local del tenant
  closeTime: z.string().default('00:00'),         // 00:00 = cierra a medianoche
  peakStart: z.string().default('18:00'),
  peakEnd: z.string().default('23:59'),
  peakWeekdays: z.array(z.number().min(0).max(6)).default([0,1,2,3,4,5,6]),
  activeDurations: z.array(z.enum(['60','90','120'])).default(['60','90']),
  depositPct: z.number().min(0).max(100).default(50),
  paymentWindowMin: z.number().default(30),       // hold de la seña
  refundCutoffHours: z.number().default(24),      // §6.4
  rollingWeeks: z.number().default(4),            // turnos fijos
  recurringRequiresDeposit: z.boolean().default(false),
  paymentsEnabled: z.boolean().default(true),     // false = Fase C1 / tenant sin MP
  reminderHours: z.array(z.number()).default([3]),
});
```

Credenciales de MP por tenant (cifradas en DB, no en settings jsonb): `mp_access_token`, `mp_webhook_secret`, `mp_user_id`.

## 9. Puntos a verificar al implementar

- **API vigente de Mercado Pago**: Checkout Pro (`POST /checkout/preferences` — campos `expiration_date_to`, `external_reference`, `notification_url`, `back_urls`), formato actual del webhook (tópico `payment`, validación de `x-signature`), `GET /v1/payments/{id}` y `POST /v1/payments/{id}/refunds`. Confirmar contra la doc oficial: los nombres de campos y la mecánica de firma cambiaron varias veces.
- **Tarifas de MP vigentes** para Checkout Pro en Argentina (comisión por cobro según plazo de liberación) — impacta cuánta seña conviene pedir y hay que poder explicárselo al dueño en el onboarding.
- **Modalidad de alta de vendedor**: si OAuth de MP (marketplace) sigue siendo viable para operar a nombre del tenant, o si conviene access token manual por tenant en el MVP. Decidir en Fase C4 con la doc vigente.
- **Plantillas utility de WhatsApp con links**: verificar política vigente de Meta sobre URLs variables en plantillas (la URL del Checkout Pro va como variable — puede requerir dominio fijo con redirect propio, ej. `https://app.dominio.com/pay/{id}`). Si Meta lo exige, construir el redirect: es trivial y además loguea el click.
- Tarifas vigentes de la Cloud API (hereda del motor §9).

## 10. Extensiones previstas (NO implementar)

- **Pago del saldo total online** (no solo la seña) — `payments` ya soporta montos arbitrarios.
- **Split de seña entre jugadores** (cada uno paga su parte) — requeriría múltiples payments por appointment; la FK actual es 1:1, migrar a tabla puente cuando llegue.
- **Precios dinámicos** (descuento por horario muerto) — `court_pricing` admitiría más bandas.
- **Buffet/kiosco y alquiler de equipamiento** como ítems del ticket.
- **Reservas web** (página pública del complejo además del bot) — `core-scheduling` y `core-payments` ya quedan listos para servirla.
- `core-payments` para el **vertical gimnasios (plan 04)**: cuotas recurrentes — por eso el paquete no asume nada de "seña" ni de appointments en sus tipos internos.
