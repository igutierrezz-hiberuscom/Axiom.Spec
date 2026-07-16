# 04. Generar spec y contexto técnico

Cómo arrancar la spec canónica y el contexto técnico de un proyecto — desde cero o adoptando código/documentación ya existentes.

## Escenario 1: proyecto nuevo (scaffold desde plantillas)

Cuando el setup de workspace (`axiom workspace setup`, o el wizard guiado de la TUI) crea desde cero el repo de Spec, lo scaffoldea automáticamente desde la base canónica de plantillas: `specs/README.md` + `specs/00..08` + `context/TECHNICAL_CONTEXT.md`/`README.md` + los directorios estructurales (`specs/{increments,bugs,archive}`, `context/*`). Es best-effort y guardado per-file — nunca sobreescribe contenido ya existente en una corrida posterior. Así, un workspace nuevo arranca ya con la estructura numerada de spec en vez de una carpeta vacía.

## Escenario 2: adoptar código existente (bootstrap)

Para un repo de código que YA existe y aún no tiene spec propia:

```bash
axiom bootstrap from-code --level minimal|basic [--role <role>]
```

Analiza el repo activo (detección de stacks, mapa de repo, detección de comandos disponibles) y redacta documentos de contexto técnico marcados con el banner `<!-- AXIOM:DRAFT -->`, poblando `TechnicalContextIndex.available` (nunca `mandatory.*`, que queda reservado a curación humana). El nivel "Standard" en adelante (comprensión arquitectónica/de negocio genuina) queda diferido.

## Escenario 3: adoptar un repo de spec legado

Para un repo que ya tenía SU PROPIA disciplina de spec/incrementos (Axiom-shaped o no):

```bash
axiom bootstrap from-legacy-sdd <path> [--dry-run]
```

Es format-aware: además de carpetas ya shaped como Axiom, detecta formatos de spec foráneos (`openspec`, `docs-adr`, `generic-folders`) vía un registro de detectores enchufable, y convierte cada entrada a la plantilla canónica de incremento, marcada con el banner `<!-- AXIOM:MIGRATED -->`. Nunca sobreescribe ni aborta el lote entero ante una colisión puntual; `--dry-run` lista el/los formato(s) detectado(s) sin escribir nada.

## Escenario 4: ingerir contexto técnico ya escrito (no-SDD)

Para incorporar documentación existente que no sigue ningún formato de incremento (README arquitectónicos, ADRs sueltos, `docs/**`):

```bash
axiom bootstrap from-context <path>
```

Ingiere ese contexto a `technical-context/*` con un `TechnicalContextIndex` en estado `draft`, consultable vía la tool MCP `spec.technicalContextIndexRead`. Cada documento ingerido lleva el banner `AXIOM:MIGRATED` (distinto del `AXIOM:DRAFT` de `from-code`). Re-correrlo no duplica contenido.

## Adopción combinada en install-time

`axiom workspace setup --adopt-spec --adopt-sdd --ingest-context [--dry-run] [-y]` orquesta las tres rutas anteriores en un solo paso al instalar sobre un proyecto preexistente: conformidad del repo de control, migración del repo de spec foráneo, e ingesta de contexto — con un preview dry-run por defecto que exige confirmación antes de escribir (`-y` la salta).

## Indexado de contexto técnico

`axiom context status` muestra el estado del proyecto activo (nombre, rootPath, profile triple, capabilities) y, de forma best-effort, un bloque de "lecciones recientes" (aprendizaje continuo) y sugerencias de delegación. `axiom context refresh` re-deriva el `ResolvedInstallProfile` y lo persiste. Todos los artefactos adoptados (código o spec legada) aterrizan en estado seguro (`draft`/`proposed`) y requieren revisión humana como cualquier artefacto nuevo — ver [09_Revisiones.md](09_Revisiones.md).

## Relacionado

- [01_Que_Es_Cada_Cosa.md](01_Que_Es_Cada_Cosa.md)
- [05_Incrementos.md](05_Incrementos.md)
- [09_Revisiones.md](09_Revisiones.md)
- [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md)
