# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos para la consolidación de una única superficie de self-update en la CLI y la eliminación completa de la implementación previa huérfana (`ACC-083`).

## Requisitos del incremento

- **REQ-083-01: Consolidación de `self-update-contract.ts`**
  Mantener exclusivamente `apps/cli/src/commands/self-update-contract.ts` como la implementación gobernada de `axiom self-update`, con operaciones `status`, `check`, `plan`, `apply`, `recover` y envelope JSON schemaVersion 1.

- **REQ-083-02: Eliminación completa de `self-update.ts` legacy**
  Eliminar `apps/cli/src/commands/self-update.ts` (27,7 KB) y su archivo de pruebas dedicado `apps/cli/tests/self-update.test.ts`.

- **REQ-083-03: Preservación de `scripts/install-global.mjs`**
  Conservar `scripts/install-global.mjs` y `scripts/install-global.test.mjs` como el mecanismo oficial de instalación inicial del shim de bootstrap fuera del CLI.

- **REQ-083-04: Cobertura de no-regresión**
  Verificar en `apps/cli/tests/self-update-contract.test.ts` que los flags y subcomandos legacy son rechazados con mensajes claros de error y que ninguna función legacy queda exportada.

## Reglas de negocio relevantes

- No se reabre el contrato transaccional de `ACC-077`..`ACC-080`.
- La separación entre "instalación inicial del shim" (`scripts/install-global.mjs`) y "actualización transaccional del producto" (`axiom self-update`) queda formalizada sin solapamiento.
