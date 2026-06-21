# TurneraBot — Gestión de Turnos y Recordatorios por WhatsApp

> Plataforma B2B todo-en-uno para salones de belleza y peluquerías de Córdoba: agenda centralizada + bot de WhatsApp que confirma, reprograma y recuerda turnos sin intervención del personal.

**Documento de producto v1 — Junio 2026**

---

## 1. El problema (y por qué vale plata)

Los salones de belleza y peluquerías pierden dinero por dos vías:

1. **No-shows**: en el rubro belleza/wellness la tasa de ausencias sin aviso ronda el **15–25%** cuando no hay recordatorios. Un salón con 30 turnos/día y ticket promedio de $15.000 ARS pierde **$1.3M–2.2M ARS por mes** solo en sillas vacías.
2. **Tiempo operativo**: la dueña o una empleada dedica 1–3 horas diarias a responder WhatsApp ("¿tenés turno el viernes?"), anotar en cuaderno o Google Calendar, y perseguir confirmaciones. Eso es tiempo de un servicio facturado que no se hace.

**Insight clave**: en Argentina el cliente final *vive* en WhatsApp. No va a descargar una app del salón ni a entrar a una web a reservar. Cualquier solución que no pase por WhatsApp fracasa en adopción del cliente final. Esa es la apuesta diferencial del producto.

### Validación pendiente (hacer ANTES de escribir código)
- [ ] Entrevistar 10–15 salones de Córdoba (mix: barrio + centro + cadenas chicas). Preguntas: ¿cuántos no-shows por semana? ¿quién maneja el WhatsApp? ¿qué usan hoy (cuaderno / Excel / AgendaPro)? ¿cuánto pagarían por recuperar esos turnos?
- [ ] Cuantificar: si el no-show real es <8%, el pitch de "recuperá plata" se debilita y hay que pivotar el mensaje a "ahorrá tiempo".
- [ ] Conseguir 3 salones que firmen carta de intención o paguen una seña por el piloto. **Sin esto, no construir.**

---

## 2. Propuesta de valor

| Para quién | Dolor | Qué le damos |
|---|---|---|
| Dueño/a del salón | Pierde plata por no-shows | Recordatorios automáticos con confirmación → tasa de ausencia baja al 3–8% |
| Recepcionista / dueña que atiende | Horas respondiendo WhatsApp | El bot agenda, confirma, reprograma y cancela solo |
| Cliente final | Llamar / esperar respuesta para un turno | Reserva en 30 segundos desde el chat que ya usa |

**One-liner de venta**: *"Recuperá los turnos que hoy perdés. El bot confirma cada cita por WhatsApp y si el cliente cancela, el lugar se libera para otro — sin que toques el teléfono."*

---

## 3. Alcance funcional

### MVP (lo mínimo vendible — 8 a 10 semanas)

1. **Agenda del salón (web app)**
   - Calendario por profesional (vista día/semana).
   - ABM de servicios (nombre, duración, precio) y profesionales (horarios, días).
   - Carga manual de turnos (la recepcionista sigue pudiendo anotar por teléfono).
   - Bloqueos (feriados, vacaciones, almuerzo).

2. **Recordatorios automáticos por WhatsApp** ← *el corazón del MVP*
   - Mensaje plantilla X horas antes (configurable: 24h y/o 2h).
   - Botones nativos: ✅ Confirmo / 🔄 Reprogramar / ❌ Cancelar.
   - Si cancela → el slot se libera y el salón ve la novedad en la agenda.
   - Estado del turno visible en el calendario (pendiente / confirmado / cancelado).

3. **Bot de reservas (flujo guiado, NO IA conversacional libre)**
   - Cliente escribe → menú con botones/listas de WhatsApp: servicio → profesional → día → hora → confirmación.
   - Fallback "hablar con una persona" que notifica al salón.
   - Reconocimiento de cliente recurrente (por número de teléfono).

4. **Panel básico**
   - Lista de turnos del día, métricas simples: turnos agendados, confirmados, no-shows evitados (cancelaciones con reocupación).

### V2 (post-validación, meses 4–8)
- Lista de espera automática: si se libera un slot, ofrece el lugar por WhatsApp a clientes en espera (esto convierte cancelaciones en facturación — feature estrella para retención).
- Señas / pago anticipado vía Mercado Pago (mata el no-show de raíz).
- Historial y ficha de cliente (frecuencia, gasto, preferencias).
- Campañas: "hace 40 días que no venís, ¿agendamos?" (re-engagement con plantillas de marketing).
- Multi-sucursal.
- Reportes de facturación y comisiones por profesional.

### Fuera de alcance (decir que NO explícitamente)
- App móvil nativa para el cliente final (WhatsApp ES la app).
- IA conversacional libre en el MVP (flujo con botones es más barato, predecible y suficiente; la IA se evalúa en V2 para casos fuera de flujo).
- Facturación electrónica / AFIP (integrar después si lo piden, no es el core).
- Otros verticales (barberías ok porque es el mismo flujo; NO médicos/odontólogos en el MVP — tienen requisitos distintos: obras sociales, privacidad).

---

## 4. Arquitectura técnica

### Decisión crítica: WhatsApp Cloud API oficial (Meta), NO libreras no oficiales

| | Cloud API (oficial) | Baileys / whatsapp-web.js (no oficial) |
|---|---|---|
| Riesgo de ban | Nulo si cumplís políticas | **Alto — te banean el número del salón** |
| Botones/listas nativas | ✅ | Limitado |
| Costo | Por mensaje plantilla (~USD 0.03–0.07 en ARG según categoría) | "Gratis" |
| Escalabilidad/SLA | Garantizado por Meta | Se rompe con cada update de WhatsApp |

Banearle el número de WhatsApp a un salón es matar su negocio y tu reputación. **Cloud API sí o sí.** Punto a verificar al implementar: Meta migró a precios por mensaje (no por conversación) en 2025 y los mensajes *utility* dentro de la ventana de servicio de 24h pasaron a ser gratuitos — verificar tarifas vigentes para Argentina en la doc oficial antes de armar el pricing.

### Conceptos clave de la Cloud API que condicionan el diseño
- **Ventana de 24 horas**: si el cliente te escribió en las últimas 24h, podés responder con texto libre gratis. Fuera de ventana, solo **plantillas pre-aprobadas** (pagas).
- **Plantillas**: los recordatorios son plantillas categoría *utility* (más baratas que *marketing*). Hay que diseñarlas y enviarlas a aprobación de Meta (24–48h por plantilla).
- **Calidad del número**: Meta puntúa tu número; muchos bloqueos/reportes degradan el rating y limitan el volumen. Por eso: siempre dar opción de "no recibir más mensajes".
- **Cada salón = su propio número** (o un número del producto por salón). Recomendado: onboarding vía **Embedded Signup** de Meta para que cada salón conecte su número bajo tu WABA como Tech Provider / Solution Partner.

### Stack sugerido (optimizado para 1 dev fullstack que ya conoce JS)

```
┌─────────────┐     webhook      ┌──────────────────────┐
│  WhatsApp    │ ───────────────▶ │  API (NestJS/Next.js  │
│  Cloud API   │ ◀─────────────── │  API routes)          │
└─────────────┘   send message   └──────┬───────────────┘
                                        │
                      ┌─────────────────┼──────────────────┐
                      ▼                 ▼                  ▼
               ┌────────────┐   ┌─────────────┐   ┌──────────────┐
               │ PostgreSQL │   │ Cola de jobs │   │ Panel web     │
               │ (turnos,   │   │ (recordator. │   │ Next.js +     │
               │  clientes) │   │  programados)│   │ shadcn/ui     │
               └────────────┘   └─────────────┘   └──────────────┘
```

- **Backend + panel**: Next.js (App Router) o NestJS + Next.js. Un solo lenguaje, deploy simple.
- **DB**: PostgreSQL (Neon/Supabase). Modelo multi-tenant desde el día 1 (`tenant_id` en todas las tablas).
- **Jobs programados**: los recordatorios son el caso de uso exacto de una cola con delay — Trigger.dev, Inngest, o BullMQ + Redis (Upstash). *No* usar cron + polling de la DB: se rompe con reprogramaciones.
- **Máquina de estados del bot**: el flujo de reserva es una FSM (esperando_servicio → esperando_profesional → esperando_fecha → ...). Guardar el estado de conversación en DB con TTL. Librería tipo XState o una FSM casera simple.
- **Hosting**: Vercel/Railway/Fly. El webhook de Meta exige HTTPS público y responder en <10s (responder 200 rápido y procesar async).

### Modelo de datos mínimo

`tenants` (salones) → `staff` (profesionales, horarios) → `services` (duración, precio) → `appointments` (cliente, servicio, staff, estado: pending/confirmed/cancelled/no_show/completed) → `customers` (teléfono como ID natural) → `conversations` (estado FSM) → `message_log` (auditoría de todo lo enviado/recibido — imprescindible para debug y disputas).

---

## 5. Modelo de negocio

### Pricing sugerido (validar en entrevistas)

| Plan | Precio ARS/mes (ref.) | Incluye |
|---|---|---|
| **Básico** | ~$25.000–35.000 | 1 profesional, recordatorios, agenda |
| **Pro** | ~$45.000–60.000 | Hasta 5 profesionales, bot de reservas completo, métricas |
| **Multi** | ~$80.000+ | Sucursales, lista de espera, campañas |

Reglas:
- **Precio en ARS ajustable** (o anclado a un índice) — con la inflación argentina, contrato mensual sin permanencia y precio revisable trimestralmente.
- **El costo de mensajería va aparte o con cupo incluido** (ej. 500 plantillas/mes incluidas, excedente a costo + margen). Nunca absorber mensajería ilimitada: un salón grande te puede comer el margen.
- **Anclar el precio al dolor**: si el salón pierde $1.5M/mes en no-shows, $50.000/mes es el 3% del problema. Ese es el pitch, no "software de agenda".
- Piloto: 1 mes gratis o 50% off a los primeros 5 salones a cambio de feedback semanal y testimonio.

### Unit economics a vigilar
- Costo de mensajería por salón activo (estimar: 30 turnos/día × 2 plantillas = ~1.300 plantillas/mes; a ~USD 0.034 c/u ≈ USD 45/mes — **verificar tarifa vigente**, esto define el piso del pricing).
- Infra: despreciable hasta ~50 salones (< USD 100/mes).
- CAC: venta presencial puerta a puerta al inicio (caro en tiempo, barato en plata).

---

## 6. Competencia y diferenciación

| Competidor | Qué es | Debilidad explotable |
|---|---|---|
| **AgendaPro** | Líder LATAM en gestión de salones | WhatsApp es add-on, no el core; precio en USD; onboarding pesado |
| **Reservo / Booksy / Fresha** | Agenda + marketplace | El cliente reserva por app/web, no por chat; soporte lejano |
| **Cuaderno + WhatsApp manual** | El competidor real (80% del mercado) | Es gratis pero come horas y no recuerda nada |
| **Bots genéricos (ManyChat, etc.)** | Constructores de chatbot | No tienen agenda, hay que armarlo todo; en inglés |

**Diferenciación**: (1) WhatsApp-first de verdad — todo el ciclo de vida del turno vive en el chat; (2) soporte local en Córdoba, en castellano, por WhatsApp obviamente; (3) setup en 48h con plantillas pre-aprobadas, no un constructor DIY.

**Riesgo competitivo honesto**: nada de esto es defendible técnicamente — AgendaPro puede copiarlo. La defensa es velocidad, nicho geográfico, relación con clientes y profundidad vertical (lista de espera, señas, campañas específicas del rubro).

---

## 7. Go-to-market (Córdoba primero)

1. **Mes 0–1 — Validación**: entrevistas + cartas de intención (sección 1).
2. **Mes 1–3 — Construcción del MVP** con los 3–5 salones piloto involucrados desde el día 1 (demo quincenal).
3. **Mes 3–4 — Piloto en producción**: medir tasa de no-show antes/después. **Este número es tu material de venta.**
4. **Mes 4–6 — Venta directa**: 
   - Caso de éxito documentado ("Salón X redujo ausencias del 20% al 5% = recuperó $X/mes").
   - Puerta a puerta en zonas de densidad de salones (Nueva Córdoba, Cerro, General Paz).
   - Referidos: 1 mes gratis por salón referido que se suscriba.
   - Instagram con contenido para dueñas de salón (el rubro vive en IG).
5. **Mes 6+**: si funciona en Córdoba → Rosario, Mendoza, conurbano. Mismo playbook.

**Meta realista año 1**: 25–40 salones pagos (~USD 1.000–2.000 MRR). Suficiente para validar y decidir si se escala con inversión o se mantiene como negocio rentable chico.

---

## 8. Riesgos principales

| Riesgo | Probabilidad | Mitigación |
|---|---|---|
| Requisitos de Meta (verificación de negocio, aprobación de plantillas) demoran el onboarding | Alta | Documentar el proceso, hacerlo VOS por el salón como parte del setup; usar Embedded Signup |
| Cambios de pricing/políticas de WhatsApp | Media | Margen de buffer en pricing; cláusula de ajuste; monitorear changelog de Meta |
| El salón no carga la agenda / la abandona | Alta | Onboarding asistido, importar su cuaderno/Excel, UX móvil-first del panel (la dueña administra desde el celu) |
| Clientes finales marcan los mensajes como spam → degrada el número | Media | Solo plantillas utility, opt-out claro, frecuencia limitada |
| Cobranza en Argentina (churn por crisis) | Media | Débito automático Mercado Pago, plan mensual barato vs. anual con descuento |
| Un solo dev = bus factor y soporte 24/7 | Alta | Flujos con botones (no IA) minimizan errores; horario de soporte definido; logging exhaustivo desde el día 1 |

---

## 9. Métricas que definen éxito

**Del producto (por salón):**
- Tasa de no-show antes vs. después (objetivo: <8%).
- % de turnos confirmados por bot sin intervención humana (objetivo: >70%).
- % de reservas que entran por el bot vs. carga manual.

**Del negocio:**
- Salones activos pagos y churn mensual (objetivo: <5%).
- MRR y costo de mensajería como % del MRR (objetivo: <20%).
- Tiempo de onboarding por salón (objetivo: <48h).

---

## 10. Próximos pasos inmediatos

1. **Esta semana**: armar guion de entrevista + lista de 20 salones objetivo en Córdoba. Agendar las primeras 5.
2. Crear cuenta Meta Business + app de desarrollo con WhatsApp Cloud API (sandbox es gratis) y prototipar el flujo de recordatorio con botones — 2 o 3 días de spike técnico para validar tiempos de aprobación de plantillas.
3. Definir nombre y registrar dominio (este doc usa "TurneraBot" como placeholder).
4. Con 3 cartas de intención firmadas → kickoff del MVP (sección 3).

---

*Regla de oro de este proyecto: el MVP no se empieza a construir hasta tener 3 salones comprometidos. El riesgo no es técnico — es de adopción y cobranza.*
