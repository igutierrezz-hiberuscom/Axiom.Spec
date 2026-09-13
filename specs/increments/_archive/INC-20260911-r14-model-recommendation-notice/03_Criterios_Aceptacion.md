# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-086-01: Ausencia de aviso en multi-mode
Para `opencode` (`multi-mode`), ningún prompt copiado contiene aviso de selección manual ni de sesión global.

### AC-086-02: Aviso global en single-mode
Para `claude-code` (`single-mode`), los prompts de acciones con slot contienen el aviso indicando que el modelo aplica a toda la sesión y especifican la clase recomendada.

### AC-086-03: Aviso de selección manual en fallback-only
Para destinos `fallback-only` (`github-copilot`, `vscode`, `cursor`, etc.), los prompts de acciones con slot contienen el aviso indicando que el enrutado automático no está soportado y que la selección de la clase recomendada debe ser manual.

### AC-086-04: Presentación clara en `axiom model show`
`axiom model show` muestra la recomendación de la política y el estado real del target para cada slot.

### AC-086-05: Tests automatizados
Pruebas unitarias en `@axiom/model-routing` y `@axiom/launcher` verifican los 3 niveles de soporte, snapshots de prompt y ausencia de aviso en acciones sin slot.
