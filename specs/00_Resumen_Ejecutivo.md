# 00 Resumen Ejecutivo

## Visión

Axiom es una plataforma de Spec-Driven Development (SDD) con un runtime MVP+post-MVP ya operativo. No es un simple reempaquetado de una herramienta existente: es un CLI Node/TypeScript (`axiom`) que, para un proyecto concreto, coordina estructura, configuración, resolución de profiles, materialización de archivos para herramientas externas (IDEs/CLIs de IA), validaciones y checks de salud.

Modelo mental del producto (`Axiom/docs/overview.md`):
- la spec dice qué reglas existen y qué contrato debe cumplirse;
- la configuración (YAML por proyecto) define cómo se activa ese contrato en un proyecto concreto;
- el CLI materializa, valida y opera sobre ese contrato.

## Estado real (verificado en el código, 2026-07-02)

Axiom NO está en fase de diseño: tiene un monorepo npm workspaces (`Axiom/`) con `apps/cli` + 28 packages bajo `packages/*` (ver [context/references/01-inventario-de-packages.md](../context/references/01-inventario-de-packages.md)), 36 ficheros de comandos en `apps/cli/src/commands/` y un histórico de al menos 40 commits (2026-06-05 a 2026-07-02) e incrementos numerados (0008 a 0039+) que fueron cerrando MVP y post-MVP en oleadas.

Según `Axiom/README.md` (2026-06-25/30), todas las capas declaradas están "implementadas y testeadas": dominio, aplicación (CLI), adapters (6 reales), telemetría, orquestación y persistencia.

## Qué hace el producto hoy

El punto de entrada del operador es `axiom` sin subcomando, que abre la TUI (los subcomandos, `--help` y `--version` siguen funcionando igual). En una carpeta sin proyecto Axiom, la TUI muestra un menú de bootstrap cuya acción de inicializar recorre un **wizard guiado multi-repo** que prepara un workspace completo en una sola operación — repo de control (SDD), repo de Spec y N repos de código por rol, cruzados entre sí, registrados y con MCP configurado (`INC-20260705-*`; supersede al antiguo wizard single-repo de `init`) — ver [05_Interfaces_Operativas.md](05_Interfaces_Operativas.md). `axiom init` sigue disponible como comando no-interactivo/scriptable single-repo.

Para un proyecto que adopta Axiom, el ciclo de vida real es:

```
axiom init → axiom join → axiom configure → axiom sync → axiom start → axiom audit → axiom doctor → axiom upgrade
```

Cada comando lee/escribe un conjunto concreto de artefactos (`axiom.yaml`, `.axiom-state/<project>/*.json`, `.axiom-state/local/*`) y, según el profile activo, genera surfaces para el IDE/CLI de destino (`.opencode/`, `.claude/`, `.github/copilot-instructions.md`, `.cursor/`, `.vscode/`, `litellm.config.json`). El detalle completo vive en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y en `context/architecture/`.

## Dos modelos distintos que conviven bajo el nombre "Axiom" (evitar confundirlos)

1. **El producto Axiom** (`Axiom/`): el CLI/runtime descrito arriba, que cualquier proyecto externo puede adoptar. Su unidad de configuración es `axiom.yaml` + estado runtime en `.axiom-state/<projectName>/` dentro del proyecto que lo instala, más un catálogo de ~20 YAML de política/capacidad que el propio Axiom espera encontrar en una carpeta `axiom.config/` (minúsculas) en la raíz del proyecto; el contenido de spec vive aparte en `axiom.spec/`. Ambas carpetas fueron renombradas en `INC-20260703-config-folder-renames` (ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y [08_Glosario.md](08_Glosario.md)).
2. **El workspace de desarrollo de Axiom** (esta raíz: `Axiom/`, `Axiom.SDD/`, `Axiom.Spec/`): el modelo de tres repos desacoplados que `Axiom.SDD/AGENTS.md` establece para construir el propio producto con disciplina SDD ligera (spec en `Axiom.Spec/`, implementación en `Axiom.SDD/`, runtime opcional en `Axiom/`). Este es el "dogfooding" de Axiom sobre sí mismo, y es un concepto de nivel distinto al `axiom.spec/` interno de un proyecto cualquiera.

Ver el glosario ([08_Glosario.md](08_Glosario.md)) para no mezclar `Axiom.Spec/` (este repo) con `axiom.spec/` (carpeta que el producto espera dentro de cualquier proyecto que lo adopte, incluido potencialmente `Axiom/` mismo).

## Auto-validación del producto restaurada (resuelto — `INC-20260708-product-repo-self-bootstrap` + `INC-20260708-fix-longstanding-test-failures`)

Supersede la "discrepancia real conocida" previa: el propio repo `Axiom/` ya se auto-valida. `INC-20260708-product-repo-self-bootstrap` scaffoldeó en su raíz la baseline canónica que su runtime/tests esperaban (`axiom.config/` con contenido schema-válido, `axiom.spec/target-axiom-skills|agents/`, `axiom.spec/templates/`, `AGENTS.md`, `axiom.skills.lock`) y corrigió un `repoRoot` que apuntaba un nivel demasiado alto en `verify-first-project-readiness.mjs`; `npm run readiness:first-project` y `npm run doctor` pasan ahora contra la raíz del propio repo. `INC-20260708-fix-longstanding-test-failures` cerró las 4 pruebas long-standing restantes (todas eran defectos de test — asserts obsoletos, un test no-hermético dependiente del reloj real — no bugs de producto), dejando la suite completa en verde (`178/178` ficheros, `1859/1859` tests en el momento de ese cierre). Ver [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md) y [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md).

## Ola de endurecimiento 2026-07-10 (auditoría + 10 incrementos, cerrada)

Tras una auditoría de premisas contra el código real (build limpio, suite en verde), se corrió una ola de 10 incrementos de reconciliación (`specs/increments/_archive/INC-20260710-*`) que cerró las brechas entre "arquitectura testeada" y "camino feliz operativo". Suite final: **212 ficheros / 2268 tests en verde**. Resumen por área:

- **Schemas duplicados reconciliados** (`schema-reconciliation`): `axiom mcp list/validate/repair` y `axiom toolchain add/show/validate` estaban rotos contra los YAML reales committeados (dos lectores incompatibles del mismo artefacto, ocultado por fixtures). Lectores tolerantes + `axiom.config/toolchain-catalog.yaml` dedicado + tests que cargan los artefactos reales.
- **Dogfooding del propio workflow** (`dogfooding-workflow-configs`): faltaban `axiom.config/workflows.yaml` y `topology.yaml`, así que `axiom-increment create` fallaba en el propio repo. `DEFAULT_WORKFLOWS` bundled con fallback + ambos ficheros materializados + checks de doctor TC-014/TC-015. Reparada también la deriva TC-011.
- **Roles dinámicos** (`dynamic-team-roles`): los roles dejan de ser un catálogo fijo. Registro de roles de equipo de primera clase en `topology.yaml#roles` (`axiom roles register/unregister`, 1..N arbitrarios), desacoplado del eje de perfiles de instalación; el validador de topología acepta los roles registrados y la topología generada por el wizard ya pasa su propio `topology validate`.
- **Planes por rol** (`plan-role-split`): `PlanMetadata.roles` + `axiom-plan create` deriva la separación por rol desde los roles/asignaciones de topología, puebla `targetRepos`/`allowedWriteScope` (ahora `validate changes` sí enforce) y genera un `role-<slug>.md` por rol.
- **Contexto técnico servido por MCP** (`technical-context-served`): las tools MCP devolvían `null` porque el contexto curado (`context/`) no estaba indexado. Generador de `technical-context/indexes/<rol>.index.yml` (`axiom context index`), índice `repo` materializado, y corregido el `inlineContent` (resolución relativa al índice + path-guard). El MCP ahora sirve el contexto real.
- **Paridad de comandos del wizard** (`workspace-command-parity`): todo lo que hacía el wizard TUI es ahora ejecutable por comando — `axiom workspace setup` headless + subcomandos granulares (`spec-base`, `adapters`, `skills`, `rules`, `mcp-config`, `config-scaffold`) para reparar o reutilizar partes de una instalación.
- **Instalación por miembro** (`per-member-install` + `architect-member-handoff`): separación explícita responsabilidad **compartida/committeada** (arquitecto: `axiom.config/*`, roles, MCPs, catálogo de utilidades, spec, contexto, skills) vs **personal/gitignored** (miembro: rutas físicas en `.axiom-state/local/topology-bindings.yaml`, config MCP nativa per-máquina, estado local de utilidades). Comandos `axiom member install` y `axiom bindings`; launch MCP resoluble (`axiom` en PATH o `node <cli>`); `.gitignore` protege `.axiom-state/`. El `workspace setup` scaffoldea las declaraciones committeadas que `member install` consume.
- **Correctitud y honestidad** (`lifecycle-correctness-fixes` + `honesty-and-toolchain-states`): `archive` reubica físicamente la carpeta a `_archive/`; `self-update` lee la versión del paquete instalado; registry v1→v2 auto-migra en `init`/`repo attach`; toolchain con estados diferenciados (`declared`/`marker`/`installed-working` con probe real, ya no "present" falso); código muerto del orchestrator (`axiom intent`) eliminado y README corregido; bugs menores (doble prefijo `gateFailure:`/`upgradeFailed:`, preview dry-run adr/decision).

Ver el detalle por área en `03`/`04`/`05`/`06`/`07` (actualizados por cada incremento) y las brechas resueltas en [../context/references/03-riesgos-y-brechas-conocidas.md](../context/references/03-riesgos-y-brechas-conocidas.md).

## Ola de endurecimiento 2026-07-11 (correctness + affinity + review, cerrada)

Tanda de 3 incrementos implementados y verificados (build + suite en verde; `specs/increments/_archive/INC-20260711-audit-bug-fixes|repo-affinity-guard|per-role-review`): 7 fixes de correctitud/consistencia auditados, un **guard de repo-affinity** (cada comando de ciclo de vida enforced contra el repo correcto en multi-repo) y un **review de write-scope por rol** en `axiom-role complete` más un agregado `axiom validate changes --all-repos` desde el spec. Detalle por área en `03`/`04`/`05`/`06`. En la misma oleada quedaron **totalmente entregados y archivados** el epic `INC-20260711-sdd-launcher-core-port` (re-home del sdd-launcher de KVP25 a Axiom core: P0-P4 + PX + los tres paneles del front, cerrados en `INC-20260711-epic-close-panels`) y el **adopter de migración de repos foráneos** (`from-legacy-sdd` format-aware + ingesta de contexto + UX de adopción); ver `05`/`06`.

## Roadmap de rediseño (cerrado, parcialmente implementado)

El incremento de planificación `specs/increments/INC-20260702-axiom-redesign-roadmap/` secuenció un rediseño (separación de repos por rol `sdd`/`spec`/`code`, registro global `~/.axiom/projects.yml`, manifiestos `axiom.yml` por repo, índices derivados/curados, MCP único por proyecto con `get_implementation_context`, TUI contextual y sistema de versionado/migraciones) en 24 incrementos (INC-01 a INC-24). A fecha 2026-07-03, los 23 incrementos de infraestructura (Fases A-G más INC-23, dogfooding) están **cerrados** — cada uno auditó primero el paquete `Axiom/packages/*` existente (modelo de reconciliación, no greenfield) antes de extenderlo. INC-24 (Workbench) permanece explícitamente diferido, sin empezar. El detalle de qué quedó resuelto, qué quedó pendiente y el registro de las 5 preguntas de arquitectura (Q1-Q5) vive repartido en las secciones correspondientes de `01` a `08` de esta carpeta (topología y registro en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md), modelo de artefactos y bootstrap en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md) y [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md), capa MCP en [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md), gobierno en [07_Gobierno_y_Seguridad.md](07_Gobierno_y_Seguridad.md)). El histórico completo de las 71 carpetas de incremento de esta cadena vive archivado en `specs/increments/_archive/`; el índice/resumen de cierre del roadmap sigue en `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md`. No confundir el modelo de `axiom.yaml schemaVersion: 1` (vigente por defecto, ver "Dos modelos distintos" arriba) con el `schemaVersion: 2` opt-in que este roadmap añadió de forma aditiva.

## Principios del producto

1. Spec first, implementación second.
2. Separación clara entre builder tooling y product runtime.
3. Axiom model primero; generated config después.
4. La trazabilidad enterprise es obligatoria (audit trail, telemetry sinks).
5. Deben soportarse los modos `local-only`, `standard` y `enterprise` (overlays operacionales).
6. Las capabilities están guiadas por profiles (`functionalProfile` + `operationalOverlay` + `adapterTarget`).
7. Las integraciones de tooling son adapters opcionales, con niveles de soporte explícitos (`multi-mode`, `single-mode`, `fallback-only`).
8. Minimizar el gasto de tokens sin reducir la auditabilidad.
9. Mantener una arquitectura modular y reemplazable (`Result<T,E>` sin excepciones, escritura atómica, path-guard por proyecto).
10. Evitar vendor lock-in.

## Roadmap y estado operativo

No existe hoy `plans/PLAN-PRODUCT-ROADMAP.md` en `Axiom.Spec/plans/` (la carpeta solo tiene un `README.md` de propósito, sin contenido de roadmap). El estado operativo consolidado y verificable vive en:
- `Axiom/README.md` (tabla de capas y paquetes operativos);
- `Axiom/docs/0015-…` a `0030-…` (increments post-MVP cerrados);
- `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md` (roadmap de rediseño, cerrado — ver sección anterior);
- `specs/increments/_archive/` (histórico de las 71 carpetas de incremento del roadmap de rediseño).