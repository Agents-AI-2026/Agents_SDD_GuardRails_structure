---
description: "Sub-agente de control de versiones para specs y planes de desarrollo"
applyTo: ".specify/specs/**,.specify/plans/**"
---

# Control de versiones — Specs y Planes

Este instruction file aplica automáticamente a todos los ficheros bajo `.specify/specs/` y `.specify/plans/`.
**Antes de modificar cualquier fichero de estas rutas**, sigue las reglas de separación de responsabilidades y el protocolo de versiones descritos a continuación.

---

## Separación obligatoria: Spec vs Plan

Esta regla es **no negociable** y se aplica a todos los ficheros nuevos y existentes.

### `.specify/specs/` — Solo contenido funcional

Una spec describe **QUÉ** hace el sistema desde la perspectiva del usuario o del negocio. Nunca contiene detalles de implementación.

**Permitido en specs:**
- Descripción general del módulo (qué resuelve, para quién)
- Flujos funcionales: actor, entrada, salida y pasos desde la perspectiva del usuario
- Reglas de negocio y restricciones comportamentales
- Requisitos de seguridad a nivel de comportamiento (ej: "las contraseñas se almacenan de forma segura", "las sesiones expiran")
- Criterios de aceptación redactados en términos de comportamiento observable
- Casos edge desde la perspectiva de negocio

**Prohibido en specs:**
- Nombres de librerías, frameworks o algoritmos específicos (BCrypt, SameSite=Strict, S256, JWKS…)
- Esquemas de base de datos (tablas, columnas, tipos de datos)
- Endpoints HTTP y contratos de API (rutas, métodos, payloads, códigos de respuesta concretos)
- Detalles de implementación interna del backend (nombres de clases, métodos, eventos internos, nombres de tokens de proveedores)
- Configuración técnica (flags de cookies, timeouts en ms, formatos internos de token)
- Diagramas de secuencia técnica

### `.specify/plans/` — Solo contenido técnico

Un plan describe **CÓMO** se implementa lo que define la spec correspondiente.

**Debe contener en plans:**
- Stack tecnológico y justificación de decisiones
- Estructura de ficheros y capas de la aplicación
- Esquema de base de datos con tipos y migraciones
- Contratos de API (endpoints, métodos HTTP, DTOs, códigos de respuesta)
- Diagramas de arquitectura y secuencia técnica
- Decisiones técnicas y alternativas descartadas
- Dependencias externas y configuración

**Referencia cruzada obligatoria:** todo plan debe incluir `> Spec de referencia: .specify/specs/<nombre>.md` en su cabecera.

---

## Protocolo de versiones

### Al crear un fichero nuevo

Incluye obligatoriamente estos campos en la cabecera (tras la línea `#` del título):

```markdown
> Versión: v1
> Fecha creación: YYYY-MM-DD
> Fecha última modificación: YYYY-MM-DD
> Estado: BORRADOR
```

Añade la sección `## Changelog` al final del fichero (tras separador `---`):

```markdown
---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | YYYY-MM-DD | Creación inicial |
```

---

### Antes de modificar un fichero existente

1. Lee el fichero y localiza la línea `> Versión: vN` en su cabecera.
2. Incrementa N → N+1 en esa línea.
3. Actualiza `> Fecha última modificación` con la fecha de hoy.
4. Añade una nueva fila al final de la tabla `## Changelog` con:
   - La nueva versión (vN+1)
   - La fecha de hoy
   - Una descripción concisa del cambio que se va a realizar

**Si el fichero no tiene la cabecera versionada:** añádela retroactivamente como `v1` con la fecha de su campo `> Fecha:` original, luego aplica el bump a `v2` para la modificación en curso.

---

## Reglas de estado

| Estado | Cuándo usarlo |
|--------|---------------|
| `BORRADOR` | Fichero en elaboración, no validado |
| `ACTIVO` | Spec/plan aprobado y en ejecución |
| `OBSOLETO` | Reemplazado por otra spec/plan o feature cancelada |

Actualiza el campo `> Estado` cuando el estado real cambie.

---

## Esquema de versiones

Se usa versionado simple: `v1`, `v2`, `v3`…

- Cada modificación sustantiva (cambio de requisito, nuevo módulo, corrección de diseño) supone un bump.
- Correcciones tipográficas menores no requieren bump de versión.

---

## Ejemplo de cabecera completa

```markdown
# Especificación — Mi Feature

> Versión: v2
> Fecha creación: 2026-06-16
> Fecha última modificación: 2026-06-17
> Estado: BORRADOR
> Autor: equipo
```

---

## Ejemplo de CHANGELOG al final del fichero

```markdown
---
## Changelog

| Versión | Fecha | Descripción del cambio |
|---------|-------|------------------------|
| v1 | 2026-06-16 | Creación inicial |
| v2 | 2026-06-17 | Añadido módulo de pagos y casos edge de stock |
```
