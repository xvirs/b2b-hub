# Plantilla — Plan de Desarrollo de un Producto/Vertical

> Copiar este archivo como `NN-NOMBRE.md` y completar TODAS las secciones. Una spec está terminada cuando una IA sin contexto puede implementarla sin hacer preguntas de diseño. Si una sección genera dudas al escribirla, la generará al implementarla: resolvela acá.

Reglas de redacción:
- Decisiones, no opciones. "Se usa X" — nunca "se podría usar X o Y".
- Lo que NO se construye se lista explícitamente (es lo que más previene scope creep con IAs).
- Todo vertical hereda stack y convenciones de `00-INDICE.md` §Convenciones y se monta sobre `01-NUCLEO-MOTOR.md`; la spec del vertical documenta solo el DELTA.
- Cada fase: máx 2 semanas, termina deployada, criterios de aceptación demostrables (test, e2e, screenshot).

---

# Plan de Desarrollo NN — [Nombre del Producto]

> [Una línea: qué es y para quién.]
> **Audiencia**: IA/dev sin contexto. Prerrequisito: motor de `01-NUCLEO-MOTOR.md` implementado hasta Fase [N].

## 1. Qué se construye (resumen ejecutivo)
[3-6 bullets de qué hace el sistema. Después: lista explícita de qué NO se construye.]

## 2. Delta sobre el motor
[Qué módulos del núcleo se reusan tal cual, cuáles se extienden y qué es 100% nuevo. Tabla recomendada.]

## 3. Modelo de datos (delta)
[Solo tablas nuevas y columnas/cambios sobre las del motor. Mismas convenciones: UUID v7, tenant_id, centavos, E.164, timestamptz.]

## 4. Flujos de WhatsApp (delta)
[Plantillas nuevas a aprobar (texto exacto con variables y botones). Cambios a la FSM: estados nuevos, transiciones, textos de los mensajes del bot en español rioplatense.]

## 5. Panel web (delta)
[Pantallas nuevas o modificadas, con qué muestra y qué acciones permite cada una.]

## 6. Lógica de negocio específica
[Las reglas del rubro que no son obvias: cómo se calcula X, qué pasa cuando Y. Esta sección es la que más valor tiene — acá vive el conocimiento del vertical.]

## 7. Fases de implementación
[Fase a fase con duración estimada y criterios de aceptación ✅ demostrables. La Fase final siempre incluye: runbook de onboarding de un cliente nuevo de este vertical.]

## 8. Configuración del tenant
[Qué entra en `tenants.settings` jsonb para este vertical, con su schema Zod.]

## 9. Puntos a verificar al implementar
[APIs externas, tarifas, límites que pueden haber cambiado desde la escritura de esta spec.]

## 10. Extensiones previstas (no implementar)
[Qué viene después, para que el diseño no lo bloquee.]
