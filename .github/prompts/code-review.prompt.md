---
description: "Worker de revisión de código — analiza un PR o conjunto de cambios"
---

# Worker: Code Review

Revisa los cambios en: **$FILES_OR_PR**

## Checklist de revisión

### Corrección
- [ ] ¿La lógica es correcta y cumple los requisitos?
- [ ] ¿Hay casos edge no contemplados?
- [ ] ¿Los errores se manejan apropiadamente?

### Seguridad (ver guardrails)
- [ ] ¿No hay secrets ni datos sensibles expuestos?
- [ ] ¿Las entradas externas están validadas?
- [ ] ¿Se siguen los principios de mínimo privilegio?

### Calidad
- [ ] ¿El código es legible y autoexplicativo?
- [ ] ¿Las funciones tienen responsabilidad única?
- [ ] ¿Hay duplicación que debería extraerse?

### Tests
- [ ] ¿Existe cobertura para la lógica nueva?
- [ ] ¿Los tests son deterministas?
- [ ] ¿Se prueban los casos de error?

### Arquitectura
- [ ] ¿Los cambios respetan la Constitution?
- [ ] ¿No introduce dependencias circulares?
- [ ] ¿El SDD necesita actualizarse?

## Formato de feedback

Para cada hallazgo:
- **Severidad**: BLOQUEANTE / SUGERENCIA / NITPICK
- **Ubicación**: archivo:línea
- **Problema**: descripción clara
- **Propuesta**: cómo corregirlo
