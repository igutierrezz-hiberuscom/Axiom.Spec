# 01 Requisitos

## Objetivo del documento

Definir el cierre correctivo R-10 con comportamiento verificable, separando la API pública de la migración de datos legacy.

## Requisitos del incremento

1. `ADAPTER_TARGETS`, validación de `init`, tipos públicos, installer y dispatches deben exponer sólo `opencode`, `claude-code`, `antigravity`, `visual-studio-2026`, `cursor`, `github-copilot`, `vscode` y `codex`.
2. `axiom init --target copilot-vscode` y cualquier llamada pública equivalente deben rechazar el valor como inválido; no deben materializar archivos ni persistir un target alias.
3. Si `init.json` existente contiene `profileTriple.adapterTarget: "copilot-vscode"`, `configure` debe reconocer únicamente ese dato persistido, migrarlo explícitamente a `github-copilot` antes de `installProfile` y de la escritura de instrucciones, y continuar sólo con el ID canónico.
4. LiteLLM no puede figurar como import, target, dispatcher, ejemplo, archivo prometido ni mock activo fuera de historia explícita.
5. La creación Core de un incremento con `--id` debe comenzar desde `draft` aunque el singleton del workflow se haya quedado en `archived` para otro `metadataId`.
6. Los documentos operativos deben declarar exactamente los ocho targets actuales y la única surface Copilot `github-copilot`.
7. Los registros humanos de ACC-039/040/041/042/045, ACC-044 y la reparación de coherencia deben diferenciar archivo histórico, evidencia anterior y resultado corregido; no pueden afirmar resultados que el código no ofrece.
8. La Decision Cavekit debe usar sólo el vínculo Core `link-increment`; su prosa no puede afirmar una supersesión formal inexistente.
9. Las specs y contexto activos deben representar el contrato final y marcar historia sólo como historia.

## Reglas de negocio relevantes

- Los artefactos ACC existentes permanecen archivados; sus estados e índices no se editan manualmente.
- La migración de un alias ocurre exclusivamente al leer estado persistido y no convierte el alias en parte de una API, tipo o target materializable.
- El cierre requiere validación ejecutada y re-review independiente sin blockers.

## Fuera de alcance funcional

No se cambia la semántica de otros targets, el contenido histórico de 0015, el grafo funcional R-10 ni los mecanismos de Decision más allá de su vínculo soportado y sus claims humanos.
