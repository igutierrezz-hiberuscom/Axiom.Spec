# 02 Cambios de Modelo

## Objetivo del documento

Separar contrato activo, evidencia ejecutable, historia y resolución final de cada divergencia investigada en R-10.

## Informe final caso a caso

| Tema | Contrato activo final | Evidencia o historia relevante | Resolución comprobable |
| --- | --- | --- | --- |
| Sintaxis SDD | Los entrypoints públicos son con guion y `axiom state`; los 19 intents son internos. | Las formas con espacio eran promesas documentales históricas. | ACC-039 las retiró de superficies activas. |
| Approval y plan | El runner común aplica preview/confirmación y `plan-approve` sólo permite `draft → plan-approved`. | Antes, `requiresApproval` era declarativo en rutas divergentes. | ACC-041/042 quedaron integrados; tests cubren CLI y launcher. |
| YAML/default | Un YAML presente válido gobierna; sólo la ausencia usa el asset empaquetado; error o schema futuro fallan cerrado. | El fallback silencioso y el default editable duplicado son estado anterior. | ACC-045 implementó el resolvedor común. |
| Archive/integrate y QA | El runner coordina archive recuperable y `QaArchiveDecision` aplica la matriz común. | Helpers y coordinaciones por superficie divergían. | ACC-041/043 resolvieron la divergencia. |
| Cavekit | No hay package ni consumidor runtime. | 0015 permanece como antecedente histórico. | La Decision `DEC-20260818-134600-3jfjak` está `proposed` y enlazada por Core al correctivo; no hay supersesión formal. |
| LiteLLM | No es target, import, dispatcher, ejemplo ni mock activo. | Las referencias archivadas son historia identificada. | Retirada confirmada por registry/docs/tests vigentes. |
| `copilot-vscode` | No es target, tipo, alias ni dispatcher público. `init` lo rechaza como opción inválida. | Antes del correctivo había aceptación pública en `runInit` y dispatch/configure; esa compatibilidad invalidaba el claim previo de retirada. | `configure` sólo migra el literal de `init.json#profileTriple.adapterTarget` a `github-copilot`, lo persiste antes de installer/dispatcher y no reexpone el alias. |
| Scope/freeze | Freeze, receipts y lifecycle resuelven el scope canónico. | Candidatos auxiliares product-owned no son fuente de cierre. | El correctivo aplica Core sobre `Axiom.Spec/specs/increments`. |

## Contratos o estados afectados

No se introducen estados runtime nuevos. Este informe documenta evidencia; las transiciones estructurales pertenecen a Core y las mutaciones de producto a sus incrementos.

## Notas de compatibilidad

La única compatibilidad preservada para `copilot-vscode` es migración de dato persistido ya existente, previa a cualquier dispatcher. No es una API pública. Los artefactos archivados, 0015 y las cifras de validación previas son historia, no evidencia automática del HEAD.
