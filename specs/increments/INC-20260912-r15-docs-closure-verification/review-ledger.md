# Review ledger

| ID | Tipo | Superficie | Severidad | Estado | Evidencia |
|---|---|---|---|---|---|
| REVIEW-088-001 | arquitectura | `@axiom/workflow` runner único | BLOCKER | resuelto | `runGovernedTransition` concentra evaluación documental, legalidad, confirmación, persistencia, archive y receipts; CLI, launcher y MCP lo consumen sin una ruta alternativa. |
| REVIEW-088-002 | alcance | `documentation-review.ts` | BLOCKER | resuelto | D-02 deriva documentos del artefacto, plan asociado y paths documentales modificados; conserva rutas anidadas project-relative y declara alcance vacío. |
| REVIEW-088-003 | superficie pública | CLI increment/bug/integrate | BLOCKER | resuelto | `--documentation-review <json>` usa el parser común; `--json` incluye la decisión documental; `--documentation-mode` conserva warning/block. |
| REVIEW-088-004 | superficie pública | launcher archive | BLOCKER | resuelto | El catálogo expone `documentationMode` y `documentationReview`; preview y ejecución validan y transportan ambos campos a `runIntegrate`. |
| REVIEW-088-005 | auditoría | receipts e integrate | WARNING | resuelto | Receipts de éxito y rechazo incluyen `documentationReview`; `integrate` devuelve la decisión también en éxito. |
| REVIEW-088-006 | validación | workflow/CLI/MCP/launcher | WARNING | resuelto | Ejecución focal conjunta final: 6 archivos y 155/155 tests PASS; cobertura documental 5/5, typecheck y build PASS. |
| REVIEW-088-007 | re-review | alcance completo de ACC-088 | resuelto | Re-review independiente N=1: OK; AC-088-01..08 y AC-GEN-01 satisfechos, sin blockers residuales. |
| REVIEW-088-008 | alcance automático | `documentation-review.ts` | BLOCKER | resuelto | Se eliminó el barrido global de `git status`; el alcance automático se limita al artefacto y plan asociado, y los paths externos requieren `affectedPaths` explícitos. Runner: 34/34 PASS. |
| REVIEW-088-009 | re-review | alcance causal y superficies públicas | resuelto | Re-review independiente N=1: OK; AC-088-01..08 y AC-GEN-01 satisfechos, sin blockers residuales. |

La primera review encontró blockers reales de contrato, no solo wording:
alcance automático insuficiente y ausencia de entrada estructurada en CLI y
launcher. Se repararon antes de la re-review; no se modificó contenido
documental automáticamente ni se abrió un segundo camino de archive. La
revisión final debe confirmar que el alcance acotado no incorpora históricos ni
documentación ajena.