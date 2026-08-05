# Criterios de aceptación

- [x] `Axiom.Spec/decisions/` contiene los ocho ADR nombrados.
- [x] Cada destino es byte-equivalente a su origen histórico y los ocho orígenes activos fueron retirados de `Axiom/docs/`.
- [x] Las referencias activas usan las rutas canónicas y no queda un claim vigente de `general-spec.md`.
- [x] `AGENTS.md`, `Axiom.SDD/AGENTS.md` y `Axiom.Spec/README.md` describen el ownership verificado.
- [x] No se modificó comportamiento de runtime; `npm run build` pasa.
- [x] La salida dedicada de build, doctor y readiness quedó persistida con sus códigos de salida.
- [x] Existe un freeze válido inmediatamente anterior al apply gobernado de cierre.
- [x] El apply gobernado posterior al freeze fue ejecutado y verificado como idempotente.
- [x] La revisión independiente y sus hallazgos quedan registrados.
- [x] La integración final actualiza las specs y el contexto aplicables.
- [x] El incremento queda integrado y archivado sin comandos Git mutantes.
