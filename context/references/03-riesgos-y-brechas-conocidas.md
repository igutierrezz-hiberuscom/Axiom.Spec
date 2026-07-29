# Riesgos y brechas conocidas (verificadas)

Este documento existe para que la spec general no maquille el estado real. Cada punto fue verificado directamente contra el filesystem del repo, no inferido de la documentación.

## Estado de resolución (actualizado 2026-07-29)

Varias brechas de este documento (redactado el 2026-07-02) ya están resueltas; las que siguen abiertas se han vuelto a medir contra el árbol real:

- **Brecha 1 — RESUELTA** (`INC-20260708-product-repo-self-bootstrap`): `Axiom/axiom.spec/`, `AGENTS.md`, `axiom.skills.lock` y `axiom.config/` (con sus YAML canónicos) existen hoy en la raíz de `Axiom/`; `_builder/` sigue ausente, pero el script de readiness lo crea vacío en el proyecto temporal.
- **Brecha 4 — RESUELTA**: el roadmap de rediseño quedó cerrado y archivado (23/24 incrementos ejecutados; INC-24 Workbench sigue diferido).
- **Ola 2026-07-10 (10 incrementos)** — resolvió además: drift de schemas en `mcp-manifest.yaml`/`integrations.yaml` (CLI `mcp`/`toolchain` ya funcionan contra los artefactos reales; con tests que los cargan de verdad, no fixtures); ausencia de `workflows.yaml`/`topology.yaml` en el propio repo (dogfooding); roles fijos → registro dinámico de roles de equipo (1..N) con validador reconciliado; planes sin separación por rol; contexto técnico que el MCP servía como `null` (ahora indexado y servido); paridad de comandos del wizard TUI; separación arquitecto↔miembro (compartido/committeado vs personal/gitignored) con `member install`/`bindings`; y correctitud (`archive` mueve carpeta, `self-update`, estados reales de toolchain, código muerto del orchestrator). Ver [../../specs/00_Resumen_Ejecutivo.md](../../specs/00_Resumen_Ejecutivo.md) §"Ola de endurecimiento 2026-07-10".
- **Brecha 2 — VIGENTE**: hay 81 ficheros de comando CLI y solo una minoría tiene página operativa dedicada.
- **Brecha 3 — VIGENTE**: la mayoría de los 43 packages no tiene README propio y su descripción requiere contrastar `src/` y `package.json`.
- **Brecha 5 — VIGENTE**: persiste la ambigüedad de nombres `Axiom.Spec/` vs `axiom.spec/`.

Las secciones siguientes describen el estado actual y conservan el origen del
baseline solo cuando ayuda a explicar la brecha. No deben citarse como
contratos actuales sin volver a comprobar las fuentes indicadas.

## 1. Artefactos de la raíz de `Axiom/` (resuelto; baseline histórico)

La observación original quedó resuelta: el listado actual de `Axiom/` contiene
`axiom.spec/`, `axiom.config/`, `AGENTS.md` y `axiom.skills.lock`. La carpeta
`axiom.spec/` contiene `increments/`, `plans/`, `target-axiom-agents/`,
`target-axiom-skills/` y `templates/`; no contiene `scripts/`, y `_builder/`
continúa siendo el único hueco menor de la readiness. La fuente de la
configuración runtime actual es `axiom.config/`, no `axiom.spec/config/`.

### Registro original del baseline 2026-07-02

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

**Resolución actual**: las afirmaciones históricas sobre el contrato de
`axiom.spec/config/*.yaml` describían lo que el producto esperaba de un
proyecto adoptante en general. La instancia materializada actual se verifica
en `Axiom/axiom.config/`; la arquitectura vigente se documenta en
`../architecture/02-modelo-de-datos-y-configuracion.md`.

## 2. Brecha de documentación operativa de comandos CLI

`apps/cli/src/commands/` contiene **81 ficheros**; 10 son helpers internos con
prefijo `_` y no comandos invocables por sí mismos. `Axiom/docs/cli/`
documenta en profundidad los 12 comandos del baseline (`init`, `join`,
`configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `tui`, `model`,
`components`, `skills`), pero la cobertura no alcanza a la mayoría de las
superficies añadidas después.

Persisten comandos sin página propia o con cobertura parcial, entre ellos las
familias `workspace*`, `app*`/launcher, `member-install`, `native-mcp-config`,
`external-sync` y varias superficies de workflow y tracker. La lista exacta
debe reconstruirse desde `apps/cli/src/commands/` antes de afirmar cobertura
para un comando concreto.

**Recomendación operativa**: antes de citar el comportamiento de cualquier comando no documentado como contrato estable en un incremento nuevo, verificar directamente en el código (`apps/cli/src/commands/<comando>.ts`), no asumir paridad con los documentados.

## 3. Inferencia de responsabilidad en packages sin README

El runtime actual tiene **43 packages**: 34 packages de primer nivel y 9
sub-packages bajo `packages/adapters/`. Algunos tienen README propio, pero la
mayoría sigue caracterizándose a partir de `src/` y `package.json`. Las tablas
de `../references/01-inventario-de-packages.md` son una aproximación técnica
trazable, no un contrato firmado por cada package.

**Impacto**: tratar las descripciones de esos packages como buena
aproximación, sujeta a corrección si se audita el código fuente completo.

## 4. Roadmap de rediseño (resuelto; INC-24 diferido)

`specs/increments/_archive/INC-20260702-axiom-redesign-roadmap/README.md`
conserva el índice histórico de 24 incrementos. INC-01 a INC-23 y el
side-quest D3 se cerraron y su conocimiento estable se consolidó en
`specs/00_Resumen_Ejecutivo.md` a `specs/08_Glosario.md`; INC-24 Workbench
permanece diferido porque no fue solicitado.

No tratar las preguntas Q1/Q2/Q5 del documento histórico como dudas abiertas
del runtime actual. Para el estado vigente, leer los ocho documentos
canónicos de `specs/` y el contexto técnico reconciliado.

## 5. Ambigüedad de nombres `Axiom.Spec/` vs `axiom.spec/`

Ver `specs/08_Glosario.md`, sección "Aviso de ambigüedad de nombres". Es una fuente real de confusión al leer los docs de `Axiom/` (que usan `axiom.spec/` en minúsculas para su propio catálogo de configuración interna) junto a este repo de workspace (`Axiom.Spec/`, en mayúsculas, con una estructura completamente distinta: `context/`, `specs/`, `templates/`, `plans/`, `prompts/`).

## Cómo mantener este documento

Al cerrar cada incremento nuevo en `Axiom.SDD`, si se resuelve alguno de
estos puntos (por ejemplo, se documenta un comando antes huérfano o se
reduce la inferencia sobre un package), actualizar esta lista quitando o
marcando como resuelto el punto correspondiente, con fecha y referencia al
incremento que lo cerró. Si cambia el árbol real de `Axiom/`, volver a
verificar primero los conteos y las rutas citadas.
