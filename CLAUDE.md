# B2B-SaaS — Negocio de SaaS por WhatsApp para comercios locales (Córdoba) 🇦🇷

> **Abrí Claude Code DESDE esta carpeta** para todo lo que cruce producto + ventas + estrategia.
> Acá vive el negocio entero. Para codear fuerte un producto puntual, abrí su repo (ej. `turnero/`),
> que tiene su propio CLAUDE.md técnico.

## Qué es esto
Un portfolio de productos SaaS B2B para comercios locales, todos sobre un **motor común**
(Next.js + Postgres/Drizzle + WhatsApp Cloud API oficial). La idea: construir el motor una vez y
sacar verticales (salones, canchas, veterinarias, gimnasios…) rápido. Modelo: **construir primero,
vender después** con un kit que hace la venta delegable.

## Mapa de carpetas
```
B2B-SaaS/
├── Planes-B2B/        ESPECIFICACIONES (la fuente de verdad del producto)
│   ├── 00-INDICE.md           convenciones globales + cómo usar las specs con una IA
│   ├── 01-NUCLEO-MOTOR.md     ⭐ el motor + vertical salones (el plan más importante)
│   ├── 02..04-VERTICAL-*.md   canchas / veterinarias / gimnasios
│   ├── 05-MAQUINA-VENTAS.md   estrategia de ventas (qué se automatiza y qué no)
│   ├── 06-PROSPECTOR.md       spec del mini-CRM de leads (aún sin construir)
│   └── tecnico/T1,T2,T3       schema DB, contratos de paquetes, API/jobs
├── Gestion Wssp/      NEGOCIO: PRODUCTO.md (mercado, pricing, competencia, riesgos)
├── Ventas/            KIT DE VENTAS usable (lo operativo para vender)
│   ├── KIT-VENTAS-SALONES.md          pitch 90s + calculadora del dolor + 8 objeciones
│   ├── demo-bot-whatsapp.html         mockups del bot para mostrar en el cel
│   ├── planilla-seguimiento-*.xlsx    mini-CRM en Excel (embudo automático)
│   └── landing-salones.html           landing con calculadora de no-shows (editar el WhatsApp)
├── turnero/           ⚙️ EL CÓDIGO del motor (repo git propio, su CLAUDE.md técnico) — producto 1
├── PLAN-MAESTRO.html / plan-maestro-site/   presentación visual del plan
└── IDEAS-B2B-LOCALES.md                     banco de ideas de productos
   [a futuro: canchas/, vet/, gym/ como repos hermanos de turnero/]
```

## Orden de autoridad ante conflicto
Specs técnicas (`Planes-B2B/tecnico/` T1>T2>T3) > planes de producto (`Planes-B2B/0X`) > negocio
(`Gestion Wssp/PRODUCTO.md`). El stack está FIJADO en `Planes-B2B/00-INDICE.md` y no se renegocia.

## Estado del negocio (2026-06)
- **Producto 1 (turnero / salones)**: motor Fases 0–5 prácticamente completo y testeado (49 tests).
  Bot de WhatsApp (reservas + recordatorios + "mis turnos" + handoff), panel completo (agenda,
  ABM, métricas, reprogramar, vista del día). **Falta solo lo "en vivo"** (no es código):
  - **Meta WhatsApp**: bloqueado del lado del usuario (el SMS de verificación de la cuenta de
    desarrollador no llega — problema de ruteo a Argentina).
  - **Hosting**: Vercel plan Hobby bloquea deploys por uso comercial. **Decisión diferida**:
    recomendado **Vercel Pro (~US$20/mes)** o Railway. Para probar gratis: túnel cloudflared a
    localhost. Detalle en `turnero/docs/PUESTA-EN-VIVO.md`.
- **Ventas**: kit listo en `Ventas/` para empezar a **pre-vender** salones de Córdoba mientras se
  destraba lo de arriba (conseguir "sí, lo pruebo" y cerrar al encender Meta).

## Cómo trabajar acá
- **Estrategia / ventas / planear verticales** → Claude Code desde `B2B-SaaS/` (ve todo, memoria
  compartida del negocio).
- **Codear un producto** → Claude Code desde su repo (`turnero/`); usa su CLAUDE.md técnico y su git.
- Las specs y el kit son **autocontenidos**: cualquier sesión o IA los lee y entiende todo sin contexto previo.
