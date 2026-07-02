# Riesgos y brechas conocidas (verificadas)

Este documento existe para que la spec general no maquille el estado real. Cada punto fue verificado directamente contra el filesystem del repo, no inferido de la documentación.

## 1. `axiom.spec/`, `AGENTS.md` y `axiom.skills.lock` faltan en la raíz de `Axiom/`

`Axiom/README.md`, `Axiom/docs/first-project-readiness.md`, `Axiom/docs/cli/*.md` y `Axiom/scripts/verify-first-project-readiness.mjs` (función `seedCanonicalBaseline`) asumen que la raíz del propio repo `Axiom/` contiene:

- `axiom.spec/config/` (~20 YAML de política/capacidad);
- `axiom.spec/templates/`;
- `axiom.spec/target-axiom-skills/`;
- `axiom.spec/target-axiom-agents/`;
- `AGENTS.md`;
- `axiom.skills.lock`;
- `_builder/`.

**Verificado por listado directo de `Axiom/` (raíz)**: ninguno de esos paths existe. Solo existen `.codegraph`, `.git`, `apps`, `docs`, `packages`, `scripts`.

**Consecuencia real**: `npm run readiness:first-project` fallaría hoy con `ENOENT` al copiar `axiom.spec/config` (y el resto) hacia el proyecto temporal. El checklist manual de `first-project-readiness.md` (paso 3, `node ../axiom.spec/scripts/doctor-validate-contracts.mjs`) también fallaría, porque ni `axiom.spec/` existe como sibling de `Axiom/` en este workspace.

**Hipótesis razonable, no confirmada**: el `git log` de `Axiom/` muestra un commit `"Remove obsolete templates and example files from the Axiom product specification, including increment, plan, and UI interaction templates..."` — es plausible que esa carpeta existiera antes y fue removida en una limpieza, sin actualizar en consecuencia los scripts/docs que la referencian. No se confirmó el commit exacto que la eliminó; no inventar esa causalidad como hecho verificado, solo como hipótesis a investigar.

**Impacto en esta spec**: cualquier afirmación sobre el "contrato de `axiom.spec/config/*.yaml`" (ver `../architecture/02-modelo-de-datos-y-configuracion.md`) describe lo que el producto ESPERA de un proyecto adoptante en general — no se pudo verificar contra una instancia real materializada en este workspace, porque ni el propio `Axiom/` la tiene.

## 2. Brecha de documentación operativa de comandos CLI

`apps/cli/src/commands/` contiene 36 ficheros (~16.400 líneas). `Axiom/docs/cli/` documenta en profundidad 12: `init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `tui`, `model`, `components`, `skills`.

Sin página propia: `app`, `app-api`, `app-plugins`, `app-plugins-azure-devops`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`. Varios de estos se pueden reconstruir parcialmente desde los documentos de incremento 0019-0030 y desde `Axiom/README.md`, pero no tienen el mismo nivel de detalle (flags, contrato de lectura/escritura) que los 12 documentados.

**Recomendación operativa**: antes de citar el comportamiento de cualquiera de estos 24 comandos como contrato estable en un incremento nuevo, verificar directamente en el código (`apps/cli/src/commands/<comando>.ts`), no asumir paridad con los documentados.

## 3. Inferencia de responsabilidad en packages sin README

De los 28 packages bajo `packages/`, solo `core`, `persistence`, `installer`, `orchestrator`, `agents`, `install-profiles`, `isolation`, `cli-commands`, `doctor` (9 de 28, aproximadamente) tienen README propio consultado. El resto (`capability-model`, `config-validation`, `filesystem-truth`, `project-resolution`, `memory`, `versioning`, `toolchain`, `topology`, `workflow`, `model-routing`, `tool-routing`, `skills`, `components`, `tui`, `document-bootstrap`, `cavekit-discipline`, `user-workspace`) fue caracterizado a partir de nombres de ficheros en `src/` y su `package.json` — es una inferencia razonable de un agente de exploración, no una lectura de documentación propia del package.

**Impacto**: tratar las descripciones de esos packages en `../references/01-inventario-de-packages.md` como buena aproximación, sujeta a corrección si se audita el código fuente completo.

## 4. Roadmap de rediseño sin fecha de inicio ni stack decidido

`specs/increments/INC-20260702-axiom-redesign-roadmap/README.md` (fechado el mismo día que esta spec, 2026-07-02) es planificación pura: 24 incrementos secuenciados (INC-01 a INC-24), sin código implementado. Dos preguntas quedan explícitamente abiertas y bloqueantes antes de que INC-01 pueda arrancar:

- **Q1**: si `Axiom.SDD`/`Axiom.Spec` deben reestructurarse hacia el shape `project.sdd`/`project.spec` (con `.axiom/` manifests, `technical-context/` bajo spec) que el propio roadmap describe.
- **Q2**: stack de implementación objetivo para Axiom Core/CLI/TUI/MCP (Node/TS ya es el stack real hoy verificado en `Axiom/`, pero el roadmap no lo da por cerrado formalmente para la fase de rediseño).

Ese roadmap describe una topología (`~/.axiom/projects.yml`, `axiom.yml` por repo con roles `sdd`/`spec`/`code`) que es **incompatible en nombre y forma** con el modelo real hoy implementado (`axiom.yaml` único + `.sdd/<project>/` + profile triple). No fusionar ambos modelos al leer `specs/03_Modelo_Operativo_y_Datos.md`.

## 5. Ambigüedad de nombres `Axiom.Spec/` vs `axiom.spec/`

Ver `specs/08_Glosario.md`, sección "Aviso de ambigüedad de nombres". Es una fuente real de confusión al leer los docs de `Axiom/` (que usan `axiom.spec/` en minúsculas para su propio catálogo de configuración interna) junto a este repo de workspace (`Axiom.Spec/`, en mayúsculas, con una estructura completamente distinta: `context/`, `specs/`, `templates/`, `plans/`, `prompts/`).

## Cómo mantener este documento

Al cerrar cada incremento nuevo en `Axiom.SDD`, si se resuelve alguno de estos puntos (p. ej. se crea `axiom.spec/` en la raíz de `Axiom/`, o se documenta un comando antes huérfano), actualizar esta lista quitando o marcando como resuelto el punto correspondiente, con fecha y referencia al incremento que lo cerró.
