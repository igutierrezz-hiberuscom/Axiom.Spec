# Review ledger

| ID | Tipo | Superficie | Estado | Nota |
|----|------|-----------|--------|------|
| REVIEW-090-001 | comparación | 45 plantillas runtime vs Spec | resuelto | 38 idénticas; siete divergentes documentadas en README; gana runtime por contenido vigente |
| REVIEW-090-002 | test | workflow golden | resuelto | Comparación completa de 13 plantillas bundleadas y prueba temporal de divergencia |
| REVIEW-090-003 | procedencia | workflow y adapter templates | resuelto | Comentarios activos ya declaran `Axiom/axiom.spec/templates/` |
| REVIEW-090-004 | retirada | `Axiom.Spec/templates/` | resuelto | Directorio y 45 archivos retirados tras el rescate |
| REVIEW-090-005 | gates | build, adapters, doctor, readiness, diff check | resuelto | 97/97 tests focales, typecheck, build, doctor PASS, readiness PASS y diff check PASS |
| REVIEW-090-006 | procedencia | referencias activas y `apps/cli/dist` | resuelto | El barrido sensible a mayúsculas no encuentra `Axiom.Spec/templates` en fuentes activas; se retiraron cuatro salidas dist ignoradas stale |
| REVIEW-090-007 | comparación | 45 plantillas runtime vs HEAD histórico | resuelto | 45/45 nombres presentes y exactamente 7 divergencias, todas registradas en README |
| REVIEW-090-008 | procedencia | `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`, `04_Flujos_SDD_y_Ciclo_de_Vida.md` | resuelto | Los claims activos fueron reescritos con la fuente runtime vigente; la comprobación sensible a mayúsculas pasa sin referencias retiradas en ambas specs |
| REVIEW-090-009 | re-review | alcance completo de ACC-090 | resuelto | Revisión independiente N=1: OK; AC-090-01..07 y AC-GEN-01 satisfechos, sin blockers residuales |
| REVIEW-090-010 | validación global | `npm test` | advertencia | La suite global no concluyó por repetición en `workspace-step-reconciliation.test.ts`; las pruebas focales, typecheck, build, doctor, readiness y diff check sí pasan |

La revisión independiente inicial encontró REVIEW-090-008 como blocker. Se
corrigió antes del cierre y la comprobación focal posterior confirmó que ambos
claims canónicos ya distinguen historia de contrato vigente.