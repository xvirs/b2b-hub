# T3 — Superficie HTTP y Catálogo de Jobs (Trigger.dev v3)

> Definición técnica de TODO lo que entra y sale por HTTP (`apps/web`) y de TODOS los jobs programados (`core-jobs`). La lógica de negocio NO vive acá: vive en los paquetes de [T2-INTERFACES-PAQUETES.md](T2-INTERFACES-PAQUETES.md); este documento define el transporte, el encolado y la programación temporal.
>
> Orden de autoridad ante conflicto entre docs: **T1 (datos) > T2 (contratos) > T3 (este doc) > planes 0X (intención)**. Convenciones globales en `../00-INDICE.md`. Stack fijo: Next.js App Router + Trigger.dev v3.
>
> Resolución de referencias desde T2: «T3 §2.1/§2.2» = webhooks de este doc; «T3 §3» (reintentos del job que envía) = **§5.2**; «T3 §4» (catálogo de plantillas) = **§8**.

---

## 1. Mapa de la superficie HTTP

| Ruta | Método | Auth | Para qué |
|---|---|---|---|
| `/api/webhooks/whatsapp` | GET | `hub.verify_token` | Challenge de verificación de Meta |
| `/api/webhooks/whatsapp` | POST | Firma `X-Hub-Signature-256` | Eventos entrantes de WhatsApp |
| `/api/webhooks/mercadopago` | POST | Firma `x-signature` (secret por tenant) | Notificaciones de pago de MP |
| `/pay/[paymentId]` | GET | Pública | Redirect al `init_point` de Checkout Pro |
| `/api/health` | GET | Pública | Healthcheck (DB ping + versión) |
| Server Actions bajo `/app` | POST (Next.js) | Sesión Auth.js | Toda la API del panel (§4) |

No existe API REST interna: el panel usa exclusivamente Server Actions. No exponer endpoints adicionales sin actualizar este doc.

---

## 2. Webhooks externos

### 2.1 `GET/POST /api/webhooks/whatsapp`

**GET — challenge de verificación** (lo dispara Meta al configurar el webhook):
- Validar `hub.mode === 'subscribe'` y `hub.verify_token === META_WEBHOOK_VERIFY_TOKEN`.
- OK → responder `hub.challenge` como texto plano, 200. Mismatch → 403.

**POST — eventos entrantes.** Secuencia exacta del route handler:

1. **Leer el raw body ANTES de parsear** — gotcha de Next.js App Router: usar `const raw = await req.text()` y recién después `JSON.parse(raw)`. La firma se calcula sobre los bytes crudos; si se parsea primero, no hay forma de validarla.
2. Validar `X-Hub-Signature-256` con `verifyWebhookSignature(raw, header, META_APP_SECRET)` (T2 §2.1) — HMAC-SHA256, comparación timing-safe. Inválida o ausente → 401, sin loguear el body.
3. `parseWebhook(body)` → `InboundEvent[]` (un POST de Meta puede traer N eventos).
4. Por cada evento, según `kind`:
   - `message_received` / `button_pressed` / `list_item_selected` → **encolar `wa.process-inbound`** (§5.1) con el evento serializado. NUNCA procesar inline.
   - `status_update` → procesar inline: `UPDATE message_log SET status = $status WHERE wa_message_id = $waMessageId` (más `error` si `failed`). Es un UPDATE indexado de <5ms; no amerita job. Si `status = 'failed'`, crear alerta en panel.
5. Responder **200 en <1s** siempre que la firma sea válida, incluso si el encolado parcial falló (se loguea y alerta; Meta no debe reintentar lo ya recibido).

La idempotencia NO vive en el endpoint: vive en el job (dedupe por `wa_message_id` contra `message_log`, T2 §7 paso 2). El endpoint puede encolar duplicados tranquilamente.

### 2.2 `POST /api/webhooks/mercadopago`

La URL registrada como `notification_url` en cada preference incluye el tenant: `https://{APP_BASE_URL}/api/webhooks/mercadopago?tenant={tenantId}` — así el handler sabe qué `mp_webhook_secret` (cifrado en DB, por tenant) usar para validar. Secuencia:

1. Leer raw body + headers. Resolver tenant por query param; inexistente o `suspended` → 200 silencioso (no darle pistas a un atacante, no hacer reintentar a MP).
2. Validar `x-signature` de MP: reconstruir el manifest `id:{data.id};request-id:{x-request-id};ts:{ts};`, HMAC-SHA256 con el `mp_webhook_secret` descifrado del tenant, comparar contra el componente `v1` del header. Inválida → 401.
3. **Encolar `mp.process-event`** (§5.4) con `{ tenantId, headers, body }` y responder 200 en <5s. Nada de lógica inline.

El job llama `PaymentsPort.processWebhookEvent()` (T2 §5), que dedupea contra `payment_events`, re-consulta `GET /v1/payments/{id}` a la API de MP (jamás confiar en el body) y devuelve `duplicate | ignored | transition`. El **ruteo del `transition` por `external_reference`**:

- Matchea regex de UUID → es `appointments.id` → **flujo canchas** (plan 02 §6.3): `approved` → appointment `confirmed` + plantilla `reserva_confirmada` + recordatorio 3h; carrera de pago tardío según plan 02 §6.3.
- Contiene `:` → es `${membershipId}:${periodStart}` → **flujo gym** (plan 04 §6.6-6.7): INSERT `membership_payment` + `applyPayment()` en transacción; UNIQUE de período hace el reintento inocuo; pago duplicado (período ya pago en efectivo) → alerta "devolver por MP".
- Cualquier otra forma → loguear y marcar `ignored` (alerta en panel, no es un 5xx).

### 2.3 Códigos de respuesta (ambos webhooks)

| Código | WhatsApp | Mercado Pago |
|---|---|---|
| 200 | Firma OK: encolado (o status_update aplicado). También challenge GET correcto | Firma OK: encolado. También tenant desconocido/suspendido (silencioso) |
| 401 | Firma `X-Hub-Signature-256` inválida o ausente | Firma `x-signature` inválida o ausente |
| 403 | Challenge GET con `verify_token` incorrecto | — |
| 422 | Firma válida pero el body no matchea el envelope de Meta (se loguea crudo antes) | Firma válida pero body sin `data.id` / tópico desconocido |

**Política inquebrantable: NUNCA responder 5xx por errores de negocio o de procesamiento.** Meta y MP reintentan los no-2xx con backoff y amplifican el problema (tormenta de duplicados justo cuando el sistema está débil). Un error interno post-validación se loguea, alerta en panel y devuelve 200; la recuperación es asunto de los jobs, no del reintento del proveedor. Los 5xx quedan reservados a fallas reales de infraestructura (DB caída) donde el reintento del proveedor SÍ es la salvación.

---

## 3. Rutas públicas auxiliares

### 3.1 `GET /pay/[paymentId]`

Redirect 302 al `mp_init_point` del payment. **Existe porque los botones URL de las plantillas de Meta exigen dominio fijo con sufijo variable** (`https://{APP_BASE_URL}/pay/{{1}}`, plan 02 §9 y plan 04 §9) — no se puede meter el init_point de MP como variable. Comportamiento:

- Payment `pending` con preference vigente → loguear el click (`payment_events` con `type = 'pay_link_clicked'`) y 302 al `mp_init_point`.
- Payment `approved` → página estática "Este pago ya fue acreditado ✅".
- Payment `expired`/`rejected`/inexistente → página estática "Este link de pago ya no está vigente" (sin detalle del motivo: el `paymentId` es público).

Sin auth (llega desde WhatsApp del cliente final). El `paymentId` es UUID v7: no enumerable.

### 3.2 `GET /api/health`

`{ status: 'ok', version, db: 'ok' }` con un `SELECT 1`. DB caída → 503 (acá sí: es infraestructura). Para uptime monitoring externo.

---

## 4. API del panel — Server Actions

Convención general (aplica a TODAS las actions, sin excepción):

- Server Actions de Next.js, una por operación, naming `<modulo>.<accion>` (archivo `actions/<modulo>.ts`).
- **Zod en input Y output**: `parse()` del input al entrar, `parse()` del resultado al salir. Nada cruza el borde sin validar.
- **El `tenantId` sale SIEMPRE del session token de Auth.js, JAMÁS del payload del cliente.** Toda query usa `tenantDb(tenantId)` de T2 §1. Un input con `tenantId` es un bug de diseño.
- Mutaciones de dominio que disparan mensajería lo hacen vía efectos/jobs (§5), nunca llamando a `core-whatsapp` inline desde la action.
- Errores de negocio → retorno tipado `{ ok: false, code, message }`; nunca throw hacia el cliente.

### 4.1 Motor (todos los verticales)

| Action | Input (resumen) | Qué hace |
|---|---|---|
| `appointments.create` | customerId \| newCustomer, staffId/courtId, serviceId?, petId?, startsAt, durationMin? | Valida slot con `core-scheduling`, crea turno `pending`/`confirmed`, programa `reminders.dispatch` |
| `appointments.reschedule` | appointmentId, newStartsAt, staffId? | Revalida slot, cancela reminders viejos (vía `trigger_run_id`), programa nuevos |
| `appointments.cancel` | appointmentId, by ('salon'), reason? | Cancela, cancela reminders, dispara `waitlist.cascade` si aplica (cancha) |
| `appointments.confirm` | appointmentId | `pending` → `confirmed` manual |
| `appointments.markNoShow` | appointmentId | Marca `no_show` (cancha: payment → `forfeited`) |
| `appointments.markCompleted` | appointmentId | Marca `completed` (vet: el front abre el modal sanitario → `healthEvents.register`) |
| `customers.search` | query (nombre/teléfono), page | Lista con búsqueda |
| `customers.create` / `customers.update` | phone E.164, name, notes? | ABM; unique por (tenant, phone) |
| `customers.setOptOut` | customerId, optedOut | Marca no contactable (el bot también lo hace con "BAJA") |
| `services.create/update/archive` | name, durationMin, priceCents, active, sortOrder | ABM servicios |
| `staff.create/update/archive` | name, active, sortOrder, serviceIds[] | ABM profesionales + N:M con servicios |
| `staff.setWeeklySchedule` | staffId, blocks[{weekday, start, end}] | Reemplaza el horario semanal completo |
| `scheduleBlocks.create/delete` | staffId? \| courtId? \| null (todo el local), startsAt, endsAt, reason | Bloqueos puntuales |
| `settings.update` | patch del settings jsonb (Zod por vertical), reminderHours[] | Config del tenant; valida contra el schema del vertical |

### 4.2 Vertical canchas (plan 02)

| Action | Input (resumen) | Qué hace |
|---|---|---|
| `courts.create/update/archive` | name, sport, indoor, active, sortOrder | ABM canchas |
| `courtPricing.upsertGrid` | courtId, rows[{durationMin, band, priceCents}] | Grilla duración × franja |
| `courtPricing.copyFrom` | fromCourtId, toCourtId | Copia precios entre canchas |
| `appointments.createWithDeposit` | (= create) + depositMode ('cash_collected'\|'no_deposit'\|'send_link') | Reserva manual; `cash_collected` crea payment manual `approved` |
| `recurringBookings.create` | customerId, courtId, weekday, startTime, durationMin, priceCents, endsOn? | Alta de fijo; genera ocurrencias iniciales |
| `recurringBookings.end` | recurringBookingId, endsOn | Termina; cancela ocurrencias futuras (refund según política) |
| `waitlist.offerNow` | waitlistId | Fuerza la oferta manual (dispara cascada §5.6 desde ese registro) |
| `waitlist.cancel` | waitlistId | Baja de la cola |
| `payments.refund` | paymentRowId, reason | Refund manual discrecional vía `PaymentsPort.refund()` |
| `payments.list` | filters (status, rango fechas), page | Listado + totales del mes |

### 4.3 Vertical veterinarias (plan 03)

| Action | Input (resumen) | Qué hace |
|---|---|---|
| `pets.create/update` | customerId, name, species, breed?, birthDate?, notes? | ABM mascotas |
| `pets.deactivate` | petId | Baja (fallecida/inactiva): corta TODOS sus recordatorios al instante |
| `healthEvents.register` | petId, eventTypeId, appliedOn, source, appointmentId?, nextDueOnOverride? | Registra aplicación, calcula `next_due_on` (override manual permitido) |
| `healthEvents.sendReminderNow` | healthEventIds[] | Envío manual desde Vencimientos; respeta anti-spam de plan 03 §6 |
| `healthEventTypes.create/update/archive` | name, defaultPeriodDays, species, serviceId?, active | ABM del catálogo por tenant |

### 4.4 Vertical gimnasios (plan 04)

| Action | Input (resumen) | Qué hace |
|---|---|---|
| `plans.create/update/archive` | name, priceCents, weeklyClasses?, active | ABM planes |
| `memberships.create` | customerId, planId, billingDay? | Alta con proración (`prorate()` de T2 §6); UNIQUE parcial de membership viva |
| `memberships.pause` / `memberships.resume` | membershipId, reason? | Pausa (detiene escalera) / reanuda extendiendo `current_period_end` |
| `memberships.cancel` | membershipId | Baja definitiva |
| `memberships.changePlan` | membershipId, newPlanId | Cambio de plan (rige desde el próximo período) |
| `memberships.registerCashPayment` | membershipId, periodStart, amountCents | Cuota en efectivo: `membership_payment` method `cash` + `registered_by`; UNIQUE de período anti doble cobro |
| `classSessions.create/update/deactivate` | discipline, staffId, weekday, startTime, durationMin, capacity | ABM de clases recurrentes |
| `classOccurrences.markAttendance` | occurrenceId, bookingId, status ('attended'\|'no_show') | Asistencia por socio |
| `classOccurrences.cancelDay` | occurrenceId | Cancela la clase del día; lista teléfonos de anotados para aviso manual (MVP) |
| `classOccurrences.overrideCapacity` | occurrenceId, capacity | Ajuste puntual del cupo |
| `checkin.search` | query | Búsqueda de socio + semáforo (verde/rojo/amarillo) + occurrence en curso ±30 min |

---

## 5. Catálogo de jobs (Trigger.dev v3)

Convención: id `<dominio>.<accion>`, definidos en `packages/core-jobs`, payloads validados con Zod al entrar al job. Los jobs orquestan (cargan datos → llaman lógica pura de T2 → ejecutan efectos); no contienen reglas de negocio propias. Todo `trigger_run_id` de jobs delayed cancelables se persiste en su fila (reminders, holds, ofertas) para poder cancelarlos al cambiar el estado.

**Cron por tenant a su hora local — patrón único decidido:** cada familia diaria tiene UN cron maestro que corre **cada hora en punto (UTC)**. El maestro consulta los tenants `active` del vertical correspondiente, calcula la hora local de cada uno con su `timezone`, y a los que coinciden con la hora objetivo del job (fija o de settings) les dispara el job por-tenant como evento. Ventajas: cero schedules dinámicos por tenant que sincronizar, soporta `send_hour` configurable, y un tenant nuevo entra solo. Los jobs por-tenant son idempotentes, así que un doble disparo del maestro es inocuo.

### 5.1 `wa.process-inbound` — evento (desde webhook §2.1)

- **Payload**: `z.object({ event: inboundEventSchema })` (el `InboundEvent` serializado de T2 §2.1, fechas ISO).
- **Hace**: ejecuta el algoritmo del orquestador de T2 §7 paso a paso: resolver tenant, dedupe, upsert customer, opt-out, TTL de conversación, `step()` de la FSM, transacción de dominio, efectos de mensajería. Los efectos `send_*` se delegan a `wa.send-message`.
- **Idempotencia**: paso 2 de T2 §7 — INSERT en `message_log` con UNIQUE(`wa_message_id`); violación = duplicado de Meta o re-run del job, fin silencioso.
- **Reintentos**: 3 intentos, backoff exponencial desde 10s. El dedupe hace el re-run seguro incluso si falló a mitad de la transacción de dominio (la transacción es atómica).

### 5.2 `wa.send-message` — evento

- **Payload**: `z.object({ tenantId: z.string().uuid(), to: e164, message: outboundMessageSchema /* text|buttons|list|template, shapes de T2 §2.2 */, customerId: z.string().uuid().nullable() })`.
- **Hace**: descifra credenciales WA del tenant, verifica `opted_out` (corta plantillas a opt-out — última barrera), llama al `WaClient`, loguea resultado en `message_log` (éxito Y fallo, regla de T2 §2.2).
- **Idempotencia**: NO es idempotente por naturaleza (un re-run = mensaje duplicado al cliente). Mitigación: solo reintenta cuando `SendResult.ok === false` o error de red ANTES de obtener `waMessageId`; con `waMessageId` en mano, el run termina ok.
- **Reintentos**: 3 intentos, backoff exponencial (30s, 2min, 8min). **Al 3er fallo: alerta en panel** (T2 §7 paso 10) y `message_log.status = 'failed'`. Los errores de mensajería jamás revierten dominio.

### 5.3 `reminders.dispatch` — delayed a `reminders.scheduled_for`

- **Payload**: `z.object({ reminderId: z.string().uuid() })`.
- **Hace**: implementa la Fase 3 del plan 01. Recarga el reminder y su appointment; **verifica que siga vigente**: reminder `scheduled`, appointment `pending|confirmed`, customer sin opt-out. Envía la plantilla del vertical (`recordatorio_turno` / `recordatorio_partido`) vía `wa.send-message` y marca `sent`.
- **Idempotencia**: el UPDATE a `sent` es condicional (`WHERE status = 'scheduled'`); segundo run no encuentra nada y termina. La cancelación de turnos cancela el run vía `trigger_run_id`, y esta verificación es la red de seguridad si la cancelación del run falló.
- **Reintentos**: 2 intentos (un recordatorio tardío >30 min pierde sentido; el fallo queda en `reminders.status = 'failed'` + alerta).

### 5.4 `mp.process-event` — evento (desde webhook §2.2)

- **Payload**: `z.object({ tenantId: z.string().uuid(), headers: z.record(z.string()), body: z.unknown() })`.
- **Hace**: descifra credenciales MP del tenant, llama `PaymentsPort.processWebhookEvent()` (T2 §5). `duplicate`/`ignored` → fin. `transition` → rutea por `external_reference` según §2.2: flujo canchas (plan 02 §6.3, incl. carrera de pago tardío con refund automático) o flujo gym (plan 04 §6.7). Mensajes de confirmación salen por `wa.send-message`.
- **Idempotencia**: doble capa — `payment_events` UNIQUE(provider, provider_event_id) dentro del Port + transiciones monótonas de `payments.status`; en gym, tercera capa: UNIQUE(membership_id, period_start).
- **Reintentos**: 5 intentos, backoff exponencial desde 30s (acá el reintento es barato y la consulta a la API de MP puede fallar transitoriamente).

### 5.5 `payments.expire-hold` — delayed a `appointments.hold_expires_at`

- **Payload**: `z.object({ appointmentId: z.string().uuid() })`.
- **Hace**: plan 02 §6.2 — si el appointment sigue `awaiting_payment` y el payment no está `approved`: appointment → `cancelled` (cancelled_by `system`), payment → `expired` (`PaymentsPort.expirePending`), preference invalidada vía `expiration_date_to`, mensaje "se venció el tiempo de pago" al cliente, y **dispara `waitlist.cascade`** (regla de integridad del plan 02 §3).
- **Idempotencia**: guard `WHERE status = 'awaiting_payment'`; si el pago entró antes del run, no hace nada. Si el run y un `approved` corren a la vez, gana la transacción que commitee primero; la carrera pago-tardío de §5.4 resuelve el otro lado.
- **Reintentos**: 3 intentos, backoff 1min.

### 5.6 `waitlist.cascade` — evento (canchas)

- **Payload**: `z.object({ tenantId: z.string().uuid(), freedSlot: z.object({ courtId: z.string().uuid(), sport: z.string(), date: z.string(), startTime: z.string(), durationMin: z.number(), priceCents: z.number() }) })`.
- **Hace**: plan 02 §6.5 — solo si el slot cae en franja pico futura con >2h. Toma los `waiting` que matcheen (sport, date, franja) en orden `created_at ASC`; al primero: plantilla `slot_liberado`, status `offered`, `offer_expires_at = now() + 15min`, programa `waitlist.expire-offer`. Cola vacía o slot ya retomado → fin.
- **Idempotencia**: guard "máximo un `offered` activo por slot" — si ya hay oferta viva para ese slot, el run termina sin ofrecer.
- **Reintentos**: 3 intentos, backoff 30s.

### 5.7 `waitlist.expire-offer` — delayed a `offer_expires_at` (canchas)

- **Payload**: `z.object({ waitlistId: z.string().uuid() })`.
- **Hace**: si el registro sigue `offered` → `expired_offer` y re-dispara `waitlist.cascade` con el mismo slot (sigue con el próximo de la cola). Si ya está `accepted` → fin.
- **Idempotencia**: UPDATE condicional `WHERE status = 'offered'`. **Reintentos**: 2.

### 5.8 `recurring.generate` — cron maestro horario → evento por tenant (canchas)

- **Payload por tenant**: `z.object({ tenantId: z.string().uuid() })`. Hora objetivo: **03:00 local** (plan 02 §6.6), fija.
- **Hace**: por cada `recurring_bookings` activo: genera appointments `confirmed` (source `recurring`, sin seña salvo `recurringRequiresDeposit`) hasta `today + rollingWeeks`, avanza `generated_until`. Conflicto → NO genera esa ocurrencia + alerta en panel. `ends_on` a ≤14 días → plantilla `fijo_por_vencer` (una sola vez).
- **Idempotencia**: `generated_until` es el cursor — un re-run no regenera lo ya cubierto; la EXCLUDE constraint de solapamiento es la última barrera. **Reintentos**: 3, backoff 5min.

### 5.9 `health.daily-scan` — cron maestro horario → evento por tenant (vet)

- **Payload por tenant**: `z.object({ tenantId: z.string().uuid() })`. Hora objetivo: `settings.vet.health_reminder_send_hour` local (default 10).
- **Hace**: el algoritmo completo del plan 03 §6: (1) evento vigente por (pet, type) de mascotas activas; (2) filtro `next_due_on ∈ [hoy−30, hoy+lead_days]`, sin `reminder_sent_at`, sin `dismissed`, sin opt-out; (3) **agrupación por customer** — 1 ítem → `recordatorio_sanitario`, ≥2 → `recordatorio_sanitario_multi` (máx `max_items_per_message`), UN solo mensaje por customer; (4) anti-spam `min_gap_days` y tope de 3 re-envíos por evento; (5) marcar `reminder_sent_at` en los incluidos, en la misma transacción que encola el envío.
- **Idempotencia**: el marcado de `reminder_sent_at` + el anti-spam por customer hacen el doble run del día inocuo (el segundo no encuentra candidatos). **Reintentos**: 3, backoff 5min.

### 5.10 `memberships.daily-cycle` — cron maestro horario → evento por tenant (gym)

- **Payload por tenant**: `z.object({ tenantId: z.string().uuid() })`. Hora objetivo: **08:00 local** (plan 04 §6.1), fija.
- **Hace**: por cada membresía no pausada/cancelada: `computeStatus()` y persiste (vencida → `overdue`); luego `dunningStep()` (T2 §6) decide el escalón (`pre_due`/`due`/`overdue`, máx uno por corrida, dedupe por `dunning_notices` kind+período). Escalón a enviar → genera preference MP fresca (`external_reference = membershipId:periodStart`), la guarda en `dunning_notices.mp_preference_id` y envía la plantilla con botón URL `/pay/{paymentId}`. Opt-out → sin aviso, queda en Cobranza.
- **Idempotencia**: dedupe por `dunning_notices` (kind+período) — el doble run no duplica avisos; `computeStatus` es puro y re-aplicable. **Reintentos**: 3, backoff 5min.

### 5.11 `classes.materialize` — cron maestro horario → evento por tenant (gym)

- **Payload por tenant**: `z.object({ tenantId: z.string().uuid() })`. Hora objetivo: **02:00 local** (decidido acá; antes que cualquier reserva del día).
- **Hace**: por cada `class_session` activa, crea las `class_occurrences` que falten hasta `today + 7 días` con su `starts_at` desnormalizado.
- **Idempotencia**: UNIQUE(class_session_id, date) — INSERT ... ON CONFLICT DO NOTHING. **Reintentos**: 3, backoff 5min.

### 5.12 `classes.no-show-sweep` — delayed a `occurrence.starts_at + duration + 2h` (gym)

- **Payload**: `z.object({ occurrenceId: z.string().uuid() })`. Lo programa `classes.materialize` al crear cada occurrence.
- **Hace**: plan 04 §6.8 — bookings aún `booked` de la occurrence → `no_show`. Sin penalidad automática (solo métricas).
- **Idempotencia**: UPDATE condicional `WHERE status = 'booked'`. Occurrence cancelada → fin. **Reintentos**: 2.

### 5.13 `classes.expire-offer` — delayed a `offer_expires_at` (gym)

- **Payload**: `z.object({ waitlistId: z.string().uuid() })`.
- **Hace**: plan 04 §6.5 — si sigue `offered` → `expired`, ofrecer al siguiente `waiting` (deadline `offerMinutes`, cap a la hora de la clase) o liberar el cupo si la cola se vació. Solo un `offered` por occurrence a la vez.
- **Idempotencia**: UPDATE condicional `WHERE status = 'offered'`. **Reintentos**: 2.

---

## 6. Variables de entorno y secretos

| Variable | Quién la usa | Notas |
|---|---|---|
| `DATABASE_URL` | core-db | Neon en prod, Docker local |
| `AUTH_SECRET` | Auth.js | |
| `AUTH_GOOGLE_ID` / `AUTH_GOOGLE_SECRET` | Auth.js | Login Google del panel |
| `META_APP_SECRET` | Webhook §2.1 | Firma `X-Hub-Signature-256` (la app de Meta es una sola, global) |
| `META_WEBHOOK_VERIFY_TOKEN` | Webhook §2.1 | Challenge GET |
| `META_GRAPH_API_VERSION` | core-whatsapp | ej. `v21.0`, verificar vigente |
| `TRIGGER_SECRET_KEY` | core-jobs | Trigger.dev cloud |
| `ENCRYPTION_KEY` | core-db | AES-256-GCM para secretos por tenant |
| `APP_BASE_URL` | `/pay`, back_urls, notification_url | Dominio fijo público, ej. `https://app.dominio.com` |

**Secretos POR TENANT (cifrados en DB con `ENCRYPTION_KEY`, jamás en env):** `wa_access_token` (+ `wa_phone_number_id`, `wa_business_account_id` en claro) y, para tenants con pagos, `mp_access_token`, `mp_webhook_secret`, `mp_user_id`. Los jobs los descifran con `decryptSecret()` (T2 §1) en cada run; nunca se cachean en memoria entre runs.

---

## 7. Decisiones tomadas (donde los planes eran ambiguos)

1. **Cron por tenant a su hora local**: un cron maestro POR FAMILIA de job que corre cada hora UTC y fan-out a los tenants cuya hora local coincide (§5, preámbulo). Sin schedules dinámicos por tenant.
2. **`status_update` de Meta se procesa inline en el webhook** (un UPDATE indexado), no se encola — los demás `InboundEvent` siempre se encolan.
3. **Resolución del tenant en el webhook de MP**: query param `?tenant={tenantId}` en la `notification_url` de cada preference (el secret de firma es por tenant y hay que elegirlo antes de validar).
4. **Disambiguación de `external_reference`**: regex de UUID → canchas; contiene `:` → gym (`membershipId:periodStart`). Formato no reconocido → `ignored` + alerta, nunca 5xx.
5. **Política 5xx**: reservados a fallas de infraestructura (DB caída); todo error de negocio/procesamiento responde 2xx/4xx y se recupera por jobs (§2.3).
6. **`/pay/[paymentId]` siempre activo** (no solo si Meta lo exige): unifica los botones URL de canchas y gym, y el log del click es una métrica gratis.
7. **Reintentos de `wa.send-message`**: 3 intentos (30s/2min/8min), alerta en panel al 3er fallo; nunca reintentar después de obtener `waMessageId` (evita duplicar mensajes al cliente).
8. **`classes.materialize` a las 02:00 local** y `classes.no-show-sweep` programado por el propio materialize al crear cada occurrence.
9. **`reminders.dispatch` re-verifica vigencia** (appointment, reminder, opt-out) aunque la cancelación de runs vía `trigger_run_id` exista — doble red.
10. **Server Actions como única API del panel** (sin REST interna), Zod en ambos bordes, tenant solo desde sesión. Las lecturas de pantallas (agenda, listados, métricas) van por Server Components con `tenantDb()`; las actions son solo mutaciones y búsquedas interactivas.

---

## 8. Catálogo consolidado de plantillas de WhatsApp

Nombres canónicos usados por `TemplateSend.name` (T2 §2.2). Los textos completos viven en los planes; acá el inventario único para no duplicar nombres entre verticales. Todas en `es_AR`, categoría **utility** (si Meta rechaza alguna como utility, ver plan 03 §9).

| Nombre | Plan | Botones | La envía |
|---|---|---|---|
| `recordatorio_turno` | 01 §4 | quick-reply: Confirmo / Reprogramar / Cancelar | `reminders.dispatch` (salon/vet) |
| `turno_creado` | 01 §4 | — | orquestador al crear turno por bot |
| `turno_liberado` | 01 §4 | (reservada para lista de espera V2 salones) | — (aprobarla ya) |
| `sena_pendiente` | 02 §4 | — (link `/pay/{id}` en el body) | orquestador al crear hold |
| `reserva_confirmada` | 02 §4 | — | `mp.process-event` (approved canchas) |
| `recordatorio_partido` | 02 §4 | quick-reply: Cancelar reserva | `reminders.dispatch` (cancha, 3h) |
| `slot_liberado` | 02 §4 | quick-reply: Sí, lo quiero / No, gracias | `waitlist.cascade` |
| `fijo_por_vencer` | 02 §4 | — | `recurring.generate` |
| `recordatorio_sanitario` | 03 §4 | quick-reply: Agendar turno / Ya lo hice en otro lado / No, gracias | `health.daily-scan` (1 vencimiento) |
| `recordatorio_sanitario_multi` | 03 §4 | ídem | `health.daily-scan` (≥2 vencimientos) |
| `cuota_por_vencer` | 04 §4 | URL `/pay/{{1}}` + quick-reply: Ya pagué en el gym | `memberships.daily-cycle` (pre_due) |
| `cuota_vencida` | 04 §4 | ídem | `memberships.daily-cycle` (due) |
| `cuota_recuperacion` | 04 §4 | URL `/pay/{{1}}` + quick-reply: Hablar con el gym | `memberships.daily-cycle` (overdue) |
| `clase_liberada` | 04 §4 | quick-reply: Lo quiero / No, gracias | oferta de lista de espera de clases (fuera de ventana) |

Regla operativa: las plantillas de cada vertical se redactan y envían a aprobación el **día 1 de su primera fase** (tardan 24-48h); el nombre acá es el contrato — cambiarlo requiere actualizar este doc y T2.
