# Plan de Desarrollo 06 — Prospector + Demo Bot + Landings (la Máquina de Ventas)

> Herramienta interna de prospección y ventas del portfolio: base de leads con score, mini-CRM con seguimientos forzados, Demo Bot que captura interesados con consentimiento, landings por vertical y liquidación de comisiones de vendedores.
> **Audiencia**: IA/dev sin contexto. Prerrequisitos: motor de `01-NUCLEO-MOTOR.md` en producción (el Demo Bot vive sobre él). Hereda stack y convenciones de `00-INDICE.md` y los contratos de `tecnico/T2-INTERFACES-PAQUETES.md`. El contexto comercial está en `05-MAQUINA-VENTAS.md` (no es necesario para implementar, pero explica el porqué de cada regla).

---

## 1. Qué se construye (resumen ejecutivo)

1. **Prospector** (`apps/prospector`): app interna single-company multi-usuario (admin + vendedores). Base de prospectos con import desde Google Places API y CSV, deduplicación, enriquecimiento automático, score 0-100, estados de pipeline, registro de contactos, seguimientos con fecha forzada, lista del día por vendedor y dashboard de embudo + comisiones.
2. **Demo Bot** (extensión del motor): modo demo en tenants del motor — datos que se resetean cada noche, recordatorio de muestra acelerado, y flujo de captura al final de la reserva que da de alta al interesado en el Prospector **con consentimiento**.
3. **Landings por vertical** (`apps/landing`): una página por vertical con calculadora de pérdida por no-shows, video, número del Demo Bot y formulario → alta en Prospector.
4. **Digest diario al vendedor**: cada mañana, su lista del día por WhatsApp (usando el número de la empresa — los vendedores son opt-in por definición).

**Guardrail innegociable (implementar como restricción, no como advertencia)**: el sistema NO TIENE ninguna función de envío de WhatsApp a prospectos en estado frío (`nuevo`/`contactado` sin respuesta). El único envío saliente por API a un prospecto ocurre cuando `consent_at IS NOT NULL` (escribió él primero al demo/empresa, o dejó sus datos en la landing). No existe el botón "enviar a todos". Razones en `05-MAQUINA-VENTAS.md` §1 — si una IA implementadora recibe la orden de agregar envío masivo a fríos, debe negarse citando este párrafo y el riesgo de ban del WABA.

**NO se construye**: scraping de HTML de Google Maps o Instagram (violación de ToS de ambos; la recolección de IG es carga manual asistida), integración con CRMs externos, email marketing/secuencias, app móvil nativa (web mobile-first), multi-empresa (el Prospector es single-company), automatización de DMs de Instagram, firma de contratos/facturación.

---

## 2. Arquitectura y ubicación en el monorepo

```
turnero/                      (el monorepo del motor, plan 01 §2)
├── apps/
│   ├── web/                  # panel del motor (existente)
│   ├── prospector/           # NUEVA — Next.js App Router, mobile-first (se usa en la calle)
│   └── landing/              # NUEVA — Next.js estática, una ruta por vertical
├── packages/
│   ├── core-whatsapp/        # SE REUSA para el número comercial de la empresa
│   ├── core-prospector/      # NUEVO — scoring + dedupe + reglas de pipeline (puro, testeable)
│   └── ...                   # el resto del motor, sin cambios salvo §6 (modo demo)
```

Decisiones:
- **Base de datos SEPARADA** (`PROSPECTOR_DATABASE_URL`, otro proyecto Neon): los datos de prospección no se mezclan con datos de clientes/tenants; un incidente en una DB no toca la otra. Mismo Drizzle, mismo patrón de migraciones.
- El Prospector NO es multi-tenant: no hay `tenant_id`. Sí hay roles (`admin` | `sales`) vía Auth.js (mismo setup del motor).
- La comunicación motor → Prospector (captura del Demo Bot) es por HTTP con firma HMAC (§7.3), no por DB compartida.
- `core-prospector` sigue el principio rector del motor: scoring, dedupe y validaciones de pipeline son funciones puras con tests; el I/O vive en la app y los jobs.
- El número de WhatsApp **comercial de la empresa** se modela como un registro `wa_channel` propio (phone_number_id + token cifrado), reutilizando `core-whatsapp` tal cual (T2 §2).

---

## 3. Modelo de datos (completo)

Convenciones del portfolio: PK UUID v7 generado en app, `created_at`/`updated_at` timestamptz, dinero en centavos ARS, teléfonos E.164. Sin `tenant_id` (§2).

```
users                              -- vendedores y admin (Auth.js)
  id, email, name, role ('admin'|'sales'), active bool,
  personal_phone (E.164, para el digest diario), zone_focus text nullable

zones                              -- catálogo editable de zonas de trabajo
  id, name ('Nueva Córdoba', 'Cerro', ...), city default 'Córdoba', sort_order

prospects
  id,
  business_name text NOT NULL,
  vertical_fit ('salon'|'barberia'|'cancha'|'vet'|'gym'|'estetica'|'otro'),
  zone_id FK zones nullable, address text nullable,
  lat double nullable, lng double nullable,        -- para la ruta del día
  google_place_id text nullable UNIQUE,            -- ancla de dedupe #1
  phone text nullable,                             -- E.164 normalizado; puede ser fijo
  phone_normalized text nullable,                  -- solo dígitos, índice UNIQUE parcial (dedupe #2)
  phone_is_mobile bool nullable,                   -- prefijo 9 en E.164 AR
  whatsapp_confirmed bool default false,           -- se detectó wa.me / botón WhatsApp
  instagram_handle text nullable, ig_followers int nullable,
  ig_active bool nullable,                         -- posteó en los últimos 30 días (carga manual)
  website text nullable,
  google_rating numeric(2,1) nullable, google_reviews_count int nullable,
  has_booking_system bool default false,           -- se detectó AgendaPro/Booksy/Fresha/Reservo (§5.2)
  booking_system_name text nullable,
  score int NOT NULL default 0,                    -- §4, recalculado por trigger de app
  status ('nuevo'|'contactado'|'respondio'|'demo_agendada'|'demo_hecha'|
          'negociando'|'cliente'|'perdido'|'no_molestar') default 'nuevo',
  lost_reason text nullable,                       -- obligatorio al pasar a 'perdido' (enum libre + nota)
  consent_at timestamptz nullable,                 -- ver guardrail §1; lo setean demo bot, landing o respuesta entrante
  consent_source ('demo_bot'|'landing'|'inbound_wsp'|'en_persona') nullable,
  assigned_to FK users nullable,
  next_followup_on date nullable,                  -- regla §5.4
  owner_name text nullable, notes text,
  source ('places'|'csv'|'manual'|'demo_bot'|'landing'|'referido'),
  import_batch_id FK import_batches nullable

prospect_touches                   -- TODO contacto, humano o sistémico
  id, prospect_id FK NOT NULL, user_id FK users nullable,   -- null = touch automático
  channel ('visita'|'ig_dm'|'wsp_manual'|'wsp_api'|'llamada'|'email'|'demo_bot'|'landing'|'referido'),
  direction ('out'|'in'),
  summary text NOT NULL,            -- "DM enviado con mención a sus reseñas", "Pidió precio"
  status_after text nullable,       -- snapshot del status tras el touch (auditoría del pipeline)
  at timestamptz NOT NULL

import_batches
  id, source ('places'|'csv'), query text nullable,   -- "peluquería + Nueva Córdoba"
  total_rows int, created int, merged int, skipped int,
  report jsonb,                     -- detalle de dedupe por fila
  created_by FK users, created_at

sales                              -- una venta cerrada = un tenant nuevo en el motor
  id, prospect_id FK UNIQUE, vendor_id FK users,
  vertical text, plan_name text, mrr_cents int,
  motor_tenant_id uuid nullable,    -- id del tenant creado en la DB del motor (referencia blanda, sin FK)
  closed_at date, churned_at date nullable   -- se marca a mano cuando el cliente se va

commission_rules                   -- editable por admin; la vigente es la de effective_from más reciente
  id, pct_first_month int default 100, pct_recurring int default 10,
  recurring_months int default 12, effective_from date

commission_entries                 -- ledger inmutable, generado por job mensual (§8)
  id, sale_id FK, vendor_id FK, period date,        -- primer día del mes liquidado
  kind ('first_month'|'recurring'), amount_cents int,
  UNIQUE (sale_id, period, kind)

wa_channel                         -- el/los números de WhatsApp de la EMPRESA (no de tenants)
  id, label ('comercial'), phone_number_id, access_token (cifrado), active bool

wa_messages                        -- message_log local (mismo shape que el del motor, T1)
  id, prospect_id nullable, direction, wa_message_id UNIQUE parcial, type, payload jsonb, status, error
```

Integridad y reglas en DB/app:
- Dedupe en el import (§5.1): match por `google_place_id`, luego por `phone_normalized`, luego fuzzy por (`business_name` normalizado + zona) → en match, MERGE (rellenar campos vacíos, nunca pisar datos cargados a mano) y contar en `import_batches.merged`.
- Transición a `perdido` exige `lost_reason`; a `cliente` exige crear el registro en `sales` (misma transacción).
- `no_molestar` es terminal: el prospecto desaparece de toda lista y sugerencia; solo admin puede revertirlo.
- CHECK de aplicación (no DB, para poder migrar datos viejos): `status IN ('respondio','demo_agendada','demo_hecha','negociando') → next_followup_on NOT NULL`. El job nocturno (§8 `violations.sweep`) lista las violaciones en el dashboard del admin.

---

## 4. Scoring (en `core-prospector`, puro y testeado)

```ts
export function computeScore(p: ProspectScoringInput, w: ScoreWeights): number; // clamp 0-100
```

Pesos default (editables por admin en una tabla `settings` clave-valor, schema Zod):

| Señal | Condición | Puntos |
|---|---|---|
| Instagram existe | `instagram_handle != null` | +15 |
| Instagram activo | `ig_active = true` | +10 |
| Seguidores | `ig_followers >= 1000` | +10 |
| Volumen real de clientes | `google_reviews_count >= 100` / `>= 30` | +20 / +10 |
| Rating sano | `4.0 <= google_rating <= 5.0` | +5 |
| WhatsApp confirmado | `whatsapp_confirmed = true` | +10 |
| Celular (whatsappeable) | `phone_is_mobile = true` | +10 |
| Sitio web propio | `website != null` (invierte en lo digital) | +5 |
| **Ya tiene sistema de reservas** | `has_booking_system = true` | **−30** |
| Datos mínimos faltantes | sin teléfono Y sin instagram | −25 (incontactable) |

Recompute: al editar cualquier campo de señal (hook en la capa de datos) y en el job de enriquecimiento. El score NUNCA se edita a mano; lo que se edita son las señales.

---

## 5. Funcionalidad del Prospector

### 5.1 Import de prospectos

**Google Places API (New)** — la única fuente automatizada:
- Pantalla admin: elegir categoría de búsqueda (texto, ej. "peluquería") + zona (del catálogo, con lat/lng/radio) → preview → confirmar → job `places.import-batch`.
- El job usa Text Search (`places:searchText`) paginado + Place Details por resultado con **field mask mínima**: `id, displayName, formattedAddress, location, nationalPhoneNumber, websiteUri, rating, userRatingCount`. Mapear a `prospects` con `source='places'`, normalizar teléfono a E.164 AR (`+549...` si es celular; heurística: tras quitar 0/15, los celulares argentinos llevan 9 tras +54).
- Dedupe del §3. Resultado: `import_batches.report` visible en pantalla ("82 traídos: 41 nuevos, 28 fusionados, 13 sin datos de contacto").
- Costo: Place Details se cobra por request — el job respeta un tope configurable de requests/día (`settings.places_daily_cap`, default 300) y reanuda al día siguiente.

**CSV** (relevamiento a pie, guías, planillas de un data-entry): columnas documentadas en la pantalla (`business_name, vertical_fit, zone, phone, instagram, notes, ...`), mismo pipeline de dedupe.

**Alta manual rápida** (en la calle, mobile): formulario de 6 campos + foto opcional de la vidriera (Vercel Blob) + botón "estoy acá" (geolocalización del navegador → lat/lng y zona sugerida).

### 5.2 Enriquecimiento automático (job `enrich.prospect`)

Para cada prospecto con `website` o `instagram_handle` y `has_booking_system` aún no determinado:
1. Fetch del `website` (timeout 8s, solo GET, user-agent propio): buscar en el HTML los patrones `agendapro|booksy|fresha|reservo|calendly|turnos.\w+` → `has_booking_system = true` + nombre; buscar `wa.me/|api.whatsapp.com` → `whatsapp_confirmed = true` y extraer el número si difiere del cargado.
2. Instagram NO se fetchea (ToS): `ig_followers`/`ig_active` son carga manual — la pantalla de detalle tiene esos dos campos con acceso en un toque y el link al perfil para mirarlo.
3. Recompute del score. Errores de fetch → `has_booking_system` queda null y se reintenta a los 7 días (máx 2 intentos).

### 5.3 Pipeline y registro de contactos

- Vista pipeline (kanban por `status`) y vista lista con filtros (vertical, zona, score, vendedor, vencidos).
- Detalle del prospecto: datos + score con desglose de señales + historial de touches + acciones rápidas de un toque, pensadas para el celular en la calle: «Visité» / «Mandé DM» / «Respondió» / «Agendé demo» — cada una crea el `prospect_touch`, pide un summary corto (o lo arma solo: "Visita registrada en la puerta") y, si corresponde, pide la transición de status y el `next_followup_on` (date-picker con sugerencias +2d/+4d/+1sem).
- **Mensaje sugerido para DM/WhatsApp manual**: plantilla por vertical con variables del prospecto ("Hola {owner_name|'!'}, soy {vendedor} 👋 Vi {business_name} en {zone} — {gancho_por_señal}") donde `gancho_por_señal` se elige por las señales (tiene muchas reseñas → "vi que tienen un monton de movimiento"; sin sistema → "¿siguen manejando los turnos por WhatsApp a mano?"). El vendedor lo COPIA y lo manda desde SU teléfono (canal `wsp_manual` o `ig_dm`). El sistema no lo envía — guardrail §1.
- Asignación: admin asigna prospectos/zonas a vendedores; un vendedor solo ve los suyos + los sin asignar de su zona.

### 5.4 Lista del día y ruta

- Query "lista del día" por vendedor: (a) followups con `next_followup_on <= hoy`, primero; (b) top-N `nuevo` por score de sus zonas (N = `settings.daily_new_target`, default 10).
- Modo ruta (para visitas): los seleccionados con lat/lng se ordenan por cercanía encadenada simple (nearest-neighbor desde la ubicación actual — no hace falta TSP perfecto) y cada tarjeta tiene deep-link `https://maps.google.com/?q=lat,lng`.
- Regla de oro (§3): nada en estados calientes sin followup fechado. La UI no permite guardar la transición sin fecha; el sweep nocturno reporta las que se filtraron.

### 5.5 WhatsApp entrante y saliente (solo con consentimiento)

- Webhook del número comercial (mismo patrón del motor, T3 §2.1, reusando `core-whatsapp`): mensaje entrante → match por teléfono contra `prospects` → si existe: touch `in` automático + `consent_at = now()` si era null + status a `respondio` si estaba frío + notificación al vendedor asignado (digest inmediato simple: mensaje de WhatsApp al `personal_phone` del vendedor con nombre y link al detalle). Si no existe: crear prospecto `source='manual'`, sin asignar, bandeja del admin.
- NO hay bot acá: las conversaciones de venta las lleva el humano desde la app (vista conversación simple sobre `wa_messages`, envío de texto libre dentro de la ventana de 24h). Fuera de ventana: una única plantilla utility `seguimiento_propuesta` ("Hola {{1}}, te escribo por la propuesta de {{2}} que vimos. ¿Seguimos?") con tope de 1 envío por prospecto por semana, aplicado en código.

### 5.6 Dashboard

- Embudo por status (conteo + valor potencial = Σ mrr estimado por vertical desde `settings.mrr_estimate_por_vertical`).
- Conversión por canal de primer touch, por vertical, por vendedor (touches → demos → clientes).
- Vencidos y violaciones de la regla de followup.
- Comisiones del mes por vendedor (de `commission_entries`) + detalle.

---

## 6. Demo Bot — extensión del motor (se implementa en el repo del motor)

Cambios en `apps/web` + paquetes del motor, detrás de `tenants.settings.demo` (Zod: `{ demo: z.object({ enabled: z.boolean(), resetHour: z.number().default(4), prospectorWebhook: z.string().url() }).optional() }`):

1. **Recordatorio acelerado**: en tenants demo, al crear un turno por el bot, los reminders se programan a `now() + 2 min` (en vez de 24h/2h) con el prefijo "🧪 *Modo demo — así le llega a tu cliente:*" en el body. Implementación: rama en el job `reminders.dispatch`-scheduler cuando `settings.demo.enabled`.
2. **Flujo de captura** (estados nuevos de la FSM, solo vertical demo — se agregan al catálogo T2 §3.3 como `demo_pitch`, `demo_capture_name`, `demo_capture_biz`): tras completarse el Flujo A del demo (el usuario tocó «Confirmo» en el recordatorio de muestra), el bot envía:
   > "¿Viste qué simple? 🤯 Eso mismo recibirían tus clientes. ¿Te gustaría tenerlo en tu negocio?" — botones: «Sí, me interesa» / «Solo estaba probando»
   - «Sí, me interesa» → "¡Genial! ¿Cómo se llama tu negocio y de qué rubro es?" → texto libre → efecto nuevo `Effect: { type: 'demo_lead'; phone; name?; bizText: string }`.
   - «Solo estaba probando» → "¡Gracias por probar! Si algún día lo querés, escribinos acá 😉" (y el teléfono NO se manda al Prospector).
3. **Entrega del lead**: el orquestador ejecuta `demo_lead` haciendo POST a `settings.demo.prospectorWebhook` (= `https://prospector…/api/leads/demo`) con `{ phone, bizText, demoVertical, at }` firmado HMAC-SHA256 (`PROSPECTOR_WEBHOOK_SECRET`). Reintentos vía job (3 con backoff); si agota, alerta en panel admin del motor.
4. **Reset nocturno** (job `demo.reset` a `resetHour`): borra customers/appointments/conversations/message_log del tenant demo creados ese día y re-aplica el seed demo. Los leads ya viajaron al Prospector — acá no queda nada personal.
5. Un tenant demo por vertical, números publicados en landings y material de venta.

En el Prospector, `POST /api/leads/demo` (verifica HMAC): upsert por teléfono → `consent_at = now()`, `consent_source = 'demo_bot'`, status `respondio`, touch automático con el `bizText`, intento de inferir `vertical_fit` por keywords del texto, bandeja "leads calientes" del admin para asignar. **Estos leads se contactan dentro de la hora** (la pantalla los muestra con cronómetro) — un lead de demo caliente se enfría en horas.

## 7. Landings (`apps/landing`)

- Una ruta por vertical (`/peluquerias`, `/canchas`, `/veterinarias`, `/gimnasios`), estáticas, mobile-first, mismo shadcn/Tailwind.
- Estructura fija: titular de dolor → **calculadora** (3 inputs: turnos/día, ticket promedio, % de ausencia con slider; output grande: "$X que perdés por mes"; los defaults por vertical salen de `Gestion Wssp/PRODUCTO.md`) → video embed 90s → CTA primario "Probalo ahora por WhatsApp" (`https://wa.me/<demo>?text=Hola!%20Quiero%20probar%20la%20demo`) → CTA secundario formulario (nombre, negocio, teléfono, vertical) → `POST /api/leads/landing` del Prospector (mismo esquema HMAC, `consent_source='landing'`).
- Tracking mínimo sin cookies invasivas: parámetro `?src=` (ig, visita, referido) persistido al lead. Nada de píxeles en el MVP.

---

## 8. Jobs (Trigger.dev, proyecto del Prospector salvo los marcados [motor])

| Job | Trigger | Qué hace | Idempotencia |
|---|---|---|---|
| `places.import-batch` | evento (pantalla admin) | §5.1: search + details + dedupe + report; respeta `places_daily_cap`, reanuda solo | re-run del batch re-dedupea: 0 duplicados |
| `enrich.prospect` | evento (post-import) + cron semanal de pendientes | §5.2 | flags determinísticos, re-run inocuo |
| `score.recompute-all` | cron diario 05:00 | recalcula score de todos (por si cambian pesos) | puro |
| `followups.daily-digest` | cron 08:30 | arma la lista del día por vendedor y la manda a su `personal_phone` vía `wa_channel` (texto, son opt-in) | dedupe por (vendor, fecha) |
| `violations.sweep` | cron diario 21:00 | prospectos calientes sin `next_followup_on` o vencidos >3 días → lista al admin | puro |
| `commissions.liquidate` | cron mensual día 1 | por cada `sale` activa: entry `first_month` (mes de cierre) o `recurring` (meses 2..N, si no `churned`) según `commission_rules` vigente | UNIQUE (sale, period, kind) |
| `leads.webhook-retry` | delayed | reintentos de §6.3 | por delivery id |
| [motor] `demo.reset` | cron diario | §6.4 | borra por created_at del día, re-seed determinista |

---

## 9. Server Actions (catálogo)

Convención del portfolio (T3 §4): Zod input/output, naming `<modulo>.<accion>`, usuario del session token.

| Módulo | Actions |
|---|---|
| prospects | `create`, `update`, `changeStatus` (valida reglas §3: lost_reason, followup, sale en transacción), `assign`, `bulkAssign`, `merge` (fusión manual de duplicados) |
| touches | `log` (channel, summary, statusAfter?, nextFollowupOn?) |
| imports | `startPlacesImport(query, zoneId)`, `uploadCsv`, `getBatchReport` |
| daily | `getMyList(date)`, `getRoute(prospectIds, fromLatLng)` |
| wa | `sendText(prospectId, text)` (valida ventana 24h y `consent_at` — guardrail §1), `sendFollowupTemplate(prospectId)` (tope semanal) |
| sales | `close(prospectId, vertical, planName, mrrCents)`, `markChurned(saleId)` |
| commissions | `getStatement(vendorId, period)`, `updateRules` (admin) |
| settings | `updateScoreWeights`, `updateGeneral` (caps, targets, mrr estimados) |
| leads (HTTP público) | `POST /api/leads/demo`, `POST /api/leads/landing` — HMAC, rate-limit 10/min/IP, 200 siempre (errores se loguean, no se exponen) |

---

## 10. Variables de entorno

```
PROSPECTOR_DATABASE_URL=
AUTH_SECRET= / AUTH_GOOGLE_ID= / AUTH_GOOGLE_SECRET=
GOOGLE_PLACES_API_KEY=            # restringida por API y por IP del server
META_APP_SECRET= / META_WEBHOOK_VERIFY_TOKEN=   # webhook del número comercial
ENCRYPTION_KEY=                   # token del wa_channel
PROSPECTOR_WEBHOOK_SECRET=        # HMAC compartido con el motor y la landing
TRIGGER_SECRET_KEY=
BLOB_READ_WRITE_TOKEN=            # fotos de vidriera
```

---

## 11. Fases de implementación

### Fase P0 — Esqueleto (2-3 días)
`apps/prospector` + DB propia + schema §3 completo con migraciones y seed (2 users, 6 zonas, 30 prospectos fake variados, 1 commission_rule), Auth con roles, CLAUDE.md.
✅ Login admin/sales con vistas distintas; seed visible en lista con filtros básicos.

### Fase P1 — Prospectos + import CSV + dedupe (1 semana)
CRUD completo, alta manual rápida mobile con foto y geolocalización, import CSV con pipeline de dedupe y report, `core-prospector` (dedupe + score §4) con tests.
✅ CSV de 100 filas con 20 duplicados → report exacto (test con fixture); score calculado y desglosado en el detalle; merge manual funciona; alta en la calle en <30s desde el celular.

### Fase P2 — Places + enriquecimiento (1 semana)
Import por Places API con cap diario y reanudación, job de enriquecimiento (detección de booking systems y wa.me), recompute de score.
✅ Import real de una categoría+zona de Córdoba termina con report coherente; prospecto con web de AgendaPro queda `has_booking_system=true` y score −30 (test con HTML fixture); cap diario corta y reanuda (test con clock simulado).

### Fase P3 — Pipeline, lista del día y guardrails (1-1,5 semanas)
Kanban + transiciones con reglas (§3), touches con acciones rápidas, followups forzados, lista del día + modo ruta, mensajes sugeridos por señal, asignación por vendedor, `violations.sweep`.
✅ Imposible pasar a `negociando` sin followup fechado (test + UI); imposible pasar a `cliente` sin crear la venta (transacción testeada); lista del día mezcla vencidos primero + top score; la ruta ordena por cercanía y abre Maps; sweep reporta violaciones plantadas a propósito.

### Fase P4 — WhatsApp comercial + digest (1 semana)
`wa_channel`, webhook entrante con match/alta y consent, vista conversación con envío en ventana, plantilla `seguimiento_propuesta` con tope, digest diario a vendedores.
✅ Mensaje entrante de un número desconocido crea prospecto con `consent_at`; de uno frío conocido lo pasa a `respondio` y notifica al vendedor; envío fuera de ventana solo permite la plantilla y respeta el tope semanal (test); intento de `wa.sendText` a prospecto sin consent → error de validación (test del guardrail §1); digest llega 08:30 con la lista correcta.

### Fase P5 — Demo Bot + landings (1-1,5 semanas, mitad en el repo del motor)
[motor] modo demo completo (§6: reminder 2 min, estados de captura con tests de FSM, webhook firmado con retry, reset nocturno). [prospector] endpoints de leads + bandeja caliente con cronómetro. [landing] las 4 páginas con calculadora y forms.
✅ E2e completo demostrable: persona ajena reserva en el tenant demo → recibe el recordatorio a los 2 min → confirma → acepta el pitch → aparece en la bandeja del Prospector con consent y vertical inferido en <1 min; «Solo estaba probando» NO genera lead (test); reset nocturno deja el demo impecable (test); la calculadora calcula y el form crea el lead con `src`.

### Fase P6 — Dashboard + comisiones (3-5 días)
Embudo, conversión por canal/vertical/vendedor, `sales` + `commissions.liquidate` + statements.
✅ Las métricas cuadran contra un dataset conocido (test); cerrar una venta en marzo genera `first_month` en marzo y `recurring` abril→marzo siguiente, frenando en `churned_at` (test del job con fechas simuladas); statement por vendedor exportable (CSV alcanza).

**Total estimado: 5-6,5 semanas.** P0-P3 son útiles desde el día uno aunque P4-P6 no existan todavía (la base + la lista del día ya cambian la prospección a pie).

---

## 12. Puntos a verificar al implementar

- **Google Places API (New)**: nombres vigentes de endpoints (`places:searchText`, Place Details), field masks, pricing por SKU y política de almacenamiento de datos (los ToS restringen cachear ciertos campos — para una herramienta interna de prospección el uso es razonable, pero confirmar qué campos permiten retención; `place_id` siempre se puede guardar).
- Heurística de normalización de celulares argentinos (+54 9) contra casos reales de Córdoba (fijos 351-4xx vs celulares).
- Política vigente de Meta sobre el contenido del digest a vendedores (es texto libre a opt-in propios — sin riesgo, pero si crece el equipo conviene plantilla utility aprobada).
- Límite de la ventana de geolocalización del navegador en iOS Safari (para el botón "estoy acá").
- Todo lo del motor (§9 del plan 01) para la parte Demo Bot.

## 13. Extensiones previstas (no implementar)

- **Referidos automatizados** (NPS día 30 dentro de los tenants del motor + código de referido) — diseño en `05-MAQUINA-VENTAS.md` §3.2; cuando se haga, el lead referido entra por el mismo endpoint de leads con `source='referido'`.
- Secuencias de email para verticales profesionales (contables).
- Detección automática de actividad de Instagram si algún día existe API pública viable (hoy no — no scrapear).
- Comisiones multi-nivel / supervisores; exportación contable.
- App "modo vendedor" offline-first si el equipo de calle crece.

El diseño no bloquea ninguna: los leads entran todos por un endpoint común, el ledger de comisiones es extensible por `kind`, y los pesos del score son data, no código.
