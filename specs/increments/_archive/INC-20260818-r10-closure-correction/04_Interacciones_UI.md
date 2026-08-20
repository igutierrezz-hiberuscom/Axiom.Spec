# 04 Interacciones UI

## Objetivo del documento

Documentar los efectos observables para operadores, sin añadir una interfaz nueva.

## Superficie UI afectada

- CLI `axiom init --target <target>`.
- CLI `axiom configure` sobre estado ya persistido.
- Manuales de instalación y archivos generados.
- CLI Core de incrementos y Decision para lifecycle/vínculo.

## Flujo de interacción

1. El operador consulta o suministra un target público: sólo puede elegir uno de los ocho IDs canónicos.
2. Si suministra `copilot-vscode`, `init` falla de forma explícita y no escribe estado.
3. Si ejecuta `configure` sobre un proyecto antiguo cuyo `init.json` contiene el alias, el runtime lo migra internamente a `github-copilot` antes de materializar cualquier surface; la salida y el archivo persistido muestran únicamente el ID canónico.
4. Los manuales recomiendan `github-copilot`, no el alias ni LiteLLM.

## Estados visibles

- Rechazo público: mensaje de opción inválida con lista de targets válidos.
- Migración legacy: `install-profile.json` y `init.json` usan `github-copilot`; no existe una opción de CLI para activar la migración.
- Lifecycle: el plan R-10 se mantiene `pending` hasta la revisión independiente.

## Cascadas y comportamiento reactivo

No hay componentes web, prompts, llamadas de red ni actualizaciones reactivas nuevas. El cambio se limita a validación CLI, lectura/migración de estado local y documentación estática.
