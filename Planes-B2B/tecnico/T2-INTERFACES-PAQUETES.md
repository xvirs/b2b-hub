# T2 — Interfaces de los Paquetes del Motor (contratos TypeScript)

> Definición técnica de las APIs públicas de cada paquete `core-*`. Estos contratos son la columna vertebral del portfolio: los cuatro planes (01-04) los comparten. **Una IA que implemente cualquier plan debe respetar estas firmas**; si una firma resulta insuficiente al implementar, se actualiza ESTE documento en el mismo PR que el código.
>
> Complementa: [T1-DB-SCHEMA.md](T1-DB-SCHEMA.md) (tablas) y [T3-API-JOBS.md](T3-API-JOBS.md) (HTTP y jobs). Convenciones globales en `../00-INDICE.md`.

Principio rector (del plan 01 §2): **`core-bot`, `core-scheduling` y `core-memberships` son lógica pura** — reciben datos, devuelven decisiones y efectos; jamás hacen I/O. `core-whatsapp` y `core-payments` son adaptadores de I/O sin lógica de negocio. La orquestación (cargar datos → llamar la lógica → ejecutar efectos) vive en `apps/web` y `core-jobs`.

---

## 1. `core-db`

Exporta el schema Drizzle (ver T1), los tipos inferidos y dos utilidades transversales:

```ts
// Toda query de negocio pasa por un scope de tenant. PROHIBIDO consultar
// tablas de negocio sin él (RLS es la segunda barrera, no la primera).
export function tenantDb(tenantId: string): TenantScopedDb;

// Wrapper de transacción con el scope incluido
export function tenantTx<T>(tenantId: string, fn: (tx: TenantScopedDb) => Promise<T>): Promise<T>;

// Cifrado de credenciales por tenant (WhatsApp y Mercado Pago) — AES-256-GCM con ENCRYPTION_KEY
export function encryptSecret(plain: string): string;
export function decryptSecret(cipher: string): string;
```

---

## 2. `core-whatsapp` (adaptador Cloud API)

### 2.1 Eventos entrantes normalizados

`parseWebhook` convierte el payload crudo de Meta en eventos tipados. **Nada fuera de este paquete lee el JSON de Meta.**

```ts
export type InboundEvent =
  | { kind: 'message_received'; waMessageId: string; from: string /* E.164 */;
      tenantPhoneNumberId: string; timestamp: Date;
      content: { type: 'text'; text: string }
               | { type: 'media'; mediaType: 'image'|'document'|'audio'|'video'; mediaId: string; caption?: string }
               | { type: 'unsupported' } }
  | { kind: 'button_pressed'; waMessageId: string; from: string; tenantPhoneNumberId: string;
      timestamp: Date; buttonId: string; buttonTitle: string;
      contextWaMessageId?: string /* mensaje al que responde — clave para mapear plantilla→appointment */ }
  | { kind: 'list_item_selected'; waMessageId: string; from: string; tenantPhoneNumberId: string;
      timestamp: Date; itemId: string; itemTitle: string }
  | { kind: 'status_update'; waMessageId: string; tenantPhoneNumberId: string;
      status: 'sent'|'delivered'|'read'|'failed'; error?: { code: number; title: string } };

export function verifyWebhookSignature(rawBody: string, signatureHeader: string, appSecret: string): boolean;
export function parseWebhook(body: unknown): InboundEvent[];   // un POST de Meta puede traer N eventos
```

Convención de `buttonId`/`itemId`: `"<accion>:<uuid>"` — ej. `confirm:1f2a…`, `cancel:1f2a…`, `resched:1f2a…`, `book_health:…`, `done_external:…`, `dismiss_health:…`, `wl_yes:…`, `wl_no:…`, `pay_cash_claim:…`. El parser NO interpreta el id (eso es de `core-bot`); solo lo transporta.

### 2.2 Cliente de envío

```ts
export interface WaCredentials { phoneNumberId: string; accessToken: string }  // ya descifradas

export interface WaClient {
  sendText(to: string, text: string): Promise<SendResult>;
  sendButtons(to: string, body: string, buttons: Array<{ id: string; title: string /* ≤20 chars */ }>): Promise<SendResult>;   // máx 3
  sendList(to: string, body: string, buttonLabel: string,
           sections: Array<{ title?: string; items: Array<{ id: string; title: string /* ≤24 */; description?: string /* ≤72 */ }> }>
          ): Promise<SendResult>;   // máx 10 ítems total
  sendTemplate(to: string, template: TemplateSend): Promise<SendResult>;
}

export interface TemplateSend {
  name: string;                       // ej. 'recordatorio_turno' — catálogo en T3 §4
  language: 'es_AR';
  bodyParams: string[];               // {{1}}..{{n}} en orden
  buttons?: Array<                    // solo si la plantilla los define
    | { type: 'quick_reply'; payload: string }       // = buttonId que volverá en button_pressed
    | { type: 'url_suffix'; suffix: string }>;       // para botón URL dinámica https://dominio/pay/{{1}}
}

export type SendResult = { ok: true; waMessageId: string } | { ok: false; errorCode: number; errorTitle: string };

export function createWaClient(creds: WaCredentials, fetchImpl?: typeof fetch): WaClient;
```

Reglas del adaptador: nunca lanza por errores de API (devuelve `ok:false`); el caller loguea SIEMPRE en `message_log` (éxito y fallo); reintentos NO viven acá (viven en el job que envía — T3 §3).

---

## 3. `core-bot` (FSM pura)

### 3.1 Contrato central

```ts
export interface BotInput {
  vertical: 'salon' | 'cancha' | 'vet' | 'gym';
  state: string;                      // 'idle' | 'awaiting_service' | … (catálogo §3.3)
  context: BotContext;                // jsonb de conversations.context
  event: BotEvent;
  data: BotData;                      // TODO lo que la FSM necesita leer, ya cargado (ver 3.2)
  now: Date;                          // inyectado — la FSM jamás llama Date.now()
}

export interface BotStepResult {
  nextState: string;
  nextContext: BotContext;
  effects: Effect[];                  // se ejecutan EN ORDEN
}

export function step(input: BotInput): BotStepResult;   // pura, total: jamás lanza por input raro

export type BotEvent =
  | { type: 'text'; text: string }
  | { type: 'button'; action: string; targetId: string }      // buttonId ya partido en accion/uuid
  | { type: 'list'; action: string; targetId: string }
  | { type: 'session_expired' }                               // TTL vencido (lo detecta el orquestador)
  | { type: 'payment_approved'; appointmentId: string }        // eventos de sistema que también pasan por la FSM
  | { type: 'payment_expired'; appointmentId: string };
```

### 3.2 `BotData` — el orquestador carga, la FSM decide

```ts
export interface BotData {
  tenant: { id: string; name: string; vertical: string; settings: unknown; timezone: string };
  customer: { id: string; phone: string; name: string | null; optedOut: boolean };
  // Secciones opcionales según vertical/estado; el orquestador sabe qué cargar
  // mirando state (tabla "needs" exportada por el paquete):
  services?: ServiceLite[];  staff?: StaffLite[];  slots?: Slot[];        // salon/vet
  pets?: PetLite[];  healthDue?: HealthDueLite[];                          // vet
  courts?: CourtLite[];  pricing?: CourtPricingLite[];                     // cancha
  disciplines?: string[];  occurrences?: OccurrenceLite[];                 // gym
  membership?: MembershipLite | null;                                      // gym
  upcomingAppointments?: AppointmentLite[];
  pendingPayment?: { appointmentId: string; initPoint: string; expiresAt: Date } | null;
}
export const dataNeeds: Record<string /* vertical:state */, Array<keyof BotData>>;
```

### 3.3 Catálogo de estados (cerrado — agregar = editar este doc)

```
comunes:   idle · main_menu · my_bookings · handoff
salon/vet: awaiting_service · awaiting_staff · awaiting_day · awaiting_time · confirming
vet:       awaiting_pet · awaiting_pet_name · awaiting_pet_species · health_pick_event · health_external_when
cancha:    awaiting_sport · awaiting_duration · awaiting_day · awaiting_slot · confirming · awaiting_payment
           wl_sport · wl_day · wl_band
gym:       awaiting_discipline · awaiting_day · awaiting_class_time · confirming_class · my_quota
```

### 3.4 Efectos (unión cerrada y discriminada)

```ts
export type Effect =
  // Mensajería (los ejecuta el orquestador vía core-whatsapp + message_log)
  | { type: 'send_text'; text: string }
  | { type: 'send_buttons'; body: string; buttons: Array<{ id: string; title: string }> }
  | { type: 'send_list'; body: string; buttonLabel: string; sections: ListSection[] }
  | { type: 'send_template'; template: TemplateSend }
  // Dominio común
  | { type: 'create_appointment'; draft: AppointmentDraft }     // → dispara schedule_reminders en el orquestador
  | { type: 'cancel_appointment'; appointmentId: string; by: 'customer' }
  | { type: 'confirm_appointment'; appointmentId: string }
  | { type: 'set_handoff'; hours: number }                       // silencia el bot + alerta panel
  | { type: 'set_opt_out' }
  // Vertical cancha
  | { type: 'create_payment_hold'; draft: AppointmentDraft; depositCents: number; windowMin: number }
  | { type: 'join_waitlist'; sport: string; date: string; timeFrom: string; timeTo: string }
  | { type: 'accept_waitlist_offer'; waitlistId: string }
  | { type: 'decline_waitlist_offer'; waitlistId: string }
  // Vertical vet
  | { type: 'create_pet'; name: string; species: 'dog'|'cat'|'other' }
  | { type: 'register_external_health_event'; healthEventTypeId: string; petId: string; approxDaysAgo: number }
  | { type: 'dismiss_health_reminders'; healthEventIds: string[] }
  // Vertical gym
  | { type: 'create_class_booking'; occurrenceId: string }
  | { type: 'cancel_class_booking'; bookingId: string }
  | { type: 'join_class_waitlist'; occurrenceId: string }
  | { type: 'accept_class_offer'; waitlistId: string }
  | { type: 'create_quota_payment_link'; membershipId: string };  // el orquestador genera preference y responde con el link
```

Reglas de la FSM (testeadas en el paquete, sin mocks de I/O): input no reconocido → repetir paso (contador en context, al 3ro ofrecer handoff); TTL lo maneja el orquestador emitiendo `session_expired`; `optedOut` corta cualquier efecto de template saliente (el orquestador filtra, la FSM ni se entera); textos en español rioplatense definidos DENTRO del paquete (módulo `texts.ts` por vertical) para que los tests los cubran.

---

## 4. `core-scheduling` (disponibilidad pura)

Generalizado a "recurso" para servir a salones (staff) y canchas (courts):

```ts
export interface Slot { startsAt: Date; endsAt: Date; resourceId: string; priceCents?: number /* solo cancha */ }

export interface SchedulingData {            // cargado por el orquestador para la fecha pedida
  resources: Array<{ id: string; kind: 'staff'|'court' }>;
  weeklyHours: Array<{ resourceId: string; weekday: number; start: string; end: string }>;  // staff_schedules u horario del complejo
  blocks: Array<{ resourceId: string | null; startsAt: Date; endsAt: Date }>;
  busy: Array<{ resourceId: string; startsAt: Date; endsAt: Date }>;     // appointments no cancelados (awaiting_payment INCLUIDO)
}

export function getAvailableSlots(args: {
  data: SchedulingData;
  date: string;                       // YYYY-MM-DD en zona del tenant
  timezone: string;
  durationMin: number;
  granularityMin: 15 | 30;            // 15 salones/vet, 30 canchas
  priceResolver?: (resourceId: string, startsAt: Date) => number;   // cancha: court_pricing + franja pico
}): Slot[];

export function findNextAvailableDays(args: { /* mismos datos */ horizonDays: number }): string[]; // para awaiting_day
```

Invariantes (tests obligatorios, plan 01 §5): turnos colindantes no se pisan; bloque que parte el día genera dos ventanas; servicio más largo que el hueco no aparece; franja que cruza medianoche (cancha, `closeTime: '00:00'`); DST argentino no aplica (no hay) pero la conversión zona↔UTC se testea igual.

---

## 5. `core-payments` (adaptador Mercado Pago — agnóstico del vertical)

```ts
export interface MpCredentials { accessToken: string; webhookSecret: string }   // por tenant, descifradas

export interface CreatePreferenceArgs {
  tenantId: string;
  amountCents: number;
  title: string;                       // "Seña — Cancha 2 · jue 21:00" / "Cuota Junio — Plan Libre"
  externalReference: string;           // appointments.id  |  `${membershipId}:${periodStart}`
  expiresAt?: Date;                    // hold de seña; las cuotas no expiran
  backUrls: { success: string; failure: string; pending: string };
}

export interface PaymentsPort {
  createPreference(args: CreatePreferenceArgs): Promise<{ paymentRowId: string; preferenceId: string; initPoint: string }>;
  // Webhook (T3 §2.2): valida firma, dedupea contra payment_events, consulta GET /v1/payments/{id}
  // y devuelve la transición; NO toca appointments ni memberships — eso es del vertical.
  processWebhookEvent(raw: { headers: Record<string,string>; body: unknown }, creds: MpCredentials):
    Promise<{ outcome: 'duplicate' | 'ignored' }
           | { outcome: 'transition'; paymentRowId: string; externalReference: string;
               newStatus: 'approved'|'rejected'; mpPaymentId: string }>;
  refund(paymentRowId: string): Promise<{ ok: boolean; error?: string }>;
  markForfeited(paymentRowId: string): Promise<void>;
  expirePending(paymentRowId: string): Promise<void>;
}
```

Garantías del paquete: transiciones de `payments.status` **monótonas** (`approved` nunca retrocede; segundo `approved` = `duplicate`); todo evento crudo a `payment_events` ANTES de procesar; jamás confía en el body del webhook (siempre re-consulta la API de MP). Los consumidores (plan 02 §6.3-6.4, plan 04 §6.6-6.7) reaccionan a `transition` dentro de SU transacción.

---

## 6. `core-memberships` (lógica pura de cuotas — plan 04)

```ts
export function prorate(priceCents: number, from: LocalDate, billingDay: number): { amountCents: number; periodStart: LocalDate; periodEnd: LocalDate };
// round(price × días/30), piso 25% — bordes testeados: alta el día B, alta el 29-31, febrero

export function computeStatus(m: { currentPeriodEnd: LocalDate; pausedAt: LocalDate | null; cancelledAt: Date | null }, today: LocalDate):
  'active' | 'overdue' | 'paused' | 'cancelled';

export function dunningStep(args: {
  membership: { billingDay: number; currentPeriodEnd: LocalDate; status: string };
  noticesSentThisPeriod: Array<'pre_due'|'due'|'overdue'>;
  settings: { preDueDays: number; dueEnabled: boolean; overdueDays: number; enabled: boolean };
  today: LocalDate;
}): { send: 'pre_due'|'due'|'overdue' } | { send: null };       // máx UN escalón por corrida, dedupe por período

export function applyPayment(m: Membership, period: { start: LocalDate; end: LocalDate }, today: LocalDate):
  { newPeriodEnd: LocalDate };  // +1 mes desde max(hoy, vencimiento): pagar tarde no regala días

export function resumeFromPause(m: Membership, today: LocalDate): { newPeriodEnd: LocalDate }; // extiende por días pausados
```

`LocalDate` = string `YYYY-MM-DD` en zona del tenant (las cuotas son fechas civiles, no instantes — evita los bugs de medianoche).

---

## 7. Orquestador de mensajes entrantes (en `apps/web`, no es paquete pero su algoritmo es contrato)

Secuencia exacta al procesar un `InboundEvent` de tipo mensaje (encolado desde el webhook — T3 §2.1):

```
1. Resolver tenant por tenantPhoneNumberId; descartar si no existe o suspended.
2. Dedupe: INSERT message_log con UNIQUE(wa_message_id) — violación = duplicado de Meta, fin.
3. Upsert customer por (tenant_id, phone); actualizar last_interaction_at.
4. Si texto ∈ {BAJA, STOP, CANCELAR SUSCRIPCION} (case-insensitive) → opt-out + confirmación + fin.
5. Cargar conversation; si expires_at < now → evento session_expired primero, luego el real.
6. Si conversation.state = handoff y no venció → fin (bot silenciado).
7. data = cargar según dataNeeds[vertical:state] · result = step(input)
8. Transacción: persistir conversation (nextState/nextContext, TTL +30min) + efectos de dominio.
9. Ejecutar efectos de mensajería en orden; loguear cada uno en message_log.
10. Errores en efectos de mensajería NO revierten el dominio (el turno creado vale aunque el
    mensaje falle); se reintenta vía job (T3 §3) y se alerta en panel al 3er fallo.
```

---

## 8. Reglas de evolución de estos contratos

- Cambios **aditivos** (nuevo efecto, campo opcional, estado nuevo de un vertical nuevo): libres, actualizando este doc.
- Cambios **breaking** (renombrar efecto, cambiar firma): requieren actualizar este doc + los planes que lo referencian + los tests de todos los verticales construidos. Si una IA lo necesita, debe proponerlo explícitamente, no hacerlo en silencio.
- Este doc NO duplica el schema de DB (T1) ni los endpoints (T3); ante conflicto entre docs, el orden de autoridad es: T1 (datos) > T2 (contratos) > T3 (transporte) > planes 0X (intención).
