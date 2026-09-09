# 03 Criterios de Aceptación

- **CA-G1 / ACC-073**: test exhaustivo recorre catálogo/workflows/adapters y demuestra mapping exacto, cero fallback accidental y comandos `axiom ...` parseables.
- **CA-G2 / ACC-073**: create/refine/verify/archive y equivalentes bug/plan/role/E2E requieren ID; ID correcto opera, ausente/mismatch falla sin state/receipt/artifact mutation.
- **CA-G3 / ACC-074**: éxito local + ADO omitido, éxito local + create/link remoto y éxito local + fallo remoto se distinguen; nunca se borra local.
- **CA-G4 / ACC-074**: `https`/`http` válidos son links; `javascript:`, `data:`, file y malformed son texto sin href.
- **CA-G5 / ACC-076**: server/wrapper real aplica token F, identity y local-first con bridge fake hermético.

Evidencia: suites launcher/workflow/lifecycle/ADO/frontend, build/typecheck y diff-check. No se marca ACC-041/045 validada.
