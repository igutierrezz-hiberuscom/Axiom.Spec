# R-10 corrección de cierre

> **Código**: INC-20260818-r10-closure-correction
> **Estado**: Implementación y revalidación completadas; freeze y cierre gobernado pendientes
> **Fecha de creación**: 2026-08-18
> **Tipo de cambio**: corregir

## Resumen

Corregir los blockers de la revisión independiente posterior al primer cierre R-10 y volver a evaluar el cierre con evidencia actual. Este incremento sustituye la conclusión de cierre prematura; no reabre ni reescribe los ACC archivados.

## Contexto y motivación

La revisión independiente posterior al primer cierre R-10 detectó una divergencia entre runtime, pruebas y contratos documentales. El correctivo eliminó la aceptación pública de `copilot-vscode`, preservó únicamente la migración acotada desde `init.json#profileTriple.adapterTarget`, retiró mocks LiteLLM restantes y reconcilió los contratos activos. Una segunda revisión encontró además claims obsoletos de MCP dividido y cobertura total de `sync`; se corrigieron para reflejar el broker único `axiom-mcp-broker` y el dispatcher real de cinco targets.

## Alcance

### Incluido

- Corregir el runner Core para que un nuevo incremento identificado no herede el singleton terminal de una instancia archivada.
- Retirar `copilot-vscode` de interfaces públicas, tipos, dispatches, ejemplos y tests; `axiom init --target copilot-vscode` debe rechazarlo inequívocamente.
- Normalizar de forma interna, explícita y acotada un `init.json` persistido legacy antes de cualquier installer o dispatcher, migrándolo a `github-copilot`.
- Retirar referencias activas a LiteLLM y mocks que no prueben compatibilidad persistida.
- Reconstruir `docs/installation.md`, `docs/generated-files.md` y cualquier documentación operativa R-10 afectada para los ocho targets vigentes.
- Rectificar el informe humano de ACC-044, los README archivados ACC-039/040/041/042/045 y el cierre del incremento de coherencia, sin alterar su metadata, IDs, receipts ni ubicación.
- Enlazar la decisión Core de Cavekit al correctivo mediante la única relación disponible y corregir sus claims: una Decision no implementa supersesión.
- Consolidar el contrato final de aliases, Cavekit y R-10 en `specs/00..08` y `context/**` sólo tras verificar el runtime.

### Excluido

- Reabrir, mover o editar metadata de los ACC archivados.
- Restaurar LiteLLM, crear aliases públicos, añadir targets, MCP obligatorio, Workbench o arquitectura adicional.
- Inventar una relación de supersesión para Decisions o modificar el antecedente histórico 0015.
- Mutaciones Git.

## Documentos del incremento

- `01_Requisitos.md`: contrato observable y límites.
- `02_Cambios_Modelo.md`: migración legacy y lifecycle Core.
- `03_Criterios_Aceptacion.md`: pruebas y condiciones de cierre.
- `04_Interacciones_UI.md`: comportamiento público de CLI y documentación.

## Dudas abiertas

Ninguna. La decisión es rechazar el alias en toda entrada pública y aceptar sólo su normalización de un estado persistido legacy, antes del dispatcher.

## Decisiones funcionales cerradas

1. Los ocho IDs canónicos son la única API pública de targets; `copilot-vscode` no es un alias público.
2. Un estado legacy puede migrarse internamente a `github-copilot`; esta excepción no añade un target, tipo o dispatch público.
3. LiteLLM está retirado, sin mocks ni documentación operativa activa.
4. `DEC-20260818-134600-3jfjak` sólo puede enlazarse a un incremento; describe 0015 como antecedente histórico y la retirada como decisión adoptada, sin claim de supersesión formal.
5. R-10 queda pendiente hasta re-revisión independiente sin blockers y validación completa actual.

## Resultado implementado

- `ADAPTER_TARGETS` contiene exactamente los ocho IDs canónicos.
- `runInit` y el parser CLI rechazan literalmente `copilot-vscode` antes de toda instalación o escritura; `init.test.ts` cubre ambas rutas.
- `migrateLegacyInitAdapterTarget` es la única migración productiva del literal y persiste `github-copilot` antes de installer/dispatcher.
- `writeCopilotInstructions` conserva el tipo restringido `github-copilot | visual-studio-2026` y su guard runtime anterior a I/O.
- Se retiraron los dos mocks LiteLLM activos que quedaban en pruebas de workflow.
- Specs 00..08, contexto y documentación operativa reflejan ocho targets, LiteLLM retirado, broker único `axiom-mcp-broker` y cobertura real de `sync` para cinco targets.

## Consolidación en la spec general

El conocimiento estable ya quedó reconciliado en los contratos vigentes de targets, lifecycle, MCP y adapters bajo `specs/00..08` y `context/**`. Los IDs MCP anteriores permanecen únicamente en snapshots históricos explícitos o como IDs de limpieza de configuración stale. La declaración terminal de cierre R-10 se registrará solo después de que verify y archive concluyan correctamente.

## Estrategia E2E

La cobertura implementada prueba el rechazo público por CLI/API, la normalización de `init.json` legacy previa al configure/dispatcher, el set exacto de ocho targets, el guard de escritura Copilot y la creación Core después de un singleton archivado. Los gates finales repiten la suite focal de 11 archivos, build, Vitest completo, doctor, readiness y `git diff --check` en ambos repos.

## Trazabilidad y fuentes

Revisión independiente posterior R-10; `apps/cli/src/commands/{init,axiom-increment,native-mcp-config,workspace-mcp}.ts`; `packages/cli-commands/src/commands/{configure,sync}.ts`; `packages/{capability-model,install-profiles,installer}`; documentación operativa; ACC-039..045; `DEC-20260818-134600-3jfjak`; specs 01/04/05/06/07/08 y `context/**`.

## Estado de validación humana

Revalidación completa ejecutada sobre el candidato técnico: suite focal exacta `11/11` archivos y `146/146` tests; `sync.test.ts` `1/1` y `11/11`; lote MCP complementario `7/7` y `59/59`; build limpio; suite completa `330/330` archivos y `3300/3300` tests; doctor `PASS` (`45/60`, 0 fallos, 4 warnings, 11 omitidos); readiness `PASS`; `git diff --check` limpio en ambos repos. Las búsquedas finales confirman `copilot-vscode` únicamente en rechazo público, migración acotada y pruebas negativas; LiteLLM únicamente en una aserción negativa y una nota histórica; los IDs MCP divididos únicamente en cleanup productivo o suites históricas íntegramente comentadas. La primera re-revisión independiente validó AC1–AC7 y detectó blockers documentales en AC8: protocolo MCP antiguo y alcance incorrecto de `sync` en specs 02/03/05/06 y contexto de adapters. Esos claims se reconciliaron con el runtime (`axiom-mcp-broker`, `kind=axiom`, `server/discover`, cinco dispatches de `sync`). La segunda revisión confirmó esas correcciones y redujo AC8 a una única definición residual de TC-019 en `specs/08_Glosario.md`; ya fue alineada con `server/discover`. Quedan pendientes el freeze final, la revisión independiente terminal, `axiom-increment verify` y archive; hasta entonces el estado gobernado sigue siendo `plan-approved`.
