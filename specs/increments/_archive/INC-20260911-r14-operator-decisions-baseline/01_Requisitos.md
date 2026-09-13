# 01 Requisitos

## Objetivo del documento

Registrar formalmente los requisitos y cierres de decisión para las dudas de arquitectura y diseño D-01 a D-06 del lote R-14.

## Decisiones registradas

- **D-01 (`DEC-20260911-223522-6ox3vz`):** Consolidación de `self-update-contract.ts` como superficie única de self-update y eliminación de `self-update.ts`. Preservación de `scripts/install-global.mjs` como bootstrap de instalación inicial externo.
- **D-02 (`DEC-20260911-223523-jii9h2`):** Desversionado de los 19 artefactos compilados bajo `src/` y 148 bajo `packages/adapters/*/dist/`. Refinamiento de `TC-009` para distinguir adapter ausente (FAIL) de falta de build (WARN/SKIP). Regla `.gitignore` y gate anti-regresión.
- **D-03 (`DEC-20260911-223524-dfl08s`):** Derivación de `RUNTIME_VERSION` desde los metadatos de build generados en tiempo de compilación. Respeto de estados persistidos en `0.1.0`. Valor por defecto de `--target-version` en upgrade mantiene la versión de runtime activa.
- **D-04 (`DEC-20260911-223525-y8j1xc`):** Contrato de recomendación y aviso por adapter. Nombramiento de `ModelClass` recomendada. Tres niveles de aviso: multi-mode (sin aviso), single-mode (aviso global de sesión), fallback-only (aviso de selección manual). Inserción en cabecera de prompt para acciones con slot SDD. Registro visible de limitaciones en diagnósticos y CLI `axiom model show`.
- **D-05 (`DEC-20260911-223526-127zvd`):** Retirada física total de `packages/tui/`, junction en `node_modules/@axiom/tui` y saneamiento de `package-lock.json`. Comprobaciones positivas de ausencia física en `tui-retirement.test.ts`.
- **D-06 (`DEC-20260911-223526-pxpi06`):** Soporte multi-instancia en `workflow-state.json` con `schemaVersion: 2`, selección explícita por `--id` y subcomando `select`, y purga de instancias terminales al archivar.

## Reglas de negocio relevantes

- No se realiza cambio de código de producto en este incremento.
- Toda decisión cuenta con su artefacto formal y trazabilidad hacia el plan general.
