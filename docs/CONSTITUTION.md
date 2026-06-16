# Constitution — Principios No Negociables

> Este documento define los principios arquitecturales inmutables del proyecto.
> Ningún agente, desarrollador o pull request puede violarlo sin aprobación explícita del equipo.

## 1. Seguridad primero

- Nunca exponer credenciales, tokens o secrets en el código
- Toda entrada externa debe ser validada antes de procesarse
- Los endpoints públicos requieren autenticación

## 2. Calidad del código

- Todo código nuevo debe tener tests
- La cobertura mínima aceptable es 80%
- No se permite código muerto ni funciones sin usar

## 3. Arquitectura

- Separación clara de responsabilidades (capas bien definidas)
- Sin dependencias circulares entre módulos
- Los cambios de esquema de base de datos requieren migraciones versionadas

## 4. Proceso

- Toda feature parte de una especificación en `.specify/specs/`
- Los PRs deben referenciar su especificación correspondiente
- No se hace merge sin que los checks de CI estén en verde

## 5. Deuda técnica

- Se registra en `docs/TECH_DEBT.md` con fecha y responsable
- No se acumula deuda sin plan de pago documentado
