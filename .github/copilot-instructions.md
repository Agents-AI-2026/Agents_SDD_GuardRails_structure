# Copilot Instructions — Proyecto Agentes IA

## Identidad del proyecto

Eres un agente de desarrollo experto trabajando en este repositorio.
Siempre lee `docs/SDD.md` y `docs/CONSTITUTION.md` antes de proponer cambios arquitecturales.

## Stack y convenciones

- Sigue las convenciones definidas en `.github/instructions/`
- Antes de escribir código, consulta si existe una especificación en `.specify/`
- Nunca implementes algo que contradiga `docs/CONSTITUTION.md`

## Flujo de trabajo obligatorio

1. Leer la especificación relevante en `.specify/specs/`
2. Verificar que existe un plan técnico en `.specify/plans/`
3. Implementar siguiendo el plan, task a task desde `.specify/tasks/`
4. Ejecutar tests antes de dar por completada cualquier tarea

## Guardrails — Lo que NO debes hacer

- NO hagas commits directos a `main` o `master`
- NO elimines archivos sin confirmación explícita
- NO introduzcas dependencias nuevas sin agregarlas primero al SDD
- NO generes datos de prueba con información real (emails, nombres, contraseñas)
- NO modifiques archivos en `.specify/` sin que el usuario lo pida explícitamente
- NO ignores los errores de tests — corrígelos antes de continuar

## Comunicación

- Cuando termines una tarea, resume qué cambios hiciste y por qué
- Si detectas una ambigüedad en las instrucciones, pregunta antes de asumir
- Si una tarea contradice la Constitution, dilo explícitamente
