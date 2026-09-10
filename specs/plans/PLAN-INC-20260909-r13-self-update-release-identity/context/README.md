# Context

## Propósito

Esta carpeta contiene, solo cuando resulte necesario, evidencia operativa pequeña para ejecutar y revisar `PLAN-INC-20260909-r13-self-update-release-identity`. Su objetivo es evitar que cada handoff tenga que redescubrir rutas/símbolos confirmados o las invariantes de no mutación, sin convertir el plan en un almacén de logs ni duplicar la especificación canónica.

En el estado documental actual no hay evidencia de implementación: el plan está draft y el incremento permanece planificado/pending. La fase 0 decide si hace falta crear alguno de los archivos sugeridos.

## Qué puede vivir aquí

- `runtime-symbol-map.md`: commit/base observado, rutas y símbolos reales para identidad, build, ManagedState, Git, cache y CLI, incluyendo cómo se verificaron.
- `validation-matrix.md`: resumen R13-AC → test/comando/evidencia y clasificación de fallos preexistentes.
- `git-no-mutation-evidence.md`: allowlist de comandos y resumen saneado de snapshots antes/después.
- `handoff-notes.md`: bloqueos, decisiones técnicas ya aprobadas y riesgos residuales para reviewer/owner.

Cada nota debe ser breve, fechada, saneada y distinguir hechos observados de propuestas. La fuente principal sigue siendo el incremento; una nota no puede cambiar sus requisitos.

## Qué no debe vivir aquí

- Código, fixtures Git, cachés, binarios, node_modules o outputs de build.
- Logs completos de Vitest/build, dumps de procesos o copies de repositorios temporales.
- Metadata/status/índices/receipts o instrucciones para evitar Core.
- Requisitos, modelo o criterios alternativos que compitan con el incremento.
- Credenciales, tokens, URLs sin sanear, rutas personales o contenido del HOME real.
- Diseño/implementación de download/apply, manifest v2 completo, Launcher o CI final.
- Afirmaciones de GO sin comandos, resultados y revisión semántica trazables.

## Estructura sugerida

```text
context/
├── README.md
├── runtime-symbol-map.md          # opcional, producido tras G0
├── validation-matrix.md           # opcional, completado para G7
├── git-no-mutation-evidence.md    # opcional, resumen de pruebas herméticas
└── handoff-notes.md               # opcional, solo si existe un handoff real
```

No se crean archivos vacíos para satisfacer la estructura. La evidencia voluminosa permanece en CI/test reports del repositorio de código y se referencia por commit/job. El owner decide la integración estable posterior; cualquier operación estructural o transición se realiza mediante Axiom/Core, nunca editando manualmente metadata o status.
