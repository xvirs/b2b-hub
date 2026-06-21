# Banco de Ideas Ampliado — Productos B2B Vendibles

> Segunda generación de ideas, más allá de la lista inicial de [IDEAS-B2B-LOCALES.md](../IDEAS-B2B-LOCALES.md). Organizadas por el módulo del motor que explotan — porque la pregunta correcta no es "¿qué producto hago?" sino "¿qué producto sale casi gratis con los módulos que ya voy a tener?".
>
> Cuando una idea se decida construir: copiarse [PLANTILLA-SPEC.md](PLANTILLA-SPEC.md) y escribir su plan ANTES de codear.

**Módulos del motor tras completar los planes 01-04**: multi-tenant + bot WhatsApp (FSM) + agenda/turnos + recordatorios programados + señas/pagos MP + cuotas recurrentes + listas de espera.

---

## Grupo A — Explotan el módulo de TURNOS (costo marginal: bajísimo, 1-3 semanas c/u)

### A1. Escuelas de manejo ⭐⭐⭐⭐
- **Dolor**: paquetes de 10 clases vendidos en papel, alumnos que faltan (la clase perdida no se recupera: el instructor y el auto estaban asignados), coordinación telefónica infinita.
- **Producto**: turnero + **paquetes de N sesiones** (comprás 10, el sistema descuenta y avisa "te quedan 3") + recordatorios + reprogramación por bot.
- **Por qué es rentable**: ticket medio-alto, pocas en cada ciudad pero todas con el mismo dolor, cero software local que las atienda. El módulo "paquete de sesiones" después sirve para estética, kinesiología, fotografía.

### A2. Centros de estética / depilación definitiva ⭐⭐⭐⭐
- Ya estaba en la lista anterior; sube de prioridad porque con el motor terminado es trivial: turnos + **protocolos de N sesiones espaciadas X semanas** ("te toca la sesión 4 de 8, agendamos?"). El recordatorio de protocolo GENERA la venta de la siguiente sesión. Mismo comprador que salones → el vendedor ofrece ambos en la misma visita.

### A3. Gestores del automotor / trámites (VTV, transferencias) ⭐⭐⭐
- **Dolor**: el cliente llega sin la documentación completa → turno perdido para los dos.
- **Producto**: turnos + **checklist de documentación por tipo de trámite enviado por WhatsApp al agendar** + recordatorio con la lista de nuevo 24h antes + estado del trámite ("tu transferencia está en el registro, te avisamos").
- El patrón "turno + requisitos previos" después sirve para laboratorios (ayuno), consultorios, etc.

### A4. Peluquerías caninas ⭐⭐⭐
- Skin directo del vertical salones + entidad mascota del vertical veterinarias (plan 03). Costo marginal casi nulo; mismo vendedor que veterinarias.

---

## Grupo B — Explotan el módulo de COBRANZAS/CUOTAS (el más rentable por cliente)

El módulo de cuotas del plan 04 (aviso de vencimiento + link MP + conciliación efectivo) es una **máquina de cobrar aplicable a cualquier negocio de ingresos recurrentes**. Estos verticales pagan bien porque les tocás la caja directamente:

### B1. Estudios contables (recordatorios impositivos a SUS clientes) ⭐⭐⭐⭐⭐
- **Dolor**: el contador persigue a 80 monotributistas por WhatsApp todos los meses: "mandame las facturas", "vence el monotributo", "pagá IIBB". Horas de secretaria.
- **Producto**: el estudio carga sus clientes y vencimientos → el sistema avisa automáticamente a cada cliente (vencimiento de monotributo, fecha de entrega de comprobantes) y **recibe los comprobantes por WhatsApp** (foto/PDF → se archiva por cliente y mes, el contador los descarga ordenados).
- **Por qué es de los mejores**: el comprador es un profesional con plata y mentalidad de eficiencia; el producto le ahorra un sueldo parcial; hay miles de estudios; churn bajísimo (una vez cargada la cartera, no se va). Ticket: $60-120k/mes.
- **Ojo**: acá el tenant tiene MUCHOS contactos → cuidar el rating del número y el costo de plantillas (va dentro del pricing).

### B2. Administración de consorcios chicos / cocheras mensuales ⭐⭐⭐⭐
- Expensas o abonos mensuales + aviso con link de pago + registro de pago en efectivo + reporte de morosidad. Los administradores de consorcios chicos lo manejan en Excel. Cocheras: idéntico flujo, venta más simple.

### B3. Transporte escolar ⭐⭐⭐
- Cobranza mensual a los padres + **aviso "la trafic está a 10 min"** (el chofer toca un botón) + aviso de ausencia del nene por parte del padre. Mercado chico pero fiel, cero competencia, se vende por recomendación entre choferes.

### B4. Clubes de barrio / academias deportivas infantiles ⭐⭐⭐
- Cuotas sociales + comunicados ("se suspende por lluvia") + ficha del socio. Variante directa del plan 04 con menos features. Venta emocional fácil ("ordenamos el club").

---

## Grupo C — Explotan CATÁLOGO + PEDIDOS (sinergia con menudigital-platform)

### C1. Pedidos mayoristas para distribuidoras ⭐⭐⭐⭐⭐
- **Dolor**: la distribuidora (bebidas, golosinas, limpieza) recibe pedidos de 200 kioscos/almacenes por WhatsApp a mano, los pasa a papel, errores constantes.
- **Producto**: catálogo con precios por lista (minorista/mayorista) + el kiosquero pide por WhatsApp con carrito + **re-pedido en un toque** ("repetir pedido de la semana pasada") + hoja de ruta de reparto del día + cuenta corriente simple.
- **Por qué es de los mejores**: ticket alto ($100-250k/mes — la distribuidora factura mucho y el sistema le ordena la operación entera), y cada distribuidora trae cientos de comercios usando tu bot (efecto red para venderles otra cosa después). Es B2B2B: pocos clientes, mucho valor c/u.
- Reusa el catálogo/carrito de menudigital + Carrito-Base.

### C2. Rotiserías / viandas / panaderías (pedidos minoristas) ⭐⭐⭐⭐
- Ya estaba en la lista anterior (combo con menudigital). Se mantiene; construirlo después de C1 porque C1 paga más y compite menos.

### C3. Florerías / regalerías con envío ⭐⭐⭐
- Pedido + fecha/franja de entrega + seña + aviso "tu pedido fue entregado" con foto. Picos en fechas (San Valentín, Día de la Madre) donde el caos manual es total. Mercado chico; hacerlo solo como skin barato de C2.

---

## Grupo D — Productos HORIZONTALES nuevos (un módulo nuevo, todos los rubros como mercado)

### D1. Reputación: reseñas de Google automatizadas ⭐⭐⭐⭐⭐
- **Dolor**: todo negocio local vive de su puntaje en Google Maps y no sabe cómo juntar reseñas; las malas lo agarran por sorpresa.
- **Producto**: post-servicio (se engancha al `completed` de cualquier vertical, o standalone con carga manual/CSV), mensaje por WhatsApp: "¿Cómo fue tu experiencia? ⭐1-5". Si responde 4-5 → "¡Gracias! ¿Nos dejás la reseña? [link directo a Google]". Si 1-3 → "Contanos qué pasó" y va PRIVADO al dueño (la queja no llega a Google, el dueño la resuelve).
- **Por qué es de los mejores**: se vende a CUALQUIER comercio (también a los que ya son clientes de otro vertical = expansión de revenue sin CAC), el valor es visible en 30 días (el puntaje sube), y es barato de construir (2-3 semanas sobre el motor). Ticket: $15-30k/mes — bajo, pero de venta masiva y churn mínimo.
- **Este es el mejor "segundo producto" para un equipo de vendedores**: simple de explicar, demo de 2 minutos, sin onboarding pesado.

### D2. Fidelización: tarjeta de sellos digital ⭐⭐⭐
- "10° corte gratis" pero por WhatsApp: el comercio suma el sello (PIN o QR), el cliente consulta su tarjeta por chat, aviso automático "te falta 1 para el premio". Simple, vendible en combo con D1. Solo, es débil; como add-on, redondea el paquete "marketing local".

### D3. Encuestas/NPS para cadenas chicas ⭐⭐
- Variante de D1 para franquicias locales (3-10 sucursales) con dashboard comparativo entre sucursales. Hacerlo solo si aparece el cliente que lo pide.

---

## Grupo E — Ideas que descarto (y por qué — para no reevaluarlas cada vez)

| Idea | Por qué NO |
|---|---|
| Control de stock para kioscos | El kiosquero no carga stock ni a punta de pistola; adopción imposible |
| Software para consultorios médicos con obra social | Facturación a obras sociales = pozo de complejidad regulatoria; existe competencia instalada |
| Marketplace de turnos (tipo Booksy) | Modelo de dos lados: hay que conquistar comercios Y consumidores a la vez; es otra liga de capital |
| App de delivery propia | Competir con PedidosYa/Rappi en logística es suicidio; lo nuestro es ayudar al comercio a NO depender de ellos |
| CRM genérico | "Genérico" = no le duele a nadie en particular = no lo compra nadie; los verticales existen por algo |

---

## Ranking consolidado para el portfolio (qué especificar después de los planes 01-04)

| Prioridad | Idea | Por qué | Plan |
|---|---|---|---|
| 1 | **D1 Reseñas Google** | Vendible a todo cliente existente y nuevo; barato; ideal para vendedores | escribir `05-RESENAS.md` |
| 2 | **B1 Estudios contables** | Ticket alto, churn bajo, comprador profesional | escribir `06-CONTABLES.md` |
| 3 | **C1 Distribuidoras mayoristas** | Ticket más alto del portfolio, efecto red | escribir `07-DISTRIBUIDORAS.md` |
| 4 | **A2 Estética** + **A1 Escuelas de manejo** | Casi gratis con el módulo de paquetes de sesiones | skins del 01 |
| 5 | Resto del grupo B (consorcios, transporte, clubes) | Skins del módulo de cuotas (plan 04) | según demanda |

**Criterio para los vendedores**: cada vendedor debería salir con un paquete por rubro, no un producto suelto. Ej.: al salón le ofrece turnero + reseñas + fidelización; a la veterinaria turnos + calendario sanitario + reseñas. El producto ancla abre la puerta; los add-ons suben el ticket y bajan el churn (cuantos más módulos usa, menos se va).
