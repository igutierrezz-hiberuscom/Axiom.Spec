# Criterios de aceptacion

- [x] Las superficies CLI/launcher ya no muestran ni aceptan `--profile`,
      `--overlay`, aliases funcionales, `--gateway` o `--no-gateway`.
- [x] Un proyecto nuevo escribe solo `builder` y `local-only` en sus estados
      efectivos; un proyecto antiguo se normaliza al leerlo.
- [x] El composer, installer y schemas tienen una unica politica y conservan
      los diez targets canonicos.
- [x] No hay provider, fallback, discovery profile o resolution mode activo
      para `axiom-gateway` o `generated-snapshots`.
- [x] `gateway-state.json` y el subcomando `axiom gateway` dejan de formar
      parte de la superficie publica.
- [x] Audit trail y `axiom audit` siguen operativos con `P365D` y sin gate
      enterprise.
- [x] `axiom-mcp-broker`, `sdd-mcp-server` y `spec-mcp-broker` no pierden
      handlers ni pruebas.
- [x] Build, doctor, readiness y suites focalizadas pasan, con fallos
      preexistentes clasificados.
- [x] Las busquedas de referencias activas no dejan opciones retiradas en
      runtime/configuracion.
