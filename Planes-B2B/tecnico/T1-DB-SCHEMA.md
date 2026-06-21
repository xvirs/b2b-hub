# T1 — Schema de Base de Datos (Drizzle ORM / PostgreSQL)

> Definición técnica COMPLETA del schema del portfolio (planes 01-04) en código Drizzle, tal como vive en `packages/core-db/src/schema/`. **Este documento es la autoridad sobre el modelo de datos**: si difiere de lo descrito en los planes 0X, manda este documento (orden de autoridad de T2 §8: T1 > T2 > T3 > planes).
>
> Convenciones aplicadas (de `00-INDICE.md`):
> - **PK UUID v7 generado en la aplicación** (`$defaultFn(() => uuidv7())`, paquete `uuidv7` npm). NO default de DB: Postgres no trae `uuidv7()` nativo en las versiones soportadas por Neon hoy, y generar en app permite conocer el id antes del INSERT (necesario para `external_reference` de MP y para los `buttonId` del bot).
> - `created_at`/`updated_at` **timestamptz** en todas las tablas; dinero en **centavos ARS (integer)**; teléfonos **E.164 varchar(20)**.
> - **Sin `deleted_at`**: ningún plan pide soft-delete por timestamp; las bajas lógicas usan los flags `active` ya especificados (staff, services, pets, courts, plans, class_sessions). Decisión documentada en §6.
> - El schema muestra el **estado FINAL** con los 4 verticales incorporados. En la práctica se llega con una migración evolutiva por vertical (motor → canchas → vet → gym), pero las columnas de verticales en `appointments` son nullable, así que tenerlas desde el día 1 no rompe nada y evita renumerar migraciones: **se recomienda crear todo el schema motor+columnas nullable en la migración 0001** y las tablas de cada vertical en su propia migración al construirlo.

---

## 1. `enums.ts` — todos los pgEnum

```ts
import { pgEnum } from 'drizzle-orm/pg-core';

// ── Motor ──────────────────────────────────────────────────────────────
export const verticalEnum = pgEnum('vertical', ['salon', 'cancha', 'vet', 'gym']);
export const tenantStatusEnum = pgEnum('tenant_status', ['active', 'suspended']);
export const userRoleEnum = pgEnum('user_role', ['owner', 'staff']);

export const appointmentStatusEnum = pgEnum('appointment_status', [
  'pending', 'awaiting_payment', 'confirmed', 'cancelled', 'no_show', 'completed',
]); // 'awaiting_payment' del plan 02 — enum ÚNICO compartido por todos los verticales (§6)
export const appointmentSourceEnum = pgEnum('appointment_source', [
  'panel', 'bot', 'health_reminder', 'recurring',
]); // 'health_reminder' (plan 03), 'recurring' (plan 02 §6.6)
export const cancelledByEnum = pgEnum('cancelled_by', ['customer', 'business', 'system']);
// 'business' generaliza el 'salon' del plan 01 (acá cancela el salón/complejo/vet/gym);
// 'system' = expiración de hold de pago (plan 02 §6.2)

export const messageDirectionEnum = pgEnum('message_direction', ['in', 'out']);
export const messageStatusEnum = pgEnum('message_status', ['sent', 'delivered', 'read', 'failed']);
export const reminderKindEnum = pgEnum('reminder_kind', ['24h', '2h', '3h']);
export const reminderStatusEnum = pgEnum('reminder_status', ['scheduled', 'sent', 'cancelled', 'failed']);

// ── Vertical canchas (plan 02) ─────────────────────────────────────────
export const sportEnum = pgEnum('sport', ['padel', 'futbol5']);
export const pricingBandEnum = pgEnum('pricing_band', ['day', 'peak']);
export const paymentProviderEnum = pgEnum('payment_provider', ['mercadopago']);
export const paymentStatusEnum = pgEnum('payment_status', [
  'pending', 'approved', 'rejected', 'expired', 'refunded', 'forfeited',
]);
export const recurringStatusEnum = pgEnum('recurring_status', ['active', 'ended']);
export const waitlistStatusEnum = pgEnum('waitlist_status', [
  'waiting', 'offered', 'accepted', 'expired_offer', 'cancelled', 'fulfilled',
]);

// ── Vertical veterinarias (plan 03) ────────────────────────────────────
export const petSpeciesEnum = pgEnum('pet_species', ['dog', 'cat', 'other']);
export const targetSpeciesEnum = pgEnum('target_species', ['dog', 'cat', 'any']); // health_event_types
export const healthEventSourceEnum = pgEnum('health_event_source', ['in_clinic', 'external']);

// ── Vertical gimnasios (plan 04) ───────────────────────────────────────
export const membershipStatusEnum = pgEnum('membership_status', ['active', 'overdue', 'paused', 'cancelled']);
export const paymentMethodEnum = pgEnum('payment_method', ['mp', 'cash']);
export const dunningKindEnum = pgEnum('dunning_kind', ['pre_due', 'due', 'overdue']);
export const classBookingStatusEnum = pgEnum('class_booking_status', ['booked', 'cancelled', 'attended', 'no_show']);
export const bookingSourceEnum = pgEnum('booking_source', ['bot', 'panel']);
export const classWaitlistStatusEnum = pgEnum('class_waitlist_status', ['waiting', 'offered', 'converted', 'expired']);
```

`conversations.state` es `varchar`, NO enum: el catálogo de estados de la FSM (T2 §3.3) evoluciona por vertical y no amerita migración de enum por cada estado nuevo.

Helpers compartidos (`_shared.ts`):

```ts
import { uuid, timestamp } from 'drizzle-orm/pg-core';
import { uuidv7 } from 'uuidv7';

export const pk = () => uuid('id').primaryKey().$defaultFn(() => uuidv7());
export const tz = (name: string) => timestamp(name, { withTimezone: true });
export const timestamps = {
  createdAt: tz('created_at').notNull().defaultNow(),
  updatedAt: tz('updated_at').notNull().defaultNow().$onUpdate(() => new Date()),
};
```

---

## 2. `motor.ts` — las 12 tablas del motor (estado final, 4 verticales incorporados)

```ts
import {
  pgTable, uuid, text, varchar, integer, boolean, jsonb, time, date, index, uniqueIndex,
  primaryKey,
} from 'drizzle-orm/pg-core';
import { pk, tz, timestamps } from './_shared';
import * as e from './enums';

export const tenants = pgTable('tenants', {
  id: pk(),
  name: text('name').notNull(),
  slug: text('slug').notNull(),
  vertical: e.verticalEnum('vertical').notNull(),
  timezone: text('timezone').notNull().default('America/Argentina/Cordoba'),
  waPhoneNumberId: text('wa_phone_number_id'),
  waBusinessAccountId: text('wa_business_account_id'),
  waAccessToken: text('wa_access_token'),      // cifrado AES-256-GCM (core-db encryptSecret)
  mpAccessToken: text('mp_access_token'),      // cifrado — plan 02 §8 (credenciales MP por tenant)
  mpWebhookSecret: text('mp_webhook_secret'),  // cifrado
  mpUserId: text('mp_user_id'),
  reminderHours: integer('reminder_hours').array().notNull().default([24, 2]),
  settings: jsonb('settings').notNull().default({}),  // validado con el Zod schema del vertical
  status: e.tenantStatusEnum('status').notNull().default('active'),
  ...timestamps,
}, (t) => [
  uniqueIndex('tenants_slug_uq').on(t.slug),
  uniqueIndex('tenants_wa_phone_uq').on(t.waPhoneNumberId), // resolución tenant en webhook (T2 §7.1)
]);

export const users = pgTable('users', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  email: text('email').notNull(),
  name: text('name').notNull(),
  role: e.userRoleEnum('role').notNull().default('staff'),
  ...timestamps,
}, (t) => [
  uniqueIndex('users_email_uq').on(t.email), // global: Auth.js resuelve user→tenant por email (§6)
]);

export const staff = pgTable('staff', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  name: text('name').notNull(),
  active: boolean('active').notNull().default(true),
  sortOrder: integer('sort_order').notNull().default(0),
  ...timestamps,
}, (t) => [index('staff_tenant_idx').on(t.tenantId)]);

export const staffSchedules = pgTable('staff_schedules', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id), // agregado para RLS uniforme (§6)
  staffId: uuid('staff_id').notNull().references(() => staff.id, { onDelete: 'cascade' }),
  weekday: integer('weekday').notNull(),          // 0=domingo … 6=sábado — CHECK en §2.6
  startTime: time('start_time').notNull(),
  endTime: time('end_time').notNull(),
  ...timestamps,
}, (t) => [index('staff_schedules_staff_idx').on(t.staffId)]);

export const scheduleBlocks = pgTable('schedule_blocks', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  staffId: uuid('staff_id').references(() => staff.id),  // null = todo el negocio
  courtId: uuid('court_id'),  // plan 02 §6.1: bloqueo por cancha; FK en migración SQL (§2.6)
  startsAt: tz('starts_at').notNull(),
  endsAt: tz('ends_at').notNull(),
  reason: text('reason'),
  ...timestamps,
}, (t) => [index('schedule_blocks_tenant_range_idx').on(t.tenantId, t.startsAt)]);

export const services = pgTable('services', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  name: text('name').notNull(),
  durationMin: integer('duration_min').notNull(),
  priceCents: integer('price_cents').notNull(),
  active: boolean('active').notNull().default(true),
  sortOrder: integer('sort_order').notNull().default(0),
  ...timestamps,
}, (t) => [index('services_tenant_idx').on(t.tenantId)]);

export const staffServices = pgTable('staff_services', {
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id), // para RLS (§6)
  staffId: uuid('staff_id').notNull().references(() => staff.id, { onDelete: 'cascade' }),
  serviceId: uuid('service_id').notNull().references(() => services.id, { onDelete: 'cascade' }),
}, (t) => [primaryKey({ columns: [t.staffId, t.serviceId] })]);

export const customers = pgTable('customers', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  phone: varchar('phone', { length: 20 }).notNull(),   // E.164
  name: text('name'),
  optedOut: boolean('opted_out').notNull().default(false),
  lastInteractionAt: tz('last_interaction_at'),
  ...timestamps,
}, (t) => [
  uniqueIndex('customers_tenant_phone_uq').on(t.tenantId, t.phone), // upsert del orquestador (T2 §7.3)
]);

export const appointments = pgTable('appointments', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  staffId: uuid('staff_id').references(() => staff.id),       // null en vertical cancha
  serviceId: uuid('service_id').references(() => services.id), // null en vertical cancha
  startsAt: tz('starts_at').notNull(),
  endsAt: tz('ends_at').notNull(),
  status: e.appointmentStatusEnum('status').notNull().default('pending'),
  source: e.appointmentSourceEnum('source').notNull().default('panel'),
  cancelledBy: e.cancelledByEnum('cancelled_by'),
  notes: text('notes'),
  // ── columnas vertical cancha (plan 02 §3) — nullable, FKs en migración SQL (§2.6) ──
  courtId: uuid('court_id'),
  durationMin: integer('duration_min'),
  priceCents: integer('price_cents'),      // precio total congelado al reservar
  depositCents: integer('deposit_cents'),  // seña congelada al reservar
  paymentId: uuid('payment_id'),
  recurringBookingId: uuid('recurring_booking_id'),
  holdExpiresAt: tz('hold_expires_at'),    // solo con status awaiting_payment
  // ── columna vertical vet (plan 03 §3) ──
  petId: uuid('pet_id'),
  ...timestamps,
}, (t) => [
  index('appointments_tenant_starts_idx').on(t.tenantId, t.startsAt),
  index('appointments_customer_idx').on(t.tenantId, t.customerId, t.startsAt),
]);

export const conversations = pgTable('conversations', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  state: varchar('state', { length: 64 }).notNull().default('idle'),
  context: jsonb('context').notNull().default({}),
  expiresAt: tz('expires_at').notNull(),   // TTL 30 min (handoff: +4h)
  ...timestamps,
}, (t) => [
  uniqueIndex('conversations_customer_uq').on(t.tenantId, t.customerId), // 1 FSM por cliente (§6)
]);

export const messageLog = pgTable('message_log', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').references(() => customers.id),
  direction: e.messageDirectionEnum('direction').notNull(),
  waMessageId: text('wa_message_id'),      // null si el envío falló antes de obtener id
  type: varchar('type', { length: 32 }).notNull(),  // 'text'|'template'|'interactive'|…
  payload: jsonb('payload').notNull(),
  status: e.messageStatusEnum('status'),
  error: text('error'),
  ...timestamps,
}, (t) => [
  index('message_log_customer_idx').on(t.tenantId, t.customerId, t.createdAt),
  // UNIQUE parcial sobre wa_message_id en §2.6 (dedupe de Meta, T2 §7.2)
]);

export const reminders = pgTable('reminders', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id), // para RLS (§6)
  appointmentId: uuid('appointment_id').notNull().references(() => appointments.id, { onDelete: 'cascade' }),
  scheduledFor: tz('scheduled_for').notNull(),
  kind: e.reminderKindEnum('kind').notNull(),
  triggerRunId: text('trigger_run_id'),    // para cancelar el job en Trigger.dev
  status: e.reminderStatusEnum('status').notNull().default('scheduled'),
  ...timestamps,
}, (t) => [index('reminders_appointment_idx').on(t.appointmentId)]);
```

### 2.6 SQL crudo del motor (migración custom — lo que Drizzle no expresa)

```sql
CREATE EXTENSION IF NOT EXISTS btree_gist;  -- requerido por EXCLUDE con (uuid =, range &&)

-- No-solapamiento por profesional (plan 01 §3). awaiting_payment y pending SÍ bloquean.
ALTER TABLE appointments ADD CONSTRAINT appointments_no_overlap_staff
  EXCLUDE USING gist (staff_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
  WHERE (staff_id IS NOT NULL AND status NOT IN ('cancelled'));

-- No-solapamiento por cancha (plan 02 §3): misma barrera, otro recurso.
ALTER TABLE appointments ADD CONSTRAINT appointments_no_overlap_court
  EXCLUDE USING gist (court_id WITH =, tstzrange(starts_at, ends_at) WITH &&)
  WHERE (court_id IS NOT NULL AND status NOT IN ('cancelled'));

-- Dedupe de Meta (T2 §7.2): el orquestador hace INSERT y trata la violación como duplicado.
-- Parcial porque los envíos fallidos no tienen wa_message_id.
CREATE UNIQUE INDEX message_log_wa_message_id_uq ON message_log (wa_message_id)
  WHERE wa_message_id IS NOT NULL;

-- FKs cross-módulo de appointments y schedule_blocks: van por SQL para evitar el ciclo de
-- imports motor.ts ⇄ canchas.ts/vet.ts (decisión §6). Se crean EN LA MIGRACIÓN DE CADA VERTICAL.
ALTER TABLE appointments ADD CONSTRAINT appointments_court_fk
  FOREIGN KEY (court_id) REFERENCES courts(id);
ALTER TABLE appointments ADD CONSTRAINT appointments_payment_fk
  FOREIGN KEY (payment_id) REFERENCES payments(id);
ALTER TABLE appointments ADD CONSTRAINT appointments_recurring_fk
  FOREIGN KEY (recurring_booking_id) REFERENCES recurring_bookings(id);
ALTER TABLE appointments ADD CONSTRAINT appointments_pet_fk
  FOREIGN KEY (pet_id) REFERENCES pets(id);
ALTER TABLE schedule_blocks ADD CONSTRAINT schedule_blocks_court_fk
  FOREIGN KEY (court_id) REFERENCES courts(id);

-- CHECKs de sanidad
ALTER TABLE appointments    ADD CONSTRAINT appointments_range_ck CHECK (ends_at > starts_at);
ALTER TABLE schedule_blocks ADD CONSTRAINT schedule_blocks_range_ck CHECK (ends_at > starts_at);
ALTER TABLE staff_schedules ADD CONSTRAINT staff_schedules_weekday_ck CHECK (weekday BETWEEN 0 AND 6);
ALTER TABLE staff_schedules ADD CONSTRAINT staff_schedules_range_ck CHECK (end_time > start_time);
ALTER TABLE appointments    ADD CONSTRAINT appointments_hold_ck
  CHECK (status <> 'awaiting_payment' OR hold_expires_at IS NOT NULL);
```

---

## 3. Tablas por vertical

### 3.1 `canchas.ts` (plan 02 §3)

```ts
export const courts = pgTable('courts', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  name: text('name').notNull(),
  sport: e.sportEnum('sport').notNull(),
  indoor: boolean('indoor').notNull().default(false),
  active: boolean('active').notNull().default(true),
  sortOrder: integer('sort_order').notNull().default(0),
  ...timestamps,
}, (t) => [index('courts_tenant_idx').on(t.tenantId)]);

export const courtPricing = pgTable('court_pricing', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  courtId: uuid('court_id').notNull().references(() => courts.id, { onDelete: 'cascade' }),
  durationMin: integer('duration_min').notNull(),     // 60 | 90 | 120
  band: e.pricingBandEnum('band').notNull(),
  priceCents: integer('price_cents').notNull(),
  ...timestamps,
}, (t) => [uniqueIndex('court_pricing_uq').on(t.courtId, t.durationMin, t.band)]);

export const payments = pgTable('payments', {            // core-payments, agnóstico del vertical
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  provider: e.paymentProviderEnum('provider').notNull().default('mercadopago'),
  amountCents: integer('amount_cents').notNull(),
  currency: varchar('currency', { length: 3 }).notNull().default('ARS'),
  status: e.paymentStatusEnum('status').notNull().default('pending'),
  externalReference: text('external_reference').notNull(), // appointments.id | `${membershipId}:${periodStart}`
  mpPreferenceId: text('mp_preference_id'),
  mpPaymentId: text('mp_payment_id'),
  mpInitPoint: text('mp_init_point'),
  refundId: text('refund_id'),
  paidAt: tz('paid_at'),
  refundedAt: tz('refunded_at'),
  ...timestamps,
}, (t) => [
  index('payments_external_ref_idx').on(t.externalReference), // resolución en webhook (T2 §5)
  index('payments_tenant_created_idx').on(t.tenantId, t.createdAt), // pantalla Pagos
]);

export const paymentEvents = pgTable('payment_events', {  // log crudo de webhooks MP — idempotencia
  id: pk(),
  tenantId: uuid('tenant_id').references(() => tenants.id),    // nullable: evento no atribuible
  paymentId: uuid('payment_id').references(() => payments.id), // nullable hasta resolver
  provider: e.paymentProviderEnum('provider').notNull().default('mercadopago'),
  providerEventId: text('provider_event_id').notNull(),
  type: text('type').notNull(),
  payload: jsonb('payload').notNull(),
  processedAt: tz('processed_at'),
  ...timestamps,
}, (t) => [
  uniqueIndex('payment_events_provider_uq').on(t.provider, t.providerEventId), // T2 §5: dedupe
]);

export const recurringBookings = pgTable('recurring_bookings', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  courtId: uuid('court_id').notNull().references(() => courts.id),
  weekday: integer('weekday').notNull(),
  startTime: time('start_time').notNull(),
  durationMin: integer('duration_min').notNull(),
  priceCents: integer('price_cents').notNull(),       // precio pactado
  status: e.recurringStatusEnum('status').notNull().default('active'),
  endsOn: date('ends_on'),                            // null = sin fin
  generatedUntil: date('generated_until').notNull(),
  expiryNotifiedAt: tz('expiry_notified_at'),         // plantilla fijo_por_vencer "una sola vez" (§6)
  ...timestamps,
}, (t) => [index('recurring_bookings_tenant_status_idx').on(t.tenantId, t.status)]);

export const waitlist = pgTable('waitlist', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  sport: e.sportEnum('sport').notNull(),
  date: date('date').notNull(),
  timeFrom: time('time_from').notNull(),
  timeTo: time('time_to').notNull(),
  status: e.waitlistStatusEnum('status').notNull().default('waiting'),
  offeredAppointmentId: uuid('offered_appointment_id').references(() => appointments.id),
  offerExpiresAt: tz('offer_expires_at'),
  ...timestamps, // orden de la cola: created_at ASC (plan 02 §6.5)
}, (t) => [index('waitlist_match_idx').on(t.tenantId, t.date, t.status, t.createdAt)]);
```

### 3.2 `vet.ts` (plan 03 §3)

```ts
export const pets = pgTable('pets', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  name: text('name').notNull(),
  species: e.petSpeciesEnum('species').notNull(),
  breed: text('breed'),
  birthDate: date('birth_date'),         // aproximada
  notes: text('notes'),
  active: boolean('active').notNull().default(true),  // false = baja/fallecida: corta recordatorios
  ...timestamps,
}, (t) => [index('pets_customer_idx').on(t.tenantId, t.customerId)]);

export const healthEventTypes = pgTable('health_event_types', {  // catálogo POR TENANT, con seed
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  name: text('name').notNull(),                       // "Antirrábica", "Pipeta"
  defaultPeriodDays: integer('default_period_days').notNull(),
  species: e.targetSpeciesEnum('species').notNull().default('any'),
  serviceId: uuid('service_id').references(() => services.id),  // a precargar en el CTA de agendar
  active: boolean('active').notNull().default(true),
  sortOrder: integer('sort_order').notNull().default(0),
  ...timestamps,
}, (t) => [index('health_event_types_tenant_idx').on(t.tenantId)]);

export const healthEvents = pgTable('health_events', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  petId: uuid('pet_id').notNull().references(() => pets.id),
  eventTypeId: uuid('event_type_id').notNull().references(() => healthEventTypes.id),
  appliedOn: date('applied_on').notNull(),
  source: e.healthEventSourceEnum('source').notNull(),
  appointmentId: uuid('appointment_id').references(() => appointments.id),
  nextDueOn: date('next_due_on').notNull(),           // applied_on + period_days, pisable a mano
  reminderSentAt: tz('reminder_sent_at'),
  reminderSendCount: integer('reminder_send_count').notNull().default(0), // tope 3 re-envíos (§6)
  reminderDismissed: boolean('reminder_dismissed').notNull().default(false),
  ...timestamps,
}, (t) => [
  index('health_events_due_idx').on(t.tenantId, t.nextDueOn),   // query del job diario
  index('health_events_vigente_idx').on(t.petId, t.eventTypeId, t.appliedOn), // "más reciente por (pet,type)"
]);
```

### 3.3 `gym.ts` (plan 04 §3)

```ts
export const plans = pgTable('plans', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  name: text('name').notNull(),
  priceCents: integer('price_cents').notNull(),       // precio mensual
  weeklyClasses: integer('weekly_classes'),           // null = ilimitado
  active: boolean('active').notNull().default(true),
  sortOrder: integer('sort_order').notNull().default(0),
  ...timestamps,
}, (t) => [index('plans_tenant_idx').on(t.tenantId)]);

export const memberships = pgTable('memberships', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  planId: uuid('plan_id').notNull().references(() => plans.id),
  status: e.membershipStatusEnum('status').notNull().default('active'),
  billingDay: integer('billing_day').notNull(),       // 1-28, CHECK en §3.4
  currentPeriodEnd: date('current_period_end').notNull(),
  pausedAt: date('paused_at'),
  pauseReason: text('pause_reason'),
  cancelledAt: tz('cancelled_at'),
  ...timestamps, // UNIQUE parcial por customer en §3.4
}, (t) => [index('memberships_dunning_idx').on(t.tenantId, t.status, t.currentPeriodEnd)]);

export const membershipPayments = pgTable('membership_payments', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  membershipId: uuid('membership_id').notNull().references(() => memberships.id),
  amountCents: integer('amount_cents').notNull(),
  periodStart: date('period_start').notNull(),
  periodEnd: date('period_end').notNull(),
  method: e.paymentMethodEnum('method').notNull(),
  paymentId: uuid('payment_id').references(() => payments.id),  // NOT NULL si method='mp' (CHECK §3.4)
  registeredBy: uuid('registered_by').references(() => users.id), // NOT NULL si method='cash'
  paidAt: tz('paid_at').notNull(),
  ...timestamps,
}, (t) => [
  uniqueIndex('membership_payments_period_uq').on(t.membershipId, t.periodStart), // anti doble cobro
]);

export const dunningNotices = pgTable('dunning_notices', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  membershipId: uuid('membership_id').notNull().references(() => memberships.id),
  kind: e.dunningKindEnum('kind').notNull(),
  periodStart: date('period_start').notNull(),  // agregado: dedupe "1 escalón por período" (§6)
  sentAt: tz('sent_at').notNull(),
  triggerRunId: text('trigger_run_id'),
  mpPreferenceId: text('mp_preference_id'),
  ...timestamps,
}, (t) => [
  uniqueIndex('dunning_notices_dedupe_uq').on(t.membershipId, t.periodStart, t.kind),
  index('dunning_notices_recovery_idx').on(t.tenantId, t.sentAt), // métrica recupero 72h
]);

export const classSessions = pgTable('class_sessions', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  discipline: text('discipline').notNull(),
  staffId: uuid('staff_id').notNull().references(() => staff.id), // profesor
  weekday: integer('weekday').notNull(),
  startTime: time('start_time').notNull(),
  durationMin: integer('duration_min').notNull(),
  capacity: integer('capacity').notNull(),
  active: boolean('active').notNull().default(true),
  ...timestamps,
}, (t) => [index('class_sessions_tenant_idx').on(t.tenantId)]);

export const classOccurrences = pgTable('class_occurrences', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  classSessionId: uuid('class_session_id').notNull().references(() => classSessions.id),
  date: date('date').notNull(),
  startsAt: tz('starts_at').notNull(),               // desnormalizado para queries
  capacityOverride: integer('capacity_override'),
  cancelled: boolean('cancelled').notNull().default(false),
  ...timestamps,
}, (t) => [
  uniqueIndex('class_occurrences_uq').on(t.classSessionId, t.date), // job de materialización idempotente
  index('class_occurrences_agenda_idx').on(t.tenantId, t.startsAt),
]);

export const classBookings = pgTable('class_bookings', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  classOccurrenceId: uuid('class_occurrence_id').notNull().references(() => classOccurrences.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  status: e.classBookingStatusEnum('status').notNull().default('booked'),
  source: e.bookingSourceEnum('source').notNull(),
  cancelledAt: tz('cancelled_at'),
  ...timestamps, // UNIQUE parcial en §3.4; cupo validado en transacción con lock (plan 04 §3)
}, (t) => [index('class_bookings_occurrence_idx').on(t.classOccurrenceId, t.status)]);

export const classWaitlist = pgTable('class_waitlist', {
  id: pk(),
  tenantId: uuid('tenant_id').notNull().references(() => tenants.id),
  classOccurrenceId: uuid('class_occurrence_id').notNull().references(() => classOccurrences.id),
  customerId: uuid('customer_id').notNull().references(() => customers.id),
  position: integer('position').notNull(),
  status: e.classWaitlistStatusEnum('status').notNull().default('waiting'),
  offeredAt: tz('offered_at'),
  offerExpiresAt: tz('offer_expires_at'),
  ...timestamps,
}, (t) => [index('class_waitlist_queue_idx').on(t.classOccurrenceId, t.status, t.position)]);
```

### 3.4 SQL crudo de los verticales (migración custom)

```sql
-- Un customer no puede tener 2 memberships "vivas" (plan 04 §3). Parcial: las cancelled no cuentan.
CREATE UNIQUE INDEX memberships_one_live_per_customer_uq ON memberships (customer_id)
  WHERE status IN ('active', 'overdue', 'paused');

-- Un socio no se anota 2 veces a la misma clase; cancelar y re-anotarse SÍ se permite.
CREATE UNIQUE INDEX class_bookings_active_uq ON class_bookings (class_occurrence_id, customer_id)
  WHERE status = 'booked';

-- Coherencia método de pago ↔ trazabilidad (plan 04 §3 y §6.6)
ALTER TABLE membership_payments ADD CONSTRAINT membership_payments_method_ck
  CHECK ((method = 'mp' AND payment_id IS NOT NULL) OR (method = 'cash' AND registered_by IS NOT NULL));

ALTER TABLE memberships     ADD CONSTRAINT memberships_billing_day_ck CHECK (billing_day BETWEEN 1 AND 28);
ALTER TABLE class_sessions  ADD CONSTRAINT class_sessions_capacity_ck CHECK (capacity > 0);
ALTER TABLE class_sessions  ADD CONSTRAINT class_sessions_weekday_ck CHECK (weekday BETWEEN 0 AND 6);
ALTER TABLE recurring_bookings ADD CONSTRAINT recurring_bookings_weekday_ck CHECK (weekday BETWEEN 0 AND 6);
ALTER TABLE court_pricing   ADD CONSTRAINT court_pricing_duration_ck CHECK (duration_min IN (60, 90, 120));
ALTER TABLE waitlist        ADD CONSTRAINT waitlist_band_ck CHECK (time_to > time_from);
```

---

## 4. Justificación de índices no obvios

| Índice | Query que sirve |
|---|---|
| `health_events_due_idx (tenant_id, next_due_on)` | Job diario de vencimientos sanitarios (plan 03 §6): rango `[hoy−30, hoy+lead_days]` por tenant |
| `health_events_vigente_idx (pet_id, event_type_id, applied_on)` | "Evento vigente = el más reciente por (pet, type)" — `DISTINCT ON` / `ROW_NUMBER` sin scan |
| `message_log_customer_idx (tenant_id, customer_id, created_at)` | Historial en panel + anti-spam vet ("¿recibió plantilla sanitaria hace < min_gap_days?") |
| `appointments_tenant_starts_idx (tenant_id, starts_at)` | Agenda del día/semana, la pantalla más usada del panel |
| `appointments_customer_idx (tenant_id, customer_id, starts_at)` | "Mis turnos/reservas" del bot e historial en ficha de cliente |
| `memberships_dunning_idx (tenant_id, status, current_period_end)` | Job diario de la escalera: membresías no pausadas con vencimiento próximo/pasado |
| `dunning_notices_recovery_idx (tenant_id, sent_at)` | Métrica "recupero por avisos" (pagos dentro de 72h de un aviso) |
| `waitlist_match_idx (tenant_id, date, status, created_at)` | Cascada de lista de espera: `waiting` que matchean (date, franja), orden FIFO |
| `payments_external_ref_idx (external_reference)` | Webhook MP: resolver appointment/membership desde la notificación |
| `class_occurrences_agenda_idx (tenant_id, starts_at)` | Vista del día en Clases + job de no-show ("occurrence terminada hace 2h") |
| `tenants_wa_phone_uq (wa_phone_number_id)` | Paso 1 del orquestador (T2 §7): resolver tenant por cada mensaje entrante |

No se indexa `appointments.hold_expires_at`: la expiración del hold la dispara un job puntual de Trigger.dev por appointment (plan 02 §6.2), no un scan periódico.

---

## 5. Row Level Security

RLS es la **segunda barrera** (la primera es `tenantDb()` de T2 §1). La app setea `app.tenant_id` por transacción (`SET LOCAL`). Patrón único, mostrado completo para `appointments`:

```sql
ALTER TABLE appointments ENABLE ROW LEVEL SECURITY;
ALTER TABLE appointments FORCE ROW LEVEL SECURITY;  -- aplica también al owner de la tabla

CREATE POLICY tenant_isolation ON appointments
  USING (tenant_id = current_setting('app.tenant_id', true)::uuid)
  WITH CHECK (tenant_id = current_setting('app.tenant_id', true)::uuid);
-- current_setting(..., true) devuelve NULL si no está seteado → ninguna fila visible (fail-closed).
```

El mismo bloque, generado en loop en la migración, aplica a: `users, staff, staff_schedules, schedule_blocks, services, staff_services, customers, appointments, conversations, message_log, reminders, courts, court_pricing, payments, recurring_bookings, waitlist, pets, health_event_types, health_events, plans, memberships, membership_payments, dunning_notices, class_sessions, class_occurrences, class_bookings, class_waitlist`.

Quedan FUERA de RLS: `tenants` (la resolución de tenant ocurre antes de tener tenant_id; acceso solo vía service role) y `payment_events` (su tenant_id es nullable — log crudo de webhooks, accedido únicamente por el worker de core-payments con rol propio). Los jobs y webhooks usan un rol que setea `app.tenant_id` apenas resuelven el tenant.

---

## 6. Decisiones tomadas (no re-discutir al implementar)

1. **UUID v7 generado en la app**, no en DB (`$defaultFn(() => uuidv7())`). Razón: sin dependencia de versión de Postgres y el id se conoce antes del INSERT (external_reference de MP, buttonIds del bot).
2. **`appointment_status` y `appointment_source` son enums ÚNICOS compartidos** por los 4 verticales. Un salón nunca tendrá `awaiting_payment`, pero la alternativa (enum por vertical) rompería `core-bot`/`core-scheduling` compartidos. La FSM de cada vertical limita qué valores usa.
3. **`cancelled_by` usa `'business'` en lugar del `'salon'`** del plan 01 (el que cancela puede ser salón, complejo, vet o gym) y agrega `'system'` (expiración de hold, plan 02 §6.2).
4. **Columnas de canchas y vet viven en `appointments` desde el día 1 como nullable.** Una tabla por vertical duplicaría reminders/agenda/message_log (argumento del plan 02 §2). Las FKs cross-módulo (`court_id`, `payment_id`, `recurring_booking_id`, `pet_id`, y `schedule_blocks.court_id`) se crean por **SQL crudo en la migración del vertical correspondiente** para evitar ciclos de import entre `motor.ts` y los archivos de vertical.
5. **`tenant_id` se agregó a `staff_schedules`, `staff_services` y `reminders`** (los planes no lo listaban): toda tabla de negocio lo lleva para que la policy de RLS sea idéntica en todas (convención de 00-INDICE).
6. **`users.email` UNIQUE global** (no por tenant): Auth.js resuelve usuario→tenant por email en el magic link. Un humano que administre 2 tenants usa 2 emails (caso inexistente en MVP).
7. **`conversations` tiene UNIQUE `(tenant_id, customer_id)`**: una sola FSM viva por cliente; reiniciar = UPDATE, nunca segunda fila.
8. **`message_log.wa_message_id` es nullable con UNIQUE parcial** `WHERE wa_message_id IS NOT NULL`: los envíos fallidos se loguean sin id de Meta; el dedupe del orquestador (T2 §7.2) usa el unique.
9. **Sin `deleted_at` en ninguna tabla**: ningún plan pide soft-delete por timestamp; las bajas son flags `active` (staff, services, courts, pets, plans, class_sessions) y los datos transaccionales nunca se borran.
10. **UNIQUE parcial de memberships**: `(customer_id) WHERE status IN ('active','overdue','paused')` — interpreta "no puede tener 2 memberships vivas" del plan 04; las `cancelled` no bloquean re-altas. Es índice único parcial (Postgres no permite UNIQUE constraint con WHERE).
11. **`dunning_notices.period_start` (columna nueva)**: el plan 04 §6.1 exige dedupe "kind+período" pero no daba dónde anclar el período; UNIQUE `(membership_id, period_start, kind)` lo materializa.
12. **`health_events.reminder_send_count` (columna nueva)**: el tope de 3 re-recordatorios por evento vigente (plan 03 §6) necesita un contador; `reminder_sent_at` solo guarda el último envío.
13. **`recurring_bookings.expiry_notified_at` (columna nueva)**: "plantilla `fijo_por_vencer` una sola vez" (plan 02 §6.6) requiere marca persistente.
14. **Especies como 2 enums** (`pet_species` con `other`, `target_species` con `any`): mezclar `other`/`any` en un enum permitiría estados sin sentido (un pet `any`, un tipo de evento `other`).
15. **Credenciales MP del tenant como columnas cifradas en `tenants`** (`mp_access_token`, `mp_webhook_secret`, `mp_user_id`), igual que las de WhatsApp — no en `settings` jsonb (plan 02 §8 lo exige).
16. **Las fechas de cuotas y calendario sanitario son `date` (fecha civil)**, no timestamptz, siguiendo `LocalDate` de T2 §6: evita bugs de medianoche/zona.
17. **Naming**: snake_case en DB, camelCase en TypeScript (mapeo explícito columna por columna); tablas en plural; índices `tabla_proposito_idx`, uniques `tabla_proposito_uq`, constraints `tabla_proposito_ck/fk`.

---

## 7. Seed de desarrollo (`packages/core-db/src/seed.ts`)

Crea **4 tenants, uno por vertical**, con credenciales WhatsApp/MP dummy y settings default del Zod schema de cada plan:

- **`salon-demo`** (salon): 2 staff con horario semanal (uno con bloque partido al mediodía), 5 services (corte, color, etc.), matriz staff_services, 6 customers, ~15 appointments pasados/futuros en estados variados, 1 schedule_block de vacaciones, reminders coherentes con los turnos futuros.
- **`complejo-demo`** (cancha): 3 courts pádel + 2 futbol5, grilla `court_pricing` completa (60/90 × day/peak), 1 recurring_booking activo con 4 ocurrencias generadas, 1 appointment `awaiting_payment` con hold vivo y su payment `pending`, 2 anotados en `waitlist` para hoy a la noche, 1 payment `approved` y 1 `forfeited`.
- **`vet-demo`** (vet): catálogo `health_event_types` del seed del plan 03 (antirrábica 365, séxtuple 365 dog, triple felina 365 cat, antiparasitario 90, pipeta 30, baño 60), 4 customers con 6 pets (1 inactiva), `health_events` con vencimientos a −10, +5 y +40 días para probar el job, 1 turno `source='health_reminder'` completado (alimenta la métrica de facturación recuperada).
- **`gym-demo`** (gym): 3 plans, 10 customers con memberships en los 4 estados (incluye 1 overdue con `dunning_notices` pre_due+due enviados), membership_payments mixtos cash/mp, 6 class_sessions con occurrences materializadas a 7 días, 1 clase llena con 2 en class_waitlist.

El seed es **idempotente** (borra y recrea los 4 tenants demo por slug) y corre con `pnpm db:seed`. Las fechas se generan relativas a `now()` para que los estados (vencido, próximo, hold vivo) sigan siendo válidos cualquier día que se corra.
