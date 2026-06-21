# Plan de Desarrollo 03 — Vertical Veterinarias

> Turnos por WhatsApp + calendario sanitario de mascotas para veterinarias de barrio (Córdoba, Argentina). El diferencial vendible: recordatorios sanitarios que GENERAN ventas — "Firulais tiene la antirrábica vencida, ¿agendamos?".
> **Audiencia**: IA/dev sin contexto. Prerrequisito: motor de `01-NUCLEO-MOTOR.md` implementado hasta Fase 5 (motor completo en producción).

## 1. Qué se construye (resumen ejecutivo)

- **Mascotas como entidad**: cada `customer` (dueño, identificado por teléfono) tiene N `pets`. Los turnos pasan a referenciar opcionalmente una mascota.
- **Calendario sanitario preventivo**: registro de eventos sanitarios (antirrábica, séxtuple, antiparasitario, baño sanitario) con próxima fecha calculada por periodicidad configurable por tenant.
- **Recordatorio sanitario que vende**: job diario que detecta vencimientos próximos y dispara una plantilla de WhatsApp con CTA "Agendar turno", que entra al flujo de reserva del bot con servicio y mascota precargados.
- **Registro en el punto de atención**: al marcar un turno `completed`, el panel ofrece registrar qué se aplicó; ese registro recalcula la próxima fecha automáticamente.
- **Métrica de venta del producto**: "facturación recuperada por recordatorios sanitarios" — pesos facturados en turnos con `source = 'health_reminder'`.

NO se construye:
- **Historia clínica** (diagnósticos, tratamientos, medicación, peso, estudios). Razón: los datos de salud animal no tienen el problema legal de los humanos, pero una historia clínica completa es scope creep puro — no aporta nada a la venta del MVP (que se vende por los recordatorios que recuperan facturación) y compite con software veterinario establecido. Solo calendario de eventos preventivos.
- Vademécum / stock de vacunas, facturación AFIP, internaciones/cirugías con seguimiento, multi-sucursal, pagos/señas (si una vet lo pide, se monta el módulo del plan 02 tal cual).
- Carnet de vacunación imprimible/PDF (extensión prevista, §10).

## 2. Delta sobre el motor

| Módulo del motor | Reuso |
|---|---|
| `core-db`, multi-tenant, RLS, auth, panel base | Tal cual |
| `core-whatsapp` (plantillas, webhook, message_log) | Tal cual |
| `core-scheduling` (disponibilidad/slots) | Tal cual, sin cambios |
| `core-bot` (FSM Flujos A y B) | **Se extiende**: paso de selección de mascota + entrada con precarga desde recordatorio sanitario |
| `core-jobs` (recordatorios de turno) | **Se extiende**: job diario de vencimientos sanitarios |
| Agenda/clientes/servicios/equipo del panel | **Se extienden**: mascota en turno y en ficha de cliente, modal post-completado |
| Métricas del panel | **Se extienden**: facturación recuperada |
| Módulo NUEVO | `pets` + `health_event_types` + `health_events` + recordatorio sanitario completo |

`tenants.vertical = 'vet'`. Todo lo nuevo vive detrás de ese flag: un tenant salón no ve nada de esto.

## 3. Modelo de datos (delta)

Convenciones del motor: UUID v7, `tenant_id` NOT NULL + índice, timestamptz, centavos, E.164.

```
pets
  id, tenant_id, customer_id (FK customers, NOT NULL),
  name, species ('dog'|'cat'|'other'),
  breed nullable, birth_date nullable (date, aproximada),
  notes text nullable, active bool default true
  -- índice (tenant_id, customer_id)

health_event_types                 -- catálogo POR TENANT, con seed default
  id, tenant_id, name,                              -- "Antirrábica", "Antiparasitario interno"
  default_period_days int,                          -- 365, 90, ...
  species ('dog'|'cat'|'any') default 'any',
  service_id nullable (FK services),                -- servicio a precargar en el CTA de agendar
  active bool, sort_order

health_events                      -- UN evento aplicado; el vigente por (pet, type) es el de applied_on más reciente
  id, tenant_id, pet_id (FK), event_type_id (FK health_event_types),
  applied_on date, source ('in_clinic'|'external'),
  appointment_id nullable (FK appointments),        -- si se registró al completar un turno
  next_due_on date,                                 -- applied_on + period_days (ver §6)
  reminder_sent_at timestamptz nullable,            -- el job lo marca al enviar
  reminder_dismissed bool default false             -- el dueño tocó "No, gracias"
  -- índice (tenant_id, next_due_on) para la query del job

appointments (cambios)
  + pet_id nullable (FK pets)                       -- nullable: compra de alimento, consulta sin mascota cargada
  + source: se agrega valor 'health_reminder'       -- turno originado por recordatorio sanitario
```

Seed del catálogo al crear un tenant vet (la vet lo ajusta después): Antirrábica 365 / Vacuna séxtuple-quíntuple 365 (dog) / Triple felina 365 (cat) / Antiparasitario interno 90 / Pipeta-antiparasitario externo 30 / Baño sanitario 60.

Integridad: registrar un `health_event` nuevo de un tipo deja obsoleto al anterior (no se borra; la query de vencimientos toma solo el más reciente por `(pet_id, event_type_id)`). Borrar una mascota es soft (`active = false`) y excluye sus eventos del job.

## 4. Flujos de WhatsApp (delta)

### Plantillas nuevas (categoría utility, enviar a aprobación al inicio de la Fase V3)

**`recordatorio_sanitario`** (un solo vencimiento):
> Hola {{1}}! 🐾 Te escribimos de {{2}}. **{{3}}** tiene {{4}} que vence el {{5}}. Es importante mantenerla al día. ¿Le agendamos un turno?

Variables: 1 = nombre del dueño, 2 = nombre de la veterinaria, 3 = nombre de la mascota, 4 = evento ("la vacuna antirrábica"), 5 = fecha ("15/07").
Botones quick-reply: `Agendar turno` / `Ya lo hice en otro lado` / `No, gracias`.

**`recordatorio_sanitario_multi`** (vencimientos agrupados del mes, una o varias mascotas):
> Hola {{1}}! 🐾 Te escribimos de {{2}}. Este mes vencen estos cuidados: {{3}}. ¿Agendamos un turno para ponerlos al día?

Variables: 1 = nombre del dueño, 2 = veterinaria, 3 = lista compacta en una línea ("Firulais: antirrábica (15/07) y pipeta (20/07) · Michi: triple felina (28/07)").
Botones quick-reply: `Agendar turno` / `Ya están al día` / `No, gracias`.

### Cambios a la FSM (`core-bot`)

**Flujo B (reserva guiada) — estado nuevo `awaiting_pet`**, primero del flujo, antes de `awaiting_service`:
```
awaiting_pet:          lista con las mascotas activas del customer + ítem "🐾 Otra mascota"
                       → "¿Para quién es el turno?"
awaiting_pet_name:     (solo si eligió "Otra mascota") "¿Cómo se llama? Escribí el nombre."
awaiting_pet_species:  botones [Perro 🐶 | Gato 🐱 | Otro] → crea el pet y sigue a awaiting_service
```
Si el customer no tiene mascotas cargadas, se saltea la lista y va directo a `awaiting_pet_name`. El `pet_id` queda en `context` y se persiste en el `appointment` al confirmar. El resumen de `confirming` incluye la mascota: "Firulais · Vacuna antirrábica · jue 17/07 10:30 con Dra. Paula".

**Flujo C (nuevo) — respuesta al recordatorio sanitario**:
```
ButtonPressed('book_health')    → entra a Flujo B con pet_id y service_id precargados
                                  (del health_event_type.service_id) → salta directo a awaiting_day.
                                  Multi-mascota o multi-evento: paso previo con lista
                                  "¿Por cuál empezamos?" (un ítem por vencimiento); el resto
                                  queda pendiente y se re-recuerda el ciclo siguiente.
                                  El appointment se crea con source = 'health_reminder'.
ButtonPressed('done_external')  → "¡Buenísimo! ¿Hace cuánto fue, más o menos?" +
                                  botones [Esta semana | Hace un mes | Hace más]
                                  → registra health_event source 'external' (ver §6) →
                                  "Listo, anotado ✅ Te avisamos cuando toque la próxima."
                                  Multi: antes, lista "¿Cuál ya hiciste?" para elegir el evento.
ButtonPressed('dismiss_health') → reminder_dismissed = true en esos eventos →
                                  "Dale, no te molestamos con esto. Cualquier cosa escribinos 🐾"
```

Sin otros cambios al Flujo A ni a las reglas transversales del motor (TTL, handoff, opt-out, carrera de slots aplican igual).

## 5. Panel web (delta)

1. **Clientes → detalle** (extendida): sección "Mascotas" con ABM inline (nombre, especie, raza, fecha de nacimiento aprox., notas) y, por mascota, su calendario sanitario: tabla evento / última aplicación / próximo vencimiento (rojo si vencido, amarillo si vence en ≤14 días) + botón "Registrar evento" (tipo, fecha, in_clinic/external).
2. **Agenda** (extendida): el turno muestra mascota junto al cliente; el alta manual de turno permite elegir mascota del cliente o crearla al vuelo.
3. **Modal post-completado** (nuevo): al marcar un turno `completed`, modal "¿Qué se le aplicó a {pet}?" con checkboxes del catálogo (preseleccionado el tipo cuyo `service_id` coincide con el servicio del turno) + fecha (default hoy) + "Nada, solo consulta". Cada check crea un `health_event` con `appointment_id` y recalcula `next_due_on`.
4. **Vencimientos** (pantalla nueva): lista de próximos vencimientos del tenant (30 días) y vencidos, con filtros por tipo y estado de recordatorio (enviado / agendó / descartó). Acción manual: "Enviar recordatorio ahora" (respeta las reglas anti-spam de §6).
5. **Configuración → Calendario sanitario** (nueva sección): ABM del catálogo `health_event_types` (nombre, periodicidad en días, especie, servicio asociado, activo) + settings de §8.
6. **Métricas** (extendida): además de las del motor — recordatorios sanitarios enviados en el mes, tasa de respuesta, turnos generados, y **"Facturación recuperada por recordatorios"**: suma de `price_cents` del servicio de los turnos `source = 'health_reminder'` con estado `completed` en el mes, número grande arriba de todo. ES la métrica con la que se vende y se retiene este producto: el dueño de la vet tiene que poder decir "el bot me facturó $X este mes".

## 6. Lógica de negocio específica

**Cálculo de próxima fecha**: `next_due_on = applied_on + period_days`, donde `period_days` sale del `health_event_types` del tenant en el momento del registro (cambiar el catálogo después NO recalcula eventos existentes — decisión: el dato histórico es inmutable). El staff puede pisar `next_due_on` manualmente al registrar (caso "el plan vacunal de este cachorro es distinto").

**Evento externo ("ya la vacuné en otro lado")**: el botón `done_external` registra un `health_event` con `source = 'external'` y `applied_on` aproximado según la respuesta: "Esta semana" → hoy − 3 días, "Hace un mes" → hoy − 30, "Hace más" → hoy − 90. La precisión no importa: el objetivo es que el ciclo de recordatorios siga vivo y no spamear con algo ya hecho. El panel distingue eventos externos (sin turno asociado, no suman a facturación recuperada).

**Selección de vencimientos (job diario, por tenant, a `send_hour` hora local)**:
1. Tomar el evento vigente (más reciente) por `(pet_id, event_type_id)` de mascotas activas.
2. Filtrar: `next_due_on` ∈ [hoy − 30, hoy + `lead_days`], `reminder_sent_at` null, `reminder_dismissed` false, customer sin `opted_out`.
3. **Agrupar por customer**: 1 vencimiento → `recordatorio_sanitario`; 2 o más (aunque sean de mascotas distintas) → `recordatorio_sanitario_multi` con hasta `max_items_per_message` ítems, ordenados por fecha. NUNCA más de un mensaje sanitario por envío: quien tiene 4 mascotas recibe UN mensaje con los vencimientos del período, no 4.
4. **Anti-spam**: no enviar si el customer recibió cualquier plantilla sanitaria hace menos de `min_gap_days` (default 30). Los vencimientos que quedaron afuera entran al ciclo siguiente.
5. Marcar `reminder_sent_at` en todos los eventos incluidos.

**Cierre del ciclo**: si el dueño agenda y el turno se completa con registro de aplicación, nace el `health_event` nuevo y el ciclo arranca de cero. Si el turno se cancela o queda no-show, el evento sigue vencido pero `reminder_sent_at` ya está marcado: se vuelve a recordar recién al ciclo siguiente (≥ `min_gap_days`). Re-recordatorio máximo: 3 envíos por evento vigente; después se muestra solo en la pantalla Vencimientos para gestión telefónica humana.

**Opt-out**: `customers.opted_out` del motor manda — opt-out es total, no hay opt-out parcial "solo sanitarios" en el MVP. `reminder_dismissed` ("No, gracias") silencia SOLO ese evento vigente; cuando se registre una nueva aplicación, el evento nuevo nace recordable.

**Mascota fallecida / dada de baja**: `pets.active = false` (acción en panel) corta todos sus recordatorios al instante. Cuidado en los textos: jamás enviar recordatorio de una mascota inactiva — es el peor error posible de este producto.

## 7. Fases de implementación

Prerrequisito: motor en producción. Cada fase termina deployada.

### Fase V1 — Mascotas (3-4 días)
Tabla `pets`, `appointments.pet_id`, ABM de mascotas en detalle de cliente, mascota visible en agenda y alta manual de turno. FSM: estados `awaiting_pet` / `awaiting_pet_name` / `awaiting_pet_species` con tests unitarios.
✅ *Criterios*: reserva por WhatsApp de punta a punta eligiendo mascota existente y creando "Otra mascota"; el turno en el panel muestra la mascota; tests de FSM cubren: cliente sin mascotas, lista con 1 y con N, nombre vacío reintentado.

### Fase V2 — Calendario sanitario (1 semana)
`health_event_types` (con seed) + `health_events`. Configuración del catálogo, calendario sanitario en ficha de mascota, registro manual de eventos, modal post-completado, pantalla Vencimientos (solo lectura todavía).
✅ *Criterios*: marcar un turno `completed` → modal → registrar "Antirrábica" → `next_due_on` correcto a 365 días visible en la ficha; pisar la fecha manualmente funciona; Vencimientos lista vencidos y próximos con semáforo correcto (test de la query con casos borde: dos eventos del mismo tipo, mascota inactiva).

### Fase V3 — Recordatorios que venden (1-1,5 semanas)
Plantillas enviadas a aprobación el día 1. Job diario en Trigger.dev con la selección/agrupación/anti-spam de §6. Flujo C completo: CTA → Flujo B precargado (`source = 'health_reminder'`), evento externo, dismiss. Envío manual desde Vencimientos.
✅ *Criterios*: e2e demostrable — seed con vencimiento a 10 días → corre el job → llega `recordatorio_sanitario` → "Agendar turno" → bot ofrece días directamente (mascota y servicio precargados) → turno creado con `source = 'health_reminder'`; customer con 2 mascotas y 3 vencimientos recibe UN solo mensaje multi; "Ya lo hice en otro lado" registra evento externo y la ficha muestra la fecha recalculada; segundo run del job el mismo mes NO reenvía a quien ya recibió; opt-out no recibe nada.

### Fase V4 — Métrica de venta, pulido y runbook (3-5 días)
Métricas extendidas con "Facturación recuperada", tasa de respuesta de recordatorios, contadores en Vencimientos. Límite de 3 re-recordatorios. Runbook de onboarding de una veterinaria nueva (crear tenant vet, ajustar catálogo y periodicidades, carga inicial de mascotas/últimas vacunas desde la libreta o Excel de la vet, trámite Meta).
✅ *Criterios*: completar un turno `source = 'health_reminder'` suma su precio a "Facturación recuperada" del mes (test); screenshot del dashboard con la métrica arriba de todo; runbook seguido por una persona sin contexto deja un tenant vet demo operativo.

**Total estimado: 3-4 semanas** sobre el motor terminado.

## 8. Configuración del tenant

En `tenants.settings` (jsonb), bajo la clave `vet`:

```ts
const vetSettingsSchema = z.object({
  health_reminder_lead_days: z.number().int().min(3).max(60).default(14),   // antelación del aviso
  health_reminder_send_hour: z.number().int().min(8).max(20).default(10),   // hora local de envío del job
  health_reminder_min_gap_days: z.number().int().min(7).max(90).default(30),// anti-spam por customer
  health_reminder_max_items_per_message: z.number().int().min(2).max(8).default(6),
  health_reminder_max_resends: z.number().int().min(1).max(5).default(3),   // por evento vigente
});
```

Editables en Configuración → Calendario sanitario. El catálogo de tipos/periodicidades NO va en settings: es la tabla `health_event_types`.

## 9. Puntos a verificar al implementar

- Aprobación de Meta de las plantillas sanitarias como **utility**: el texto con CTA de agendar bordea marketing según el criterio vigente de Meta; si las rechazan como utility, re-someter como marketing (cambia el costo por mensaje — verificar tarifa Argentina vigente) y avisar al dueño del producto porque afecta el costo unitario del recordatorio.
- Longitud máxima vigente de variables de plantilla: la variable {{3}} del multi concatena varios ítems — confirmar el límite actual de caracteres por parámetro y truncar la lista con "…y N más" si hace falta.
- Todo lo de §9 del motor (tarifas Cloud API, versión Graph API, límites de interactivos) aplica igual.

## 10. Extensiones previstas (no implementar)

- **Carnet sanitario PDF/link** para el dueño (compartible, con branding de la vet) — sale casi gratis de `health_events`.
- **Planes vacunales multi-dosis** (cachorros: séxtuple ×3 cada 21 días) — hoy se resuelve pisando `next_due_on` a mano; a futuro, `health_event_types` con secuencias.
- **Historia clínica liviana** (peso, observaciones por visita) — solo si los tenants pagos la piden; nunca antes.
- **Recordatorio de cumpleaños de la mascota** con promo (marketing, requiere plantilla marketing y opt-in explícito).
- El diseño no debe bloquear nada de esto: `health_events` ya guarda historial completo y `health_event_types` es extensible por tenant.
