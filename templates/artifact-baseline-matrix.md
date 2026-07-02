# Matriz de baseline por artefacto

## Objetivo

Esta matriz define qué archivos y carpetas forman parte de la baseline de un repo spec de Axiom, cuáles son obligatorios, cuáles son condicionales y cuál es su fuente de verdad.

## Reglas globales de baseline

- La baseline es mínima: solo se materializan archivos cuando aplican.
- La fuente de verdad por artefacto vive en `metadata.yml` o `plan.metadata.yml`.
- Los registros maestros globales son derivados y se generan por script o instalador cuando hagan falta.
- Los incrementos y bugs cerrados o integrados se mueven físicamente a `archive/`.
- No existe `CHANGELOG.md` por artefacto en la baseline inicial.
- `technical-context/` es obligatorio y se separa de `context/`.
- `context/` sirve como bolsa única de material adicional y no se divide entre contexto y evidencia.

## Spec general canónica

| Ruta | Estado | Regla |
|------|--------|-------|
| `specs/README.md` | Obligatorio | Índice navegable de la spec general |
| `specs/00_Resumen_Ejecutivo.md` | Obligatorio | Visión y alcance del producto |
| `specs/00_Glosario.md` | Obligatorio | Terminología canónica |
| `specs/01_Requisitos_Funcionales.md` | Obligatorio | RF trazables |
| `specs/02_Requisitos_No_Funcionales.md` | Obligatorio | NFR y restricciones transversales |
| `specs/03_Modelo_Dominio_o_Datos.md` | Condicional | Solo si hay modelo, contratos, estados o estructura relevantes |
| `specs/04_Flujos_Negocio.md` | Condicional | Solo si hay flujos funcionales multi-paso relevantes |
| `specs/05_Interfaces_Usuario.md` | Condicional | Solo si existe superficie UI relevante |
| `specs/06_Integraciones.md` | Condicional | Solo si hay integraciones internas o externas relevantes |
| `specs/07_Seguridad.md` | Condicional | Solo si seguridad, permisos o auditoría requieren documento propio |
| `specs/increments/` | Obligatorio | Familia de incrementos activos |
| `specs/increments/archive/` | Obligatorio | Destino físico de incrementos archivados |
| `specs/bugs/` | Obligatorio | Familia de bugs activos |
| `specs/bugs/archive/` | Obligatorio | Destino físico de bugs archivados |
| `specs/e2e/` | Condicional | Solo si la instalación activa flujo E2E |

## Incrementos

| Ruta | Estado | Regla |
|------|--------|-------|
| `README.md` | Obligatorio | Resumen, alcance, trazabilidad y consolidación |
| `metadata.yml` | Obligatorio | Fuente de verdad estructural del incremento |
| `01_Requisitos.md` | Obligatorio | Requisitos funcionales del incremento |
| `03_Criterios_Aceptacion.md` | Obligatorio | Criterios observables y verificables |
| `02_Cambios_Modelo.md` | Condicional | Solo si hay cambio real de modelo, contrato, estado o estructura |
| `04_Interacciones_UI.md` | Condicional | Solo si hay interacción visible relevante o comportamiento reactivo |
| `context/` | Opcional | Material añadido del artefacto |
| `artifacts/plans/` | Condicional | Solo si existe al menos un plan |

## Bugs

| Ruta | Estado | Regla |
|------|--------|-------|
| `README.md` | Obligatorio | Puede contener toda la spec del bug si es pequeño |
| `metadata.yml` | Obligatorio | Fuente de verdad estructural del bug |
| `01_Requisitos.md` | Condicional | Solo si el bug necesita separar reglas o requisitos afectados |
| `03_Criterios_Aceptacion.md` | Condicional | Solo si los criterios no caben de forma clara en el README |
| `02_Cambios_Modelo.md` | Condicional | Solo si hay cambio real de modelo, contrato, estado o estructura |
| `04_Interacciones_UI.md` | Condicional | Solo si hay interacción visible relevante o comportamiento reactivo |
| `context/` | Opcional | Material añadido del artefacto |
| `artifacts/plans/` | Condicional | Solo si existe al menos un plan |

## Planes

| Ruta | Estado | Regla |
|------|--------|-------|
| `README.md` | Obligatorio | Portada y resumen del plan |
| `plan.metadata.yml` | Obligatorio | Fuente de verdad estructural del plan |
| `role-<slug>.md` | Condicional | Obligatorio por cada rol impactado por el plan |
| `context/` | Opcional | Material añadido del plan |

## Technical Context

| Ruta | Estado | Regla |
|------|--------|-------|
| `technical-context/TECHNICAL_CONTEXT.md` | Obligatorio | Puerta de entrada al conocimiento técnico estable |
| `technical-context/architecture/` | Opcional | Profundización de arquitectura |
| `technical-context/conventions/` | Opcional | Convenciones técnicas |
| `technical-context/operations/` | Opcional | Reglas operativas y runbooks |
| `technical-context/testing/` | Opcional | Estrategia y límites de testing |
| `technical-context/integrations/` | Opcional | Contexto técnico de integraciones |
| `technical-context/references/` | Opcional | Referencias curadas |

## Contexto añadido

| Ruta | Estado | Regla |
|------|--------|-------|
| `context/README.md` | Condicional | Solo si existe contexto añadido a la spec general |
| `context/manuals/` | Opcional | Manuales de usuario o anexos operativos |
| `context/screenshots/` | Opcional | Capturas o materiales visuales |
| `context/meeting-notes/` | Opcional | Notas o transcripciones relevantes |
| `context/annexes/` | Opcional | Material complementario diverso |