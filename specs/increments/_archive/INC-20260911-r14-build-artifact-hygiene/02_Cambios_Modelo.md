# 02 Cambios de Modelo

## Objetivo del documento

Detallar los cambios en la configuración de exclusión y en el comportamiento del doctor.

## Modificaciones de configuración

- `Axiom/.gitignore`:
  - `packages/*/*/dist/`
  - `packages/*/*/*.tsbuildinfo`
  - `packages/**/src/**/*.js`
  - `packages/**/src/**/*.d.ts`
  - `packages/**/src/**/*.map`
  - `apps/**/src/**/*.js`
  - `apps/**/src/**/*.d.ts`
  - `apps/**/src/**/*.map`

## Comportamiento de comprobaciones (`TC-009`)

- `runAdapterRuntimeCoverageCheck`:
  - Evalúa la existencia de `src/generator.ts` para los 8 adapters canónicos (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `codex`, `antigravity`, `visual-studio-2026`).
  - Si alguno carece de generador: `FAIL`.
  - Si todos tienen generador y además `dist/index.js`: `PASS`.
  - Si falta `dist/index.js` pero el generador está presente: `WARN` informativo indicando qué adapters requieren `npm run build`.
