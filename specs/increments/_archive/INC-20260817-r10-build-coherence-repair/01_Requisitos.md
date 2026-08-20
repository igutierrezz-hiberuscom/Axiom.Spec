# 01 Requisitos

## Objetivo del documento

Restaurar la coherencia de compilación R-10 sin reabrir decisiones funcionales: retirar residuos activos de LiteLLM y de `copilot-vscode`, y alinear los consumidores MCP y Visual Studio con sus contratos públicos actuales.

## Requisitos del incremento

1. El registry, tipos, instalador, documentación operativa y dispatches materializables deben exponer únicamente los ocho targets canónicos: `opencode`, `claude-code`, `antigravity`, `visual-studio-2026`, `cursor`, `github-copilot`, `vscode` y `codex`.
2. LiteLLM no debe quedar como import, target, dispatcher, ejemplo operativo, archivo prometido ni mock activo.
3. `copilot-vscode` no debe ser una entrada, tipo ni dispatch público; la compatibilidad de datos persistidos queda fuera de este incremento y se documenta en el correctivo R-10 posterior.
4. Los consumidores MCP deben usar `McpServerContext` y mantener la validación project-scoped antes de proyectar configuración.
5. El caller de Visual Studio debe usar exclusivamente la API pública actual de su generator.

## Reglas de negocio relevantes

- Los targets son un vocabulario público cerrado; un identificador retirado no se reintroduce para satisfacer tipos o tests.
- La proyección MCP sólo puede escribir dentro del proyecto resuelto y conserva las entradas no gestionadas del usuario.
- El arreglo es de coherencia: no modifica el workflow, el modelo de QA ni el ciclo de vida de los artefactos.

## Fuera de alcance funcional

No se restauran LiteLLM ni aliases públicos, no se agregan targets, no se rediseña MCP ni se altera el comportamiento de lifecycle/plan/QA. Las decisiones históricas y la metadata estructural de este incremento no se modifican.
