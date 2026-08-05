# Criterios de aceptación

- [x] El inventario de las cinco familias de `Axiom/axiom.spec/` está registrado.
- [x] Los consumidores y la topología `specRepo.ref: ../Axiom.Spec` están verificados.
- [x] ADR-0032 decide conservar `Axiom/axiom.spec/` sin moverla, borrarla ni renombrarla.
- [x] Las specs y el contexto explican la frontera sin claims activos contradictorios.
- [x] No se modificó comportamiento de runtime ni la baseline generada; `npm run build` pasa.
- [x] La salida dedicada de build, doctor y readiness quedó persistida con sus códigos de salida.
- [x] La revisión independiente y sus hallazgos quedan registrados.
- [x] Existe un freeze válido inmediatamente anterior al apply gobernado de cierre.
- [x] El apply gobernado posterior al freeze fue ejecutado y verificado como idempotente.
- [x] El incremento queda integrado y archivado sin comandos Git mutantes.
