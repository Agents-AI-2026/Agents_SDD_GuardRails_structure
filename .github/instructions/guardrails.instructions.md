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

## Clean Code y reutilización

### Principios SOLID
- **S** — Single Responsibility: una clase/función, una razón para cambiar
- **O** — Open/Closed: abierto a extensión, cerrado a modificación (usa interfaces y herencia, no condicionales masivos)
- **L** — Liskov Substitution: las subclases deben poder sustituir a sus clases base sin romper el comportamiento
- **I** — Interface Segregation: interfaces pequeñas y específicas; no fuerces a implementar métodos que no se usan
- **D** — Dependency Inversion: depende de abstracciones (interfaces), no de implementaciones concretas

### DRY (Don't Repeat Yourself)
- No dupliques lógica de negocio — extráela a un servicio o utilidad compartida
- En el frontend, extrae lógica repetida a custom hooks o funciones en `utils/`
- En el backend, extrae lógica común a clases de servicio o métodos privados reutilizables
- Antes de escribir código nuevo, verifica si ya existe algo similar en el proyecto

### Reutilización de componentes (Frontend)
- Los componentes React deben ser genéricos y parametrizables mediante props
- Evita hardcodear valores dentro de componentes — pásalos como props o constantes
- Los componentes de UI puros (sin lógica de negocio) van en `components/`; los que consumen datos van en `pages/`
- Extrae lógica de estado compleja a custom hooks en `hooks/`

### Reutilización de código (Backend)
- La lógica de negocio vive exclusivamente en la capa `service/`; los controllers solo delegan
- Los repositorios no contienen lógica de negocio — solo consultas a la BD
- Usa `@Component` / `@Service` / `@Repository` según la capa; no mezcles responsabilidades entre capas
- Los DTOs de request/response son independientes de las entidades JPA — usa MapStruct para mapear

### Legibilidad
- Las funciones no deben superar 30 líneas; si lo hacen, refactoriza
- Los métodos deben poder leerse como una oración: `calculateOrderTotal()`, `findActiveProductsByCategory()`
- Evita comentarios que expliquen *qué* hace el código — el código debe ser autoexplicativo; comenta solo el *por qué*

## Tests

- Escribe tests para toda lógica de negocio nueva
- Los tests deben ser deterministas (sin depender de tiempo real o datos externos)
- Usa mocks para dependencias externas en tests unitarios

## Git

- Los mensajes de commit siguen el formato: `tipo(alcance): descripción`
  - Tipos válidos: `feat`, `fix`, `docs`, `test`, `refactor`, `chore`
- No hagas commits directos a ramas protegidas (`main`, `master`, `release/*`)
- Un PR = una feature o fix, no mezcles cambios no relacionados
