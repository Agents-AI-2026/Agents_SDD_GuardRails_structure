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
