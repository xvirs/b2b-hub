# Portfolio de Productos B2B — Índice Maestro

> Carpeta de planes de desarrollo listos para ejecutar con cualquier IA (Claude Opus, etc.). Cada plan está escrito para que un modelo SIN contexto previo pueda implementarlo de punta a punta: decisiones tomadas, stack fijado, modelo de datos definido, fases con criterios de aceptación.

**Última actualización: 12 de junio de 2026**

---

## Cómo usar estos planes con una IA

Cada documento es autocontenido. Para arrancar un desarrollo, el prompt es simplemente:

```
Lee /Users/xavier/Proyectos/Planes-B2B/01-NUCLEO-MOTOR.md completo, y de la carpeta
Planes-B2B/tecnico/ los documentos T1 (schema de DB), T2 (interfaces de paquetes)
y T3 (API y jobs) — son las definiciones técnicas exactas a respetar.
Implementá la Fase 1 exactamente como está especificada.
No cambies el stack, el schema ni los contratos sin consultarme; si algo resulta
insuficiente, proponé el cambio y actualizá el documento técnico en el mismo PR.
Al terminar cada fase, verificá los criterios de aceptación listados.
```

Reglas para mantener la calidad cuando trabajes con cualquier modelo:

1. **Una fase por sesión.** No pidas "hacé todo el sistema". Cada spec está dividida en fases de 1-2 semanas justamente para eso.
2. **Los criterios de aceptación son el contrato.** Antes de dar por cerrada una fase, pedile a la IA que demuestre cada criterio (test, screenshot, curl).
3. **El stack no se negocia.** Si la IA sugiere otra librería "mejor", la respuesta es no — la consistencia entre productos vale más que cualquier mejora marginal, porque el motor se comparte.
4. **Actualizá la spec cuando decidas algo nuevo.** La spec es la fuente de verdad, no el chat. Si en una sesión se tomó una decisión, agregala al documento antes de cerrar.
5. **Cada repo lleva un CLAUDE.md** que apunta a su spec y resume convenciones (la Fase 0 de cada plan lo genera).

---

## Documentos

| Doc | Qué es | Estado |
|---|---|---|
| [00-INDICE.md](00-INDICE.md) | Este archivo | — |
| [01-NUCLEO-MOTOR.md](01-NUCLEO-MOTOR.md) | **El motor común + vertical salones.** El desarrollo más importante: multi-tenant, WhatsApp, agenda, recordatorios, panel. Todos los demás se montan sobre éste. | Spec completa |
| [02-VERTICAL-CANCHAS.md](02-VERTICAL-CANCHAS.md) | Reservas de canchas (pádel/F5) con señas MP | Spec completa |
| [03-VERTICAL-VETERINARIAS.md](03-VERTICAL-VETERINARIAS.md) | Turnos + calendario sanitario de mascotas | Spec completa |
| [04-VERTICAL-GIMNASIOS.md](04-VERTICAL-GIMNASIOS.md) | Cuotas, morosidad y cupos por clase | Spec completa |
| [05-MAQUINA-VENTAS.md](05-MAQUINA-VENTAS.md) | Cómo vender el portfolio: qué se automatiza (Prospector, Demo Bot, kit de ventas) y qué no (blast frío por WhatsApp = prohibido) | Plan de ventas |
| [06-PROSPECTOR.md](06-PROSPECTOR.md) | **Spec técnica completa de la máquina de ventas**: Prospector (leads+score+CRM+comisiones), Demo Bot (extensión del motor) y landings | Spec completa |
| [PLANTILLA-SPEC.md](PLANTILLA-SPEC.md) | Plantilla para especificar futuros productos con este estándar | — |
| [IDEAS-AMPLIADAS.md](IDEAS-AMPLIADAS.md) | Banco de ideas nuevas (más allá de turnos) para futuros planes | — |
| [tecnico/T1-DB-SCHEMA.md](tecnico/T1-DB-SCHEMA.md) | Schema completo de DB en código Drizzle (motor + 4 verticales), constraints SQL, RLS | Definición técnica |
| [tecnico/T2-INTERFACES-PAQUETES.md](tecnico/T2-INTERFACES-PAQUETES.md) | Contratos TypeScript de los paquetes core-* (FSM, scheduling, payments, memberships) y algoritmo del orquestador | Definición técnica |
| [tecnico/T3-API-JOBS.md](tecnico/T3-API-JOBS.md) | Webhooks (Meta, MP), Server Actions del panel y catálogo de jobs Trigger.dev | Definición técnica |

**Orden de autoridad ante conflictos entre documentos**: T1 (datos) > T2 (contratos) > T3 (transporte) > planes 0X (intención de producto). Los planes dicen QUÉ y POR QUÉ; los T dicen exactamente CÓMO. Al implementar un plan, leer también los tres T.

Documentos relacionados fuera de esta carpeta:
- [Gestion Wssp/PRODUCTO.md](../Gestion%20Wssp/PRODUCTO.md) — análisis de negocio del turnero de salones (mercado, pricing, go-to-market). El plan técnico vive en 01-NUCLEO-MOTOR.md.
- [IDEAS-B2B-LOCALES.md](../IDEAS-B2B-LOCALES.md) — primera lista de 12 ideas rankeadas con estrategia de motor común.

---

## Orden de construcción recomendado

```
1. NUCLEO + Salones (01)   ← 3 meses. Acá se paga el costo del motor.
2. Canchas (02)            ← +4-6 semanas. Agrega módulo de señas MP.
3. Veterinarias (03)       ← +3-4 semanas. Agrega recordatorios recurrentes.
4. Gimnasios (04)          ← +6 semanas. Agrega módulo de cuotas/suscripciones.
```

Después del paso 4 el motor tiene los 3 módulos grandes (turnos, señas, cuotas) y cualquier vertical nuevo de IDEAS-AMPLIADAS sale en 2-4 semanas.

**Regla comercial** (de [IDEAS-B2B-LOCALES.md](../IDEAS-B2B-LOCALES.md)): los productos se construyen en este orden técnico, pero un vertical sin clientes es inventario. Cuando haya vendedores designados, cada vertical necesita su caso de éxito medible antes de escalar la venta (ej: "salón X bajó no-shows del 20% al 5%").

---

## Convenciones técnicas globales (aplican a TODOS los productos)

Fijadas una vez acá para no repetirlas y para que ningún modelo las "mejore" por su cuenta:

- **Monorepo por producto, motor como paquetes compartidos** (`packages/core-*`). Ver detalle en 01.
- **Stack**: Next.js (App Router) + TypeScript estricto + PostgreSQL + Drizzle ORM + shadcn/ui + Tailwind. Jobs: Trigger.dev. WhatsApp: Cloud API oficial de Meta, nunca librerías no oficiales.
- **Multi-tenant por `tenant_id`** en toda tabla de datos de negocio, con Row Level Security como segunda barrera.
- **Idioma**: UI y mensajes en español rioplatense (voseo: "elegí", "confirmá"). Código e identificadores en inglés.
- **Moneda**: ARS, precios siempre en centavos (integer), nunca float.
- **Fechas**: timestamps en UTC en DB; zona del tenant (default `America/Argentina/Cordoba`) solo al mostrar y al interpretar input del usuario.
- **Teléfonos**: formato E.164 (`+549351...`) como identificador natural del cliente final.
- **Todo mensaje de WhatsApp enviado/recibido se loguea** en `message_log` (auditoría, debug, disputas).
