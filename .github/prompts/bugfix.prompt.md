---
description: "Flujo para diagnosticar y corregir un bug"
---

# Worker: Bug Fix

Bug a corregir: **$BUG_DESCRIPTION**

## Paso 1 — Reproducción

Reproduce el bug localmente o describe los pasos exactos para reproducirlo.
Si hay un issue de GitHub, léelo completo.

## Paso 2 — Diagnóstico

- Identifica el archivo y línea donde ocurre el error
- Entiende POR QUÉ ocurre, no solo dónde
- Verifica si hay otros lugares en el código con el mismo problema

## Paso 3 — Test de regresión

Escribe un test que falle con el bug actual ANTES de corregirlo.
Esto garantiza que el bug no vuelva a aparecer.

## Paso 4 — Corrección

Aplica el fix mínimo necesario. No refactorices código no relacionado.

## Paso 5 — Verificación

- El test de regresión debe pasar
- Los tests existentes no deben romperse
- Verifica que no introduces un bug diferente

## Paso 6 — Entrega

Describe: qué era el bug, cuál era la causa raíz, qué cambiaste.
