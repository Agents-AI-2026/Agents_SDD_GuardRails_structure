---
description: "Flujo completo para implementar una nueva feature siguiendo SDD"
---

# Worker: Nueva Feature

Sigue estos pasos en orden para implementar la feature: **$FEATURE_NAME**

## Paso 1 — Especificación

Lee `.specify/specs/` y verifica si ya existe una spec para esta feature.
Si no existe, crea `.specify/specs/$FEATURE_NAME.md` con:
- Qué hace la feature
- Criterios de aceptación
- Casos edge a contemplar

## Paso 2 — Plan técnico

Crea `.specify/plans/$FEATURE_NAME.md` con:
- Archivos a crear/modificar
- Cambios de modelo de datos (si aplica)
- APIs nuevas o modificadas

## Paso 3 — Implementación

Implementa task a task según el plan. Por cada tarea:
1. Escribe el código
2. Escribe los tests correspondientes
3. Verifica que los tests pasen

## Paso 4 — Revisión

Antes de terminar:
- [ ] ¿Cumple todos los criterios de aceptación de la spec?
- [ ] ¿Los tests cubren los casos edge?
- [ ] ¿Se respetan los guardrails de `.github/instructions/guardrails.instructions.md`?
- [ ] ¿No viola ningún principio de `docs/CONSTITUTION.md`?

## Paso 5 — Entrega

Resume los cambios realizados y crea el PR referenciando la spec.
