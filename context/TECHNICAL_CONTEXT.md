# TECHNICAL_CONTEXT

Puerta de entrada obligatoria al conocimiento técnico estable del producto Axiom, tal como existe en el código real de `Axiom/`. Baseline original: 2026-07-02; reconciliado el 2026-07-29 (ver "Fuentes auditadas y última validación" abajo). Cada documento enlazado aquí cita su fuente (código, README de package, o `Axiom/docs/`) y distingue explícitamente entre estado verificado y planificación futura.

## Propósito del contexto técnico

Este contexto explica CÓMO está construido el producto Axiom hoy: su arquitectura por capas, su modelo de datos, su ciclo de vida operativo, sus integraciones y sus riesgos conocidos. No es un resumen de todo el código ni documentación ficticia de módulos no implementados — cada afirmación aquí es trazable a un fichero concreto del repo `Axiom/` o de sus docs oficiales.

Para QUÉ debe hacer el producto (requisitos, alcance, principios), ver `specs/00_Resumen_Ejecutivo.md` a `specs/08_Glosario.md`. Este contexto técnico es el complemento de "cómo está hecho" a esa spec de "qué debe hacer".

## Estado real del producto (resumen)

- Monorepo npm workspaces en `Axiom/`: `apps/cli` + **43 packages** bajo `packages/*` — **34 top-level** (verificado: `ls packages/*/package.json | wc -l` → 34; `packages/adapters/` en sí mismo NO tiene `package.json` propio, es solo un directorio contenedor) + **9 sub-packages de `packages/adapters/*`** (verificado: `ls packages/adapters/*/package.json | wc -l` → 9: `opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `litellm`, `codex`, `antigravity`, `visual-studio-2026`). Packages nuevos desde el baseline 2026-07-02 (no estaban en el inventario de 28): `launcher`, `mcp-server`, `mcp-tools`, `providers`, `technical-context`, `telemetry`, `tracker`, `tracker-ado`.
- Runtime MVP cerrado 2026-06-25; oleadas post-MVP (0019-0039) cerradas hasta 2026-07-01/02; evolución posterior continua (renames de carpetas de config, dedup de `init.json`/`topology.yaml`, wizard TUI de `init`, launcher como front por defecto, scaffolding de adopción, paridad de 9 adapters, servidores MCP propios, `cmm` reemplaza `codegraph`/`graphify` — ver `references/02-historial-de-incrementos.md`).
- CLI real con **81 ficheros** en `apps/cli/src/commands/*.ts` (verificado: `ls apps/cli/src/commands/*.ts | wc -l` → 81; de esos, 10 son helpers compartidos con prefijo `_` — p. ej. `_shared.ts`, `_tracker-status.ts` — no comandos por sí mismos), de los cuales solo una minoría tiene página de documentación operativa dedicada en `Axiom/docs/cli/`. Familias nuevas relevantes: `workspace*` (16 ficheros, incl. `workspace-setup.ts`/`workspace-adopt.ts`), `app*`/`app-launcher*` (front operador = launcher), `member-install.ts`, `native-mcp-config.ts`, `tracker`-backed commands (`_tracker-status.ts`, `app-launcher-ado.ts`, `external-sync.ts`).
- Renombres de carpetas de estado/config ya cerrados y verificados en código: `.sdd/` → **`.axiom-state/`** y `axiom.spec/config/` → **`axiom.config/`** (`packages/filesystem-truth/src/discovery.ts#LOCAL_OVERLAY_DIRNAME`/`AXIOM_CONFIG_DIRNAME`, re-exportado vía `@axiom/core`). Cualquier mención a `.sdd/` o `axiom.spec/config/` como ruta actual en este contexto es un residuo del baseline 2026-07-02 y fue corregida en la reconciliación 2026-07-29.
- Axiom ya no carece de servidor MCP propio: existen `@axiom/mcp-server` (dispatcher JSON-RPC 2.0 hand-rolled, sin SDK) y `@axiom/mcp-tools` (handlers), proyectados a config nativa de cada tool por `apps/cli/src/commands/native-mcp-config.ts` (servers gestionados `sdd-mcp-server` / `spec-mcp-broker`). Ver `integrations/01-capabilities-providers-y-toolchain.md`.
- El repo `Axiom/` ya se auto-adoptó (`INC-20260708-product-repo-self-bootstrap` + reconciliaciones posteriores): `axiom.spec/`, `AGENTS.md`, `axiom.skills.lock` y `axiom.config/` existen hoy en su raíz (falta solo `_builder/`). Ver `references/03-riesgos-y-brechas-conocidas.md`.

## Estructura de `context/`

- `architecture/` — cómo está construido el runtime: capas, modelo de datos/configuración, ciclo de vida CLI/orquestación, adapters y model routing.
- `operations/` — cómo se instala, onboarda, diagnostica y audita un proyecto real.
- `integrations/` — capabilities, providers, toolchain externo (Serena, cmm, etc.) y servidores/bridges MCP.
- `references/` — material de consulta puntual: inventario de los 43 packages, historial de incrementos 0015-0030 + resumen de oleadas posteriores, y riesgos/brechas conocidas verificadas.

### Índice de documentos

1. [architecture/01-vision-general-y-capas.md](architecture/01-vision-general-y-capas.md)
2. [architecture/02-modelo-de-datos-y-configuracion.md](architecture/02-modelo-de-datos-y-configuracion.md)
3. [architecture/03-ciclo-de-vida-cli-y-orquestacion.md](architecture/03-ciclo-de-vida-cli-y-orquestacion.md)
4. [architecture/04-adapters-y-model-routing.md](architecture/04-adapters-y-model-routing.md)
5. [operations/01-instalacion-y-onboarding.md](operations/01-instalacion-y-onboarding.md)
6. [operations/02-doctor-troubleshooting-y-telemetria.md](operations/02-doctor-troubleshooting-y-telemetria.md)
7. [integrations/01-capabilities-providers-y-toolchain.md](integrations/01-capabilities-providers-y-toolchain.md)
8. [references/01-inventario-de-packages.md](references/01-inventario-de-packages.md)
9. [references/02-historial-de-incrementos.md](references/02-historial-de-incrementos.md)
10. [references/03-riesgos-y-brechas-conocidas.md](references/03-riesgos-y-brechas-conocidas.md)

## Decisiones arquitectónicas vigentes

- `Result<T, E>` sin excepciones en todo el dominio (`@axiom/core`).
- Escritura atómica (tmp+rename) e idempotencia byte a byte en toda superficie generada.
- Aislamiento project-scoped obligatorio (`@axiom/isolation`): ninguna ruta/cache/binding cruza `projectId`.
- YAML declarativo como fuente de configuración (profiles, capabilities, providers, telemetry, policy), separado por responsabilidad.
- GATEs numeradas y verificables por `@axiom/doctor`, no solo documentadas.
- Ver detalle y enlaces a incrementos concretos en `references/02-historial-de-incrementos.md`.

## Convenciones y reglas operativas

- No promover como estado actual nada que solo esté en el roadmap de rediseño, hoy **cerrado y archivado** en `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/` (23/24 incrementos ejecutados; INC-24 Workbench diferido). Su conocimiento estable vive consolidado en `specs/00_Resumen_Ejecutivo.md` a `specs/08_Glosario.md`, no en este contexto técnico.
- No confundir `Axiom.Spec/` (este repo) con `axiom.spec/` (carpeta interna que el producto espera dentro de un proyecto adoptante). Ver glosario en `specs/08_Glosario.md`.
- Cada documento de este contexto debe citar su fuente (path de código, README de package o doc oficial) para cada afirmación no trivial.
- Al añadir un documento nuevo bajo `context/`, actualizar el índice de esta página.

## Riesgos, excepciones y límites conocidos

Ver detalle completo en [references/03-riesgos-y-brechas-conocidas.md](references/03-riesgos-y-brechas-conocidas.md). Resumen (post-reconciliación 2026-07-29):

1. **RESUELTA** — `axiom.spec/`, `AGENTS.md`, `axiom.skills.lock` y `axiom.config/` ya existen en la raíz de `Axiom/` (falta solo `_builder/`, gap menor preexistente).
2. Persiste, a mayor escala: la mayoría de los ~81 ficheros de comando CLI reales no tienen documentación operativa dedicada propia en `Axiom/docs/cli/`.
3. Los packages sin `README.md` (mayoría de los 43) siguen caracterizados a partir de su estructura de `src/`, no de documentación propia — tratar como inferencia razonable, no como contrato firmado.
4. **RESUELTA** — el roadmap de rediseño de topología de repos (INC-01 a INC-24) cerró y está archivado en `specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/`.

## Fuentes auditadas y última validación

- Fuentes (baseline 2026-07-02): `Axiom/README.md`, `Axiom/package.json`, `Axiom/docs/**/*.md` (40+ ficheros), `Axiom/packages/**/README.md`, estructura de `Axiom/packages/**/src`, `Axiom/apps/cli/src/commands/*.ts` (listado), `Axiom/scripts/verify-first-project-readiness.mjs`, `Axiom.SDD/AGENTS.md`, `git log` de `Axiom/`.
- Pase de reconciliación 2026-07-29 (`INC-20260729-technical-context-reconciliation`): re-verificó contra código real conteos de packages/comandos/adapters, los renames `.axiom-state`/`axiom.config`, el dedup de `init.json`/`topology.yaml`, el wizard TUI de `init`, el launcher como front por defecto, el scaffolding de adopción, los servidores MCP propios (`mcp-server`/`mcp-tools`), el reemplazo `codegraph`/`graphify`→`cmm` (ADR-0031), y el estado ampliado de `@axiom/doctor`. Cubrió `architecture/01-03`, `operations/01-02`, `integrations/01`, `references/01-03` (`architecture/04` ya estaba vigente desde 2026-07-26 y solo se revisó, no se reescribió).
- Última validación: 2026-07-29.
- Responsable humano: pendiente de asignar por el equipo (no declarado en el momento de esta redacción).

## Regla crítica

Este contexto técnico no debe convertirse en un resumen de código ni en documentación ficticia de módulos no implementados. Toda afirmación sobre "estado actual" debe ser verificable releyendo el código o los docs citados; toda afirmación sobre planes futuros debe marcarse explícitamente como tal.
