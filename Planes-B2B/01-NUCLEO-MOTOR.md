# Plan de Desarrollo 01 — Núcleo Motor + Vertical Salones

> Plataforma SaaS multi-tenant de gestión de turnos con bot de WhatsApp. Este documento especifica el MOTOR COMÚN (reutilizado por todos los verticales del portfolio) y su primer vertical: salones de belleza/peluquerías/barberías.
>
> **Audiencia de este documento**: una IA o desarrollador SIN contexto previo del proyecto. Todo lo necesario está acá. El contexto de negocio (mercado, pricing, competencia) está en `../Gestion Wssp/PRODUCTO.md` y no es necesario para implementar.

---

## 1. Qué se construye (resumen ejecutivo)

Un sistema donde:
1. El **salón** (tenant) administra desde un panel web su agenda: profesionales, servicios, horarios y turnos.
2. El **cliente final** interactúa SOLO por WhatsApp: recibe recordatorios de sus turnos con botones (Confirmo / Reprogramar / Cancelar) y puede reservar turnos nuevos mediante un flujo guiado de botones y listas (sin IA conversacional).
3. El sistema envía **recordatorios automáticos** programados (24h y 2h antes, configurable por tenant) y mantiene la agenda sincronizada con las respuestas del cliente.

NO se construye (fuera de alcance de este plan): app móvil nativa, IA conversacional libre, pagos/señas (plan 02), cuotas (plan 04), facturación AFIP, multi-sucursal, historia clínica.

---

## 2. Stack tecnológico (FIJO — no sustituir componentes)

| Capa | Tecnología | Notas |
|---|---|---|
| Framework | **Next.js 15+ (App Router)** + TypeScript `strict` | Panel web + API routes (webhooks y API interna) |
| UI | **shadcn/ui + Tailwind CSS** | Panel mobile-first: la dueña administra desde el celular |
| DB | **PostgreSQL** (Neon en prod, Docker local) | |
| ORM | **Drizzle ORM** + drizzle-kit para migraciones | Schema en `packages/core-db` |
| Jobs/colas | **Trigger.dev v3** | Recordatorios programados, envíos diferidos, reintentos |
| Mensajería | **WhatsApp Cloud API oficial (Meta)** | PROHIBIDO Baileys/whatsapp-web.js (riesgo de ban) |
| Auth panel | **Auth.js (NextAuth v5)** con email magic link + Google | Usuarios del salón, no clientes finales |
| Validación | **Zod** en todos los bordes (webhooks, forms, API) | |
| Hosting | Vercel (panel/API) + Neon (DB) + Trigger.dev cloud | Webhook de Meta exige HTTPS público |
| Tests | Vitest (unitario) + Playwright (e2e del panel) | La FSM del bot debe tener cobertura unitaria completa |

### Estructura de monorepo

```
turnero/
├── apps/
│   └── web/                 # Next.js: panel + API + webhooks
├── packages/
│   ├── core-db/             # Schema Drizzle + migraciones + seed
│   ├── core-whatsapp/       # Cliente Cloud API + parser de webhooks + envío de plantillas
│   ├── core-bot/            # FSM de conversaciones (pura, testeable, sin I/O)
│   ├── core-scheduling/     # Lógica de disponibilidad/slots (pura, testeable)
│   └── core-jobs/           # Definiciones de tareas Trigger.dev
├── docker-compose.yml       # Postgres local
└── CLAUDE.md                # Generado en Fase 0
```

**Principio rector**: `core-bot` y `core-scheduling` son funciones puras sin I/O (reciben estado, devuelven estado + efectos a ejecutar). Esto los hace testeables sin mocks de WhatsApp y reutilizables en todos los verticales.

---

## 3. Modelo de datos

Convenciones: PK `id` UUID v7; `created_at`/`updated_at` timestamptz en todas; `tenant_id` NOT NULL + índice en toda tabla de negocio; soft-delete (`deleted_at`) solo donde se indica; dinero en centavos ARS (integer); teléfonos E.164.

```
tenants
  id, name, slug, vertical ('salon'|'cancha'|'vet'|'gym'),
  timezone (default 'America/Argentina/Cordoba'),
  wa_phone_number_id, wa_business_account_id, wa_access_token (cifrado),
  reminder_hours int[] (default '{24,2}'),
  settings jsonb (extensión por vertical), status ('active'|'suspended')

users                              -- usuarios del PANEL (dueña, recepcionista)
  id, tenant_id, email, name, role ('owner'|'staff'), auth via Auth.js

staff                              -- profesionales que atienden
  id, tenant_id, name, active bool, sort_order

staff_schedules                    -- horario laboral semanal por profesional
  id, staff_id, weekday (0-6), start_time, end_time   -- múltiples bloques por día permitidos

schedule_blocks                    -- bloqueos puntuales (vacaciones, feriado, almuerzo)
  id, tenant_id, staff_id nullable (null = todo el salón), starts_at, ends_at, reason

services
  id, tenant_id, name, duration_min int, price_cents int, active bool, sort_order

staff_services                     -- qué profesional hace qué servicio (N:M)
  staff_id, service_id

customers                          -- cliente final, identificado por teléfono
  id, tenant_id, phone (E.164, unique por tenant), name nullable,
  opted_out bool default false, last_interaction_at

appointments
  id, tenant_id, customer_id, staff_id, service_id,
  starts_at, ends_at (timestamptz),
  status ('pending'|'confirmed'|'cancelled'|'no_show'|'completed'),
  source ('panel'|'bot'), cancelled_by ('customer'|'salon') nullable,
  notes text
  -- CONSTRAINT de exclusión: no solapar turnos del mismo staff
  -- (usar EXCLUDE USING gist con tstzrange, status NOT IN ('cancelled'))

conversations                      -- estado FSM del bot por cliente
  id, tenant_id, customer_id, state varchar,     -- ej: 'awaiting_service'
  context jsonb,                                  -- selecciones parciales del flujo
  expires_at                                      -- TTL 30 min; expirado = flujo reinicia

message_log                        -- TODO mensaje in/out, sin excepción
  id, tenant_id, customer_id nullable, direction ('in'|'out'),
  wa_message_id, type ('text'|'template'|'interactive'|...),
  payload jsonb, status ('sent'|'delivered'|'read'|'failed') nullable, error text

reminders                          -- recordatorios programados y su resultado
  id, appointment_id, scheduled_for, kind ('24h'|'2h'),
  trigger_run_id,                                 -- id del job en Trigger.dev (para cancelarlo)
  status ('scheduled'|'sent'|'cancelled'|'failed')
```

Reglas de integridad clave:
- Al cancelar/reprogramar un turno → cancelar sus `reminders` pendientes (vía `trigger_run_id`) y crear los nuevos si corresponde.
- `customers.opted_out = true` → NUNCA enviarle plantillas; el panel lo muestra como "no contactable".
- La constraint de exclusión de solapamiento es la última barrera; la lógica de disponibilidad la previene antes.

---

## 4. Integración WhatsApp Cloud API (conocimiento imprescindible)

Conceptos que condicionan TODO el diseño — el implementador debe entenderlos:

1. **Ventana de servicio de 24h**: si el cliente escribió en las últimas 24h, se puede responder con mensajes libres (texto, botones interactivos) sin costo. Fuera de la ventana, SOLO plantillas pre-aprobadas por Meta (categoría utility para recordatorios). Los recordatorios siempre se envían como plantilla (no se puede asumir ventana abierta).
2. **Mensajes interactivos** (dentro de ventana): `interactive.button` (máx. 3 botones) y `interactive.list` (máx. 10 ítems por sección). El flujo del bot se diseña alrededor de estos límites: listas para servicios/horarios, botones para confirmaciones.
3. **Webhook**: Meta envía POST a un endpoint público. Hay que: (a) responder el challenge GET de verificación con `hub.verify_token`; (b) validar firma `X-Hub-Signature-256` con el app secret; (c) responder 200 en <10s — encolar el procesamiento, nunca procesar inline; (d) ser idempotente: Meta reintenta y duplica mensajes (dedupe por `wa_message_id` contra `message_log`).
4. **Plantillas a crear y enviar a aprobación** (proceso manual en Meta Business, 24-48h):
   - `recordatorio_turno` (utility): "Hola {{1}}! Te recordamos tu turno en {{2}} el {{3}} a las {{4}} para {{5}}." + 3 botones quick-reply: Confirmo / Reprogramar / Cancelar.
   - `turno_creado` (utility): confirmación al agendar.
   - `turno_liberado` (utility, para V2/lista de espera — crearla ya, aprobada queda).
5. **Calidad del número**: Meta degrada números con muchos bloqueos. Mitigación obligatoria: respetar `opted_out`, comando "BAJA" que setea opt-out, frecuencia limitada (nunca más de N plantillas/día por cliente).
6. **Modo desarrollo**: la app de Meta en modo dev permite mensajes solo a números de prueba registrados — suficiente para las Fases 1-4. El alta de producción (verificación del negocio) es un trámite paralelo, no bloquea el desarrollo.

`packages/core-whatsapp` expone: `sendTemplate()`, `sendInteractive()`, `sendText()`, `parseWebhook()` (normaliza el payload de Meta a eventos tipados: `MessageReceived`, `ButtonPressed`, `StatusUpdate`).

---

## 5. La máquina de estados del bot (`core-bot`)

Diseño: FSM pura. Firma: `step(state, context, event) → { nextState, nextContext, effects[] }`. Los `effects` (enviar mensaje X, crear turno, etc.) los ejecuta la capa exterior. Cero I/O dentro de la FSM.

### Flujo A — Respuesta a recordatorio (botones de plantilla)
```
ButtonPressed('confirm')  → appointment.status = confirmed  → enviar "✅ Te esperamos!"
ButtonPressed('cancel')   → status = cancelled (by customer) → liberar slot → "Listo, cancelado. Escribinos cuando quieras reagendar."
ButtonPressed('resched')  → entra al Flujo B con servicio/staff precargados en context
```

### Flujo B — Reserva guiada
```
idle → (cualquier mensaje) → menú principal [Reservar turno | Mis turnos | Hablar con el salón]
awaiting_service:  lista de services activos (máx 10; si hay más, paginar con ítem "Ver más")
awaiting_staff:    lista de staff que hacen ese servicio + opción "Cualquiera"
awaiting_day:      lista con los próximos 7 días que tengan al menos un slot libre
awaiting_time:     lista de horarios libres del día (de core-scheduling)
confirming:        resumen + botones [Confirmar | Cambiar | Cancelar]
done:              crea appointment (status pending, source bot), envía plantilla turno_creado,
                   programa recordatorios
```

Reglas transversales:
- Estado con TTL 30 min (`conversations.expires_at`); expirado → cualquier mensaje reinicia en menú principal con texto amable.
- Input no reconocido → repetir el paso actual con "No te entendí, elegí una opción 👇" (máx. 2 veces; a la 3ra ofrecer "Hablar con el salón").
- "Hablar con el salón" → marca la conversación `handoff`, el bot se silencia para ese cliente por 4h, y el panel muestra alerta (la respuesta humana sale del WhatsApp normal del salón en el MVP — no construir inbox propio).
- Carrera de slots: si al confirmar el slot ya fue tomado, mensaje "Uy, ese horario se acaba de ocupar 😞" y volver a `awaiting_time` con disponibilidad fresca.
- "Mis turnos": lista turnos futuros del cliente con botones para cancelar/reprogramar cada uno.

### Disponibilidad (`core-scheduling`)
`getAvailableSlots(tenantId, serviceId, staffId|null, date) → Slot[]`:
intersección de (horario semanal del staff) − (schedule_blocks) − (appointments no cancelados), discretizada cada 15 min, filtrando slots donde no entra `duration_min` del servicio. staff null = unión de todos los que hacen el servicio. Función pura sobre datos cargados; cubrir con tests los bordes (turnos colindantes, bloques que parten el día, servicio más largo que el hueco).

---

## 6. Panel web (pantallas del MVP)

Mobile-first. Rutas bajo `/app` (autenticado, scoped al tenant del usuario):

1. **Agenda** (home): vista día (columnas por profesional) y semana. Color por estado del turno (pendiente amarillo / confirmado verde / cancelado gris). Crear turno manual con: cliente (buscar por teléfono o crear), servicio, profesional, fecha/hora (mostrando solo slots válidos). Acciones sobre turno: confirmar manual, cancelar, marcar no-show, marcar completado.
2. **Clientes**: lista con búsqueda, detalle con historial de turnos, indicador opt-out.
3. **Servicios**: ABM (nombre, duración, precio, activo, orden).
4. **Equipo**: ABM de profesionales + editor de horario semanal + bloqueos.
5. **Configuración**: datos del salón, horas de recordatorio (24h/2h on-off), estado de conexión WhatsApp.
6. **Métricas** (simple): turnos del mes por estado, tasa de confirmación vía bot, tasa de no-show, comparativa mes anterior.

Onboarding de un tenant nuevo (lo hace el admin del producto, no self-service en MVP): script/pantalla interna para crear tenant + conectar credenciales de WhatsApp.

---

## 7. Fases de implementación

Cada fase termina con sus criterios de aceptación verificados y deployada. **No empezar una fase sin cerrar la anterior.**

### Fase 0 — Esqueleto (2-3 días)
Monorepo con la estructura de §2, Postgres en Docker, schema Drizzle completo de §3 con migraciones y seed (1 tenant demo, 2 staff, 5 servicios, horarios), Auth.js funcionando, deploy a Vercel + Neon, CLAUDE.md del repo (stack, comandos, convenciones, link a esta spec).
✅ *Criterios*: `pnpm dev` levanta todo; login funciona; seed visible en una page placeholder; CI con typecheck + tests corre en cada push.

### Fase 1 — Agenda en panel (1,5-2 semanas)
Pantallas 1-4 de §6 completas. Turnos manuales con validación de disponibilidad (`core-scheduling` con tests).
✅ *Criterios*: crear/editar/cancelar turnos desde el celular; imposible solapar turnos del mismo staff (probado con test de la constraint y de la lógica); bloqueos funcionan; suite de `core-scheduling` cubre los bordes listados en §5.

### Fase 2 — Infraestructura WhatsApp (1 semana)
`core-whatsapp` completo. Webhook (verificación, firma, dedupe, encolado). Echo-bot de prueba detrás de un flag. Plantillas redactadas y ENVIADAS A APROBACIÓN (hacerlo al inicio de la fase: tarda 24-48h).
✅ *Criterios*: mensaje a número de prueba → aparece en `message_log` y el echo responde; webhook responde <1s; mensaje duplicado de Meta no genera doble procesamiento; firma inválida → 401.

### Fase 3 — Recordatorios (1 semana) ← **primer entregable vendible**
Jobs en Trigger.dev: al crear turno se programan recordatorios según config del tenant; al cancelar/reprogramar se cancelan/reprograman. Plantilla con botones + Flujo A completo. Estados de turno reflejados en la agenda en tiempo casi real (polling 30s alcanza).
✅ *Criterios*: e2e demostrable — crear turno para dentro de 25h → llega plantilla a las 24h → tocar "Confirmo" → turno verde en panel; tocar "Cancelar" → slot liberado y visible; reprogramar desde panel → recordatorio viejo cancelado en Trigger.dev, nuevo programado; cliente con opt-out no recibe nada.

### Fase 4 — Bot de reservas (2 semanas)
`core-bot` con Flujo B completo + tests unitarios de la FSM (toda transición y todo caso raro de §5). "Mis turnos", handoff, TTL, carrera de slots.
✅ *Criterios*: reserva completa de punta a punta desde WhatsApp en <60s sin tocar el panel; los tests de FSM cubren: input inválido ×3, expiración de sesión, slot robado en carrera, opt-out a mitad de flujo; turno creado por bot dispara sus recordatorios.

### Fase 5 — Métricas, pulido y producción (1 semana)
Pantalla de métricas, manejo de errores de envío (reintentos con backoff en Trigger.dev, alerta en panel si WhatsApp falla), rate-limit defensivo, revisión de seguridad (RLS activo, tokens cifrados, webhook firmado), alta de producción en Meta.
✅ *Criterios*: tenant real con número real funcionando; checklist de seguridad pasada; runbook de onboarding de un salón nuevo escrito (paso a paso del trámite Meta incluido).

**Total estimado: 7-9 semanas** de desarrollo efectivo.

---

## 8. Variables de entorno

```
DATABASE_URL=
AUTH_SECRET=                    # Auth.js
AUTH_GOOGLE_ID= / AUTH_GOOGLE_SECRET=
META_APP_SECRET=                # validación de firma del webhook
META_WEBHOOK_VERIFY_TOKEN=      # challenge de verificación
META_GRAPH_API_VERSION=v21.0    # o la vigente
TRIGGER_SECRET_KEY=
ENCRYPTION_KEY=                 # para cifrar wa_access_token por tenant (AES-256-GCM)
```
(Las credenciales de WhatsApp son POR TENANT y viven cifradas en DB, no en env.)

---

## 9. Puntos a verificar al momento de implementar (pueden haber cambiado)

- Tarifas vigentes de la Cloud API para Argentina (Meta pasó a precio por mensaje en 2025; utility dentro de ventana abierta quedó gratis — confirmar en la doc oficial).
- Versión vigente de la Graph API y del flujo Embedded Signup para conectar números de tenants.
- Límites actuales de mensajes interactivos (botones/listas).

## 10. Extensiones previstas (NO implementar ahora, pero no bloquear)

El diseño debe dejar la puerta abierta a: señas con Mercado Pago (plan 02 — por eso `appointments` podrá referenciar un `payments` futuro), recordatorios recurrentes no atados a turno (plan 03 — por eso `reminders` tiene `kind` extensible), lista de espera, campañas de marketing, multi-sucursal (un tenant por sucursal alcanza al principio). `tenants.vertical` y `tenants.settings` jsonb son el mecanismo de extensión por vertical.
