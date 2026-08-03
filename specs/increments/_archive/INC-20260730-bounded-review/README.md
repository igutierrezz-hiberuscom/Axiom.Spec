# INC-20260730-bounded-review

Status: closed
Date: 2026-07-30
Plan: F1.4

## Goal

Extender el contrato actual del `review-ledger` integrando el concepto de "Review acotada y tiering por riesgo" (Tier 0, 1, 2) y múltiples lentes (4R: risk, readability, reliability, resilience) de manera determinista, sin bucles de corrección infinitos. 

## Context

Axiom hoy tiene un reviewer generalista (lente `review`) y un `review-ledger-contract` con un bucle de pasadas exhaustivas (loop until dry). Para alinear con Gentle-AI, debemos incorporar *tiering* basado en la evidencia de riesgo (auth, permisos, migraciones, MCP, etc.) para definir el presupuesto y las lentes a usar, limitando las correcciones a una única iteración.

## Scope

- Modificar `packages/document-bootstrap/src/review-ledger-contract.ts` para incluir las reglas de Tier (Tier 0: 0 lenses, Tier 1: 1 lens dominante, Tier 2: 4R lenses).
- Modificar las reglas para limitar a 1 única corrección permitida (bounded review).
- Actualizar `REQUIRED_LEDGER_CLAUSES` y tests para cubrir los nuevos términos (Tier, 1 corrección, 4R, risk, auth).
- Actualizar prompts/agentes de revisión (ej. `.github/prompts/axiom-review.prompt.md`, `.agents/workflows/axiom-review.md`) si es necesario para sincronizarlos con el nuevo contrato.

## Non-goals

- Crear un segundo sistema de revisión paralelo.
- Cambiar la infraestructura de memoria o Engram.

## Acceptance criteria

- [x] El contrato en `review-ledger-contract.ts` define claramente los Tiers 0, 1 y 2 basados en riesgo empírico (auth, permisos, MCP, skills, memoria compartida).
- [x] El contrato limita explícitamente el ciclo a exactamente 1 corrección permitida.
- [x] La test suite de `packages/document-bootstrap` se ejecuta y pasa correctamente sin errores de drift en el string generado.
- [x] Todas las referencias a los tiers y reglas de riesgo están presentes en las superficies de `Axiom.SDD` correspondientes.

## Validation

- `npx vitest run packages/document-bootstrap/tests/review-ledger-contract.test.ts`
- Verificado y con drift resuelto.
