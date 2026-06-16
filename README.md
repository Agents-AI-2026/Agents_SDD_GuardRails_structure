# Proyecto — Agentes IA con SDD y Guardrails

Estructura base para desarrollar con agentes IA (GitHub Copilot) siguiendo Spec-Driven Development.

## Estructura

```
.
├── .github/
│   ├── copilot-instructions.md        # Instrucciones globales para Copilot
│   ├── instructions/
│   │   └── guardrails.instructions.md # Guardrails automáticos por tipo de archivo
│   └── prompts/
│       ├── new-feature.prompt.md      # /new-feature — worker para features
│       ├── bugfix.prompt.md           # /bugfix — worker para bugs
│       └── code-review.prompt.md      # /code-review — worker de revisión
├── .specify/
│   ├── specs/    # Especificaciones (QUÉ construir)
│   ├── plans/    # Planes técnicos (CÓMO construirlo)
│   └── tasks/    # Tareas atómicas para el agente
└── docs/
    ├── CONSTITUTION.md  # Principios arquitecturales no negociables
    ├── SDD.md           # Software Design Document
    └── TECH_DEBT.md     # Registro de deuda técnica
```

## Flujo de trabajo con Copilot

1. Define la feature en `.specify/specs/`
2. Crea el plan técnico en `.specify/plans/`
3. En Copilot Chat: `/new-feature` para ejecutar el worker
4. El agente implementa siguiendo los guardrails automáticamente

## Recursos

- [GitHub Copilot Custom Instructions](https://docs.github.com/copilot/customizing-copilot/adding-custom-instructions-for-github-copilot)
- [GitHub Spec Kit](https://github.com/github/spec-kit)
- [Prompt Files — GitHub Docs](https://docs.github.com/en/copilot/tutorials/customization-library/prompt-files)
