---
description: "Sub-agente de control de versiones para specs y planes de desarrollo"
applyTo: ".specify/specs/**,.specify/plans/**"
---

# Control de versiones — Specs y Planes

Este instruction file aplica automáticamente a todos los ficheros bajo `.specify/specs/` y `.specify/plans/`.
**Antes de modificar cualquier fichero de estas rutas**, sigue el protocolo de versiones descrito a continuación.

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
