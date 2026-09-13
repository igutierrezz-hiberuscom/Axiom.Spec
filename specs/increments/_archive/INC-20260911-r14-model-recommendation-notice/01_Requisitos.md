# 01 Requisitos

## Objetivo del documento

Definir los requisitos para el contrato de recomendación de modelo y aviso dependiente del adapter al copiar prompts y al inspeccionar el enrutado en la CLI (`ACC-086`).

## Requisitos del incremento

- **REQ-086-01: Cálculo del modelo recomendado por slot**
  Para cada slot (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`), calcular la `ModelClass` recomendada (`cheap`, `medium`, `strong`, `local`) a partir de la política base y los overrides project-scoped, independientemente del nivel de soporte del destino.

- **REQ-086-02: Aviso condicionado en prompts del launcher**
  Al generar/copiar prompts en `packages/launcher/`:
  - `multi-mode` (`opencode`): No emitir aviso (el enrutado por slot es automático).
  - `single-mode` (`claude-code`): Emitir aviso indicando que el modelo aplica a nivel global de sesión y señalar la clase recomendada.
  - `fallback-only` (`github-copilot`, `vscode`, `cursor`, `codex`, `antigravity`, `visual-studio-2026`): Emitir aviso indicando que la selección es manual y cuál es la clase de modelo recomendada.
  - Acciones sin slot asociado: no emiten aviso.

- **REQ-086-03: Presentación honesta en CLI `axiom model show`**
  `axiom model show` debe mostrar la clase recomendada para cada slot y declarar explícitamente el soporte del adapter (`multi-mode`, `single-mode` o `fallback-only`), evitando enmascarar la recomendación bajo un fallback plano.

- **REQ-086-04: Registro efectivo de la limitación**
  Materializar la regla `fallback.whenPerSubagentRoutingUnsupported: use-default-and-record-limitation` mediante la exposición estructurada de las limitaciones registradas en la resolución de modelos.

## Reglas de negocio relevantes

- No se promete ni simula control sobre destinos `fallback-only`: el comportamiento es 100% honesto hacia el operador.
- El `SUPPORT_MATRIX` de `@axiom/model-routing` es la única fuente de verdad sobre el nivel de soporte por adapter.
