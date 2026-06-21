# Plan 05 — La Máquina de Ventas (automatizar lo automatizable)

> Cómo vender el portfolio minimizando la parte que más cuesta (la venta en frío), automatizando todo lo automatizable y dejando claro qué NO se puede automatizar y por qué. Incluye la spec del **Prospector** (la herramienta interna de leads + CRM) y del **Demo Bot** (el vendedor que trabaja 24/7).

---

## 1. La verdad incómoda primero: qué NO se puede automatizar (y por qué)

### ❌ WhatsApp masivo en frío a una base comprada/scrapeada

La fantasía: "armo una base de 5.000 comercios y les mando a todos una plantilla ofreciendo el producto". **No lo hagas.** Razones concretas, no morales:

1. **Meta lo penaliza rápido**: mensajes a gente que nunca te escribió = bloqueos y reportes → tu número pierde calidad → Meta te limita el volumen → te banea. Y un número de empresa baneado es muy difícil de recuperar.
2. **Es TU materia prima**: todo tu portfolio corre sobre WhatsApp Cloud API. Tu negocio depende de que Meta te considere un actor limpio (más aún cuando seas Tech Provider con números de clientes bajo tu cuenta). Spamear es quemar el activo central de la empresa.
3. **La ironía comercial**: vendés sistemas que cuidan la reputación del WhatsApp del cliente. Si tu primer contacto es spam, el pitch muere ahí.
4. En Argentina además rige la Ley 25.326 de protección de datos personales — para B2B con datos públicos de comercios el riesgo legal es bajo, pero el riesgo de plataforma (puntos 1-3) es total.

**Regla**: por la API oficial, SOLO se le escribe a quien dio su número en un contexto de consentimiento (llenó un formulario, escribió primero, te lo dio en persona). El contacto FRÍO va por otros canales (§3).

### ❌ "Cerrar la venta" sin humano (al principio)

Los primeros 5-10 clientes de cada vertical los tenés que cerrar vos. No porque la IA o un sistema no puedan conversar — sino porque en esas conversaciones está la información que ningún sistema te da: qué objeción aparece siempre, qué precio aguanta, qué feature les brilla los ojos. Sin eso, ni el mejor vendedor contratado sabe qué decir. La buena noticia: no hace falta "saber vender" — hace falta un guion (§5) y una demo que vende sola (§4). Después de esos primeros clientes, delegás en vendedores a comisión con el kit armado.

---

## 2. El Prospector — base de datos de leads + mini-CRM (herramienta interna, construible)

Esto es exactamente lo que pediste y SÍ es automatizable. Una herramienta interna (otro tenant de tu propio motor, irónicamente) con:

### 2.1 Recolección de la base

| Fuente | Método | Qué da |
|---|---|---|
| **Google Places API** | Búsqueda por categoría + zona ("peluquería, Nueva Córdoba", "veterinaria, Cerro") | Nombre, dirección, teléfono, web, rating, cantidad de reseñas, horarios |
| **Instagram** | Búsqueda manual/asistida por hashtags y ubicación (#peluqueriacordoba) | Handle, seguidores, actividad, si responde DMs |
| **Caminar la zona** (sí, en serio) | Foto de la vidriera + carga por voz en el celu | Los datos más valiosos: ¿tiene cartel de "turnos por WhatsApp"? ¿cuánta gente había? |
| Guías locales / páginas amarillas / muni | Import CSV | Relleno |

Notas: la Places API es la vía legítima (el scraping directo de Maps viola los términos de Google; la API tiene reglas sobre almacenamiento — para uso interno de prospección el riesgo práctico es bajo, pero sabelo). Instagram no permite scraping automatizado — la recolección ahí es manual asistida (15-20 por sesión mientras mirás la tele) o con una persona contratada por hora para cargar datos.

### 2.2 Modelo de datos del Prospector

```
prospects
  id, business_name, vertical_fit ('salon'|'cancha'|'vet'|'gym'|'otro'),
  zone, address, phone (E.164, puede ser fijo), whatsapp_confirmed bool,
  instagram, followers, website, google_rating, google_reviews_count,
  score int,                  -- §2.3
  status ('nuevo'|'contactado'|'respondio'|'demo_agendada'|'demo_hecha'|
          'negociando'|'cliente'|'perdido'|'no_molestar'),
  owner_name nullable, notes, source, next_followup_on date nullable

prospect_touches              -- cada contacto, por cualquier canal
  id, prospect_id, channel ('visita'|'ig_dm'|'wsp_manual'|'email'|'llamada'|'referido'),
  direction, summary, at
```

### 2.3 Score automático (a quién ir primero)

Señales de que un comercio va a comprar: tiene Instagram activo (ya invierte en lo digital) + >50 reseñas en Google (volumen de clientes real) + NO tiene sistema de reservas online detectable (no aparece AgendaPro/Booksy en su perfil) + responde rápido los DMs. El Prospector calcula un score 0-100 con esos campos y te ordena la lista del día. **Esto convierte "una base gigante" en "los 10 mejores de hoy"** — que es lo único accionable.

### 2.4 Seguimiento (acá SÍ hay automatización completa)

- Todo contacto queda registrado; `next_followup_on` dispara la lista "para hoy".
- **Regla de oro que el sistema fuerza**: ningún prospecto `respondio`/`negociando` sin próximo paso fechado. La plata B2B está en el seguimiento — el 80% de las ventas cae porque nadie volvió a escribir a los 4 días.
- Cuando un prospecto YA te escribió por WhatsApp, ahí sí: ventana de 24h, respuestas semiautomáticas, secuencia de seguimiento con plantillas utility ("te paso el resumen de la demo") — todo legítimo porque él inició.
- Dashboard simple: prospectos por estado, tasa de conversión por canal y por vertical (te dice dónde poner la energía), ventas por vendedor (para cuando los tengas).

**Esfuerzo de construcción**: 2-3 semanas con tu stack. Es además tu dogfooding: lo usás vos antes que ningún cliente.

---

## 3. Los canales de contacto frío que SÍ funcionan (rankeados para este mercado)

1. **Visita en persona** 🥇 — para comercios de barrio en Córdoba no hay canal que le gane. 10 visitas bien elegidas (score alto, zona densa) > 500 mensajes. El Prospector arma la ruta del día por zona. Conversión esperable: 1 demo cada 3-4 visitas.
2. **Referidos** 🥈 — automatizable de verdad: al mes de uso, el sistema le pregunta al cliente contento (NPS por WhatsApp, legítimo porque es tu cliente) y al promotor le ofrece "1 mes gratis por cada colega que sume". El rubro belleza/fitness se conoce entre sí.
3. **Instagram DM** 🥉 — el canal frío digital correcto para este mercado (el rubro VIVE ahí). Manual pero asistido: el Prospector te da los 10 del día con un mensaje sugerido personalizado (corto, sobre SU negocio, sin parecer template). Nada de bots de DM masivo — Instagram también banea, mismo problema.
4. **WhatsApp manual 1-a-1 desde tu número personal/comercial** — culturalmente aceptado en Argentina para B2B SI es personalizado, de a poquito (10-15/día máximo), y con nombre y apellido ("Hola Marcela, soy Xavier, vi la peluquería en Bv. San Juan..."). Esto es distinto del blast por API: es una persona escribiendo a otra. Tolerable; si alguien se molesta → `no_molestar` y listo.
5. **Contenido en Instagram** — 2-3 reels/semana mostrando el bot en acción ("mirá cómo este salón confirma 40 turnos solo"). No vende directo: hace que cuando llegue tu DM, ya te hayan visto. Generable con IA en gran parte.
6. **Email frío** — casi inútil en este segmento (no leen el mail). Solo para verticales profesionales (estudios contables sí).

---

## 4. El Demo Bot — la automatización de ventas más rentable de todas

**El producto se demuestra a sí mismo.** Un tenant demo por vertical con un número de WhatsApp publicado en todos lados (tarjeta, IG, landing):

> "Probalo vos mismo: escribile al 351-XXX-XXXX y reservá un turno como si fueras tu clienta."

El prospecto reserva en 40 segundos, recibe el recordatorio de muestra, toca «Confirmar»... y lo ENTENDIÓ sin que vos digas una palabra. Después el bot mismo remata: *"¿Te imaginás esto en tu negocio? Dejanos tu nombre y te contactamos hoy"* → alta automática en el Prospector como `respondio` (¡con consentimiento! ahora SÍ podés seguirlo por WhatsApp legítimamente).

Complemento: **landing por vertical** (una page, calculadora de "cuánta plata perdés por no-shows por mes" + video de 90 segundos + el número del demo). Costo: un fin de semana por vertical. Esta pieza convierte las visitas, los DMs y los reels en demos sin tu intervención.

**Este es el punto clave para tu miedo a vender**: tu trabajo deja de ser "convencer" y pasa a ser "lograr que prueben el demo". Eso es mucho más fácil — y delegable.

---

## 5. El Kit de Ventas (lo que hace delegable la venta a vendedores)

Para designar vendedores a comisión sin que dependan de tu cabeza, cada vertical necesita su kit (1-2 días de armado cada uno, la mitad generable con IA):

1. **Guion de 90 segundos**: dolor → demo en el celular del prospecto → precio anclado al dolor ("perdés ~$1,5M/mes en sillas vacías; esto sale $50k"). Una página.
2. **Tabla de objeciones** (las 8 de siempre: "mis clientas son grandes y no usan eso" → "usan WhatsApp todas", "ya tengo cuaderno" → costo de los no-shows, "es caro" → ancla al dolor, etc.) con la respuesta probada en TUS primeras ventas — por eso las primeras las hacés vos.
3. **Caso de éxito con números reales** del piloto (la pieza que más vende; sin esto el kit no existe).
4. **Demo bot + landing** (§4) para que el vendedor solo tenga que mostrar y anotar.
5. **Modelo de comisión sugerido**: 50-100% del primer mes + 10% recurrente los primeros 12 meses (alinea al vendedor con la retención, no solo con el cierre). El Prospector registra qué vendedor tocó qué prospecto — las comisiones salen solas del sistema.

---

## 6. Resumen: qué se automatiza y qué no

| Etapa | ¿Automatizable? | Cómo |
|---|---|---|
| Armar la base de prospectos | ✅ 80% | Places API + carga asistida (Prospector §2) |
| Elegir a quién contactar | ✅ 100% | Score + lista del día |
| Primer contacto frío | ⚠️ Solo asistido | IG DM / visita / WhatsApp personal, personalizado y de a poco. NUNCA blast por API |
| Demo | ✅ 95% | Demo Bot + landing — el prospecto se demuestra el producto solo |
| Seguimiento post-respuesta | ✅ 90% | Ventana 24h + plantillas legítimas + followups del Prospector |
| Cierre y objeciones | ❌ humano | Vos los primeros 5-10 por vertical; después vendedores con el kit |
| Referidos | ✅ 80% | NPS automático + incentivo, dentro de tu propia plataforma |
| Onboarding del cliente nuevo | ✅ 70% | Runbooks que ya están en cada plan (Fase final) |

## 7. Orden de ejecución

1. **Con el motor en piloto (mes 3)**: construir el Prospector (2-3 semanas) y cargar 200 prospectos de salones con score.
2. **Antes de salir a vender**: Demo Bot + landing del vertical 1 (un fin de semana).
3. **Meses 4-6**: vos vendés con el sistema asistiéndote (lista del día, ruta, followups). Objetivo: 10 clientes + caso de éxito + tabla de objeciones real.
4. **Mes 6+**: armar el kit, sumar el primer vendedor a comisión, vos pasás a supervisar el embudo desde el dashboard.
5. **Cada vertical nuevo**: repetir 2-4 con el kit del vertical (cada vez más barato: el Prospector y el patrón ya existen).
