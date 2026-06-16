---
description: "Guardrails de seguridad y calidad aplicables a todo el repositorio"
applyTo: "**"
---

# Guardrails — Reglas obligatorias

## Seguridad

- NUNCA incluyas secrets, API keys o contraseñas en el código
- Usa variables de entorno para toda configuración sensible
- Valida y sanitiza toda entrada que venga del usuario o de APIs externas
- No uses `eval()`, `exec()` ni equivalentes con input dinámico

## Calidad de código

- Cada función/método debe tener una única responsabilidad
- Nombra variables y funciones de forma descriptiva, sin abreviaciones crípticas
- Evita números mágicos — usa constantes con nombre

## Tests

- Escribe tests para toda lógica de negocio nueva
- Los tests deben ser deterministas (sin depender de tiempo real o datos externos)
- Usa mocks para dependencias externas en tests unitarios

## Git

- Los mensajes de commit siguen el formato: `tipo(alcance): descripción`
  - Tipos válidos: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`
- No hagas commits directos a ramas protegidas (`main`, `master`, `release/*`)
- Un PR = una feature o fix, no mezcles cambios no relacionados
