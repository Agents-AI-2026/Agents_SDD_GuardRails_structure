# .specify — Spec-Driven Development

Este directorio contiene toda la documentación de especificación que guía al agente.

## Estructura

```
.specify/
├── specs/     # Qué construir (requisitos funcionales)
├── plans/     # Cómo construirlo (diseño técnico detallado)
└── tasks/     # Trabajo atómico para el agente
```

## Flujo SDD

```
Idea → SPEC → PLAN → TASKS → Implementación → PR
```

## Cómo crear una nueva feature

1. Crea `specs/mi-feature.md` describiendo QUÉ hace
2. Crea `plans/mi-feature.md` describiendo CÓMO se implementa
3. Crea `tasks/mi-feature.md` con la lista de tareas atómicas
4. Pide al agente que ejecute `/new-feature` con el nombre de la feature

## Convención: sincronización specs ↔ tasks

> **Regla**: cada vez que se modifica un `spec` o un `plan`, el archivo de `tasks` correspondiente debe actualizarse en el mismo cambio.

### Qué actualizar según el tipo de cambio

| Cambio en spec/plan | Acción en tasks |
|---------------------|-----------------|
| Nueva entidad / tabla | Añadir subtarea de migración Flyway en la épica correspondiente |
| Nuevo endpoint API | Añadir TASK con contrato, subtareas de implementación y criterio de aceptación |
| Nuevo componente frontend | Añadir subtarea dentro de la TASK de su épica o crear TASK nueva si es significativo |
| Cambio de stack / librería | Actualizar subtareas afectadas y criterios de aceptación |
| Nuevas variables de entorno | Añadir subtarea en la TASK de infraestructura / `.env.example` |
| Cambio de path / estructura de carpetas | Revisar rutas referenciadas en subtareas |
| Decisión de arquitectura (ADR) | Actualizar criterios de aceptación de las TASKs afectadas |

### Archivos de tasks por módulo

| Módulo | Spec | Plan | Tasks |
|--------|------|------|-------|
| Tienda base | `specs/tienda-muebles-online.md` | `plans/tienda-muebles-online.md` | `tasks/tienda-muebles-online-tasks.md` |
| Autenticación | `specs/login.md` | `plans/login.md` | `tasks/login-tasks.md` |
| Pasarela de pagos | `specs/pasarela-pagos.md` | `plans/pasarela-pagos.md` | `tasks/pasarela-pagos-tasks.md` |
| Frontend Juvise | `specs/juvise-frontend.md` | `plans/juvise-frontend.md` | `tasks/juvise-frontend-tasks.md` |
