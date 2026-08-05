# Criterios de aceptación

- [x] Install/join/setup desde launcher tienen preview y confirmacion.
- [x] Adopcion de spec, SDD/control y contexto se puede iniciar desde la UI.
- [x] Preview y dry-run no producen writes.
- [x] Confirmacion delega en primitives canonicos y respeta no-clobber,
      provenance, colisiones e idempotencia.
- [x] Single-repo, multi-repo y proyectos existentes quedan probados.
- [x] Registry, doctor, rutas y warnings se reflejan desde estado real.
- [x] Freeze, receipts, review, validacion ejecutable e integracion canonica
    quedan documentados.

### Evidencia

- Prueba HTTP nueva: `apps/cli/tests/launcher-onboarding-migration.test.ts`
    cubre preview, dry-run, paths absolutos y solapados, destino foráneo,
    multi-repo, fuentes legacy read-only, registry, provenance y rerun
    idempotente.
- Suite focalizada final: 7 archivos, 90 tests, todos verdes.
- Build: `npm run build`, verde.
- Freeze pre-apply histórico: hash
    `45150e80579291001560fd8f56c343fc9a0388f9262093a50f613015a2729504`.
    El cierre usa un freeze final y un receipt `verify` emitidos sobre esta
    versión documental, no ese hash anterior.
