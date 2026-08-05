# ADR-0032 — Boundary entre `Axiom.Spec/` y `Axiom/axiom.spec/`

## Título

Boundary entre el repo canónico `Axiom.Spec/` y el baseline product-owned `Axiom/axiom.spec/`.

## Identificador

ADR-0032.

## Estado

accepted, verificado el 2026-08-03 como resultado de ACC-003 de R-00.

## Contexto

El workspace contiene dos rutas con grafías casi idénticas y responsabilidades distintas:

- `Axiom.Spec/` (mayúsculas) es el repositorio canónico de especificación del workspace.
- `Axiom/axiom.spec/` (minúsculas) es contenido versionado dentro del repositorio/runtime de Axiom.

La topología activa de `Axiom` declara `specRepo.ref: ../Axiom.Spec` en
`Axiom/axiom.config/topology.yaml`. El contrato `Axiom/axiom.yaml` identifica además
`Axiom.Spec` como `workspaceRoot` del repo con rol
`functional-source-of-truth`. Por separado, el runtime usa `axiom.spec` como scope
por defecto para artefactos de proyectos single-repo y conserva allí una baseline
seleccionada de contenido que puede materializarse en proyectos adoptantes.

## Inventario verificado de `Axiom/axiom.spec/`

El árbol actual tiene exactamente cinco familias y 81 archivos versionados:

| Familia | Conteo verificado | Función observable |
| --- | ---: | --- |
| `increments/` | 2 archivos, 1 subdirectorio | Contiene el artefacto local `increments/INC-20260710-105321-iu074t/` (`README.md` y `metadata.yml`), un probe en estado `specifying`. No es el incremento congelado de ACC-003 ni sustituye el almacén canónico del workspace. |
| `plans/` | 0 archivos | Directorio disponible para el scope de artefactos por defecto; no contiene un plan product-owned en el snapshot verificado. |
| `target-axiom-agents/` | 14 archivos `.md` | Fuentes largas de agents/proceso que los catálogos y las superficies de proceso identifican por `source`. |
| `target-axiom-skills/` | 20 archivos `.md` | Fuentes Markdown de skills que los catálogos y los scaffolders usan como baseline de skills. |
| `templates/` | 45 archivos | Plantillas de bootstrap, artefactos, decisiones, contexto, requisitos, planes, skills, seguridad, troubleshooting y validación. |

### `target-axiom-agents/`

Los 14 sources verificados son:

`axiom-explorer.md`, `axiom-phase-reviewer.md`, `axiom-product-reviewer.md`,
`axiom-qa-validator.md`, `axiom-reviewer.md`, `axiom-role-implementer.md`,
`axiom-role-planner.md`, `axiom-sdd-orchestrator.md`, `axiom-security-reviewer.md`,
`axiom-spec-author.md`, `axiom-spec-integrator.md`, `axiom-spec-planner.md`,
`axiom-tech-context.md` y `axiom-tester.md`.

### `target-axiom-skills/`

Los 20 sources verificados son:

`axiom-capability-router.md`, `axiom-code-intelligence.md`,
`axiom-concision-discipline.md`, `axiom-context-persistence.md`,
`axiom-functional-checklist-coverage.md`, `axiom-install-profile.md`,
`axiom-phase-reviewer.md`, `axiom-plan-drift-alignment.md`,
`axiom-qa-validator.md`, `axiom-role-close-doc.md`, `axiom-role-implementer.md`,
`axiom-role-planner.md`, `axiom-sdd-orchestrator.md`, `axiom-spec-author.md`,
`axiom-spec-integrator.md`, `axiom-structured-doubts.md`, `axiom-tech-context.md`,
`axiom-telemetry.md`, `axiom-terminal-output-efficient.md` y
`axiom-token-optimization.md`.

### `templates/`

Los 45 archivos verificados son:

`agent-log-template.md`, `agents-md-template.md`,
`artifact-baseline-matrix.md`, `bootstrap-template.md`, `bug-acceptance-template.md`,
`bug-metadata-template.yaml`, `bug-model-changes-template.md`,
`bug-requirements-template.md`, `bug-template.md`, `bug-ui-interactions-template.md`,
`business-flows-template.md`, `context-readme-template.md`,
`copilot-instructions.template.md`, `decision-template.md`,
`discovery-provider-overview-template.md`, `domain-model-template.md`,
`executive-summary-template.md`, `functional-requirements-template.md`,
`getting-started-template.md`, `glossary-template.md`,
`increment-acceptance-template.md`, `increment-metadata-template.yaml`,
`increment-model-changes-template.md`, `increment-requirements-template.md`,
`increment-template.md`, `increment-ui-interactions-template.md`,
`integrations-template.md`, `memory-candidate-template.md`, `migration-template.md`,
`non-functional-requirements-template.md`, `onboarding-member-template.md`,
`plan-metadata-template.yaml`, `plan-template.md`, `product-skill-template.md`,
`role-plan-template.md`, `security-template.md`, `skills-lock-template.yaml`,
`spec-template.md`, `specs-readme-template.md`, `technical-context-template.md`,
`token-metrics-template.md`, `troubleshooting-template.md`,
`user-interfaces-template.md`, `verify-multirepo-evidence-manifest-template.yaml` y
`verify-role-global-state-mapping.yaml`.

## Consumidores y flujo verificado

### Increments y plans

`Axiom/packages/workflow/src/artifact-store.ts` define los directorios de artefactos
`increments` y `plans`. También define `DEFAULT_SPEC_REL_PATH = 'axiom.spec'` para
single-repo/self-hosted y `DEDICATED_SPEC_REL_PATH = '.'` cuando el repo resuelto
tiene `role: spec`. Por eso el workflow puede leer y escribir
`<projectRoot>/axiom.spec/increments` y `<projectRoot>/axiom.spec/plans` en el
modelo co-located, mientras un repo de spec dedicado recibe esos directorios en
la raíz de su propio scope.

En este workspace concreto, los comandos que resuelven el spec repo dedicado,
como `Axiom/apps/cli/src/commands/phase.ts`, `freeze.ts` y `knowledge.ts`, operan
sobre `specRepoRoot/specs/increments/<id>` y resuelven `specRepoRoot` desde la
topología. Para `Axiom`, ese root es `../Axiom.Spec`, no
`Axiom/axiom.spec`. El `increments/` local verificado es, por tanto, baseline
product-owned y no la ubicación canónica de este incremento. `plans/` no tiene
archivos actuales; su uso observable es el contrato de path del artifact store,
por lo que se conserva como familia disponible sin atribuirle contenido actual.

### Agents y skills

Los catálogos activos de `Axiom/axiom.config/agents-catalog.yaml` y
`Axiom/axiom.config/skills-catalog.yaml` tienen respectivamente 14 y 20 entries,
y cada entry apunta a un source existente bajo `axiom.spec/target-axiom-agents/`
o `axiom.spec/target-axiom-skills/`, respectivamente. Los paquetes
`Axiom/packages/agents/src/catalog.ts` y `Axiom/packages/skills/src/catalog.ts`
leen esos catálogos; `Axiom/packages/doctor/src/checks.ts` los verifica mediante
TC-011 y TC-010, incluyendo la existencia y el hash del source.

`Axiom/apps/cli/src/commands/workspace-skills.ts` usa el mismo boundary para
scaffolding: escribe una baseline de sources bajo
`<repo>/axiom.spec/target-axiom-skills/`, crea el catálogo de skills y aplica el
set mediante `loadSkillRegistry` y `applySkillSet`. No convierte
`Axiom.Spec/` en una ruta de materialización.

`Axiom/apps/cli/src/commands/workspace-process-surfaces.ts` contiene las formas
bundleadas de las superficies de proceso que corresponden a los sources de
agents y skills del producto. `materializeProcessSurfaces` las emite en cada repo
como `.axiom/agents`, `.axiom/skills`, comandos y formas nativas del adapter,
rellenando `{role}`, `{repoPath}`, `{specRepo}` y `{projectId}`. Después,
`Axiom/apps/cli/src/commands/workspace-catalog-scaffold.ts` puede derivar
`axiom.config/agents-catalog.yaml` y `axiom.skills.lock` desde las superficies ya
materializadas; no copia el catálogo específico del repo producto de Axiom.

### Templates y readiness

`Axiom/packages/workflow/src/artifact-store.ts` busca plantillas en
`<specScope>/templates` y tiene fallback bundleado. `Axiom/apps/cli/src/commands/
configure.ts` lee explícitamente `axiom.spec/templates/copilot-instructions.template.md`
para generar la superficie de Copilot. `workspace-adapter-templates.ts` resuelve
el template on-disk en `axiom.spec/templates/agents-md-template.md` y mantiene
una copia bundleada para topologías adoptadas sin ese archivo.

`Axiom/scripts/verify-first-project-readiness.mjs` es el flujo operativo más
concreto: siembra en un proyecto temporal `axiom.config/`,
`axiom.spec/templates/`, `axiom.spec/target-axiom-skills/` y
`axiom.spec/target-axiom-agents/` desde el repo producto, y después ejecuta
`init -> configure -> toolchain repair -> sync -> start -> gateway -> audit ->
doctor`. Esto confirma que esas tres familias son baseline de instalación.

El scaffolder de un spec repo dedicado es distinto: `workspace-setup.ts` llama a
`workspace-spec-base.ts` sólo cuando crea un repo de spec nuevo y materializa allí
la base `specs/` y `context/`; no usa `Axiom/axiom.spec/` como repo canónico ni
mueve su contenido.

## Decisión

1. `Axiom.Spec/` (mayúsculas) se mantiene como el repositorio canónico de
   especificación del workspace: specs numeradas, incrementos, bugs, planes,
   decisiones y contexto estable.
2. `Axiom/axiom.spec/` (minúsculas) se mantiene en su ubicación actual como
   contenido product-owned dentro del repo/runtime adoptante. Es baseline de
   artefactos single-repo y de instalación/materialización, no un duplicado del
   repositorio canónico del workspace.
3. En este pase no se mueve, elimina, renombra ni copia ningún archivo de
   `Axiom/axiom.spec/`. No se cambia runtime, catálogos ni el contenido de sus
   cinco familias.
4. La documentación debe conservar ambas grafías y calificarlas por su
   responsabilidad. No se debe usar la ruta física del workspace `Axiom.Spec/`
   como destino de materialización de contenido de un proyecto adoptante; esa
   operación debe resolver el `specRepo` del proyecto y su scope. La existencia
   de un scaffolder para un spec repo nuevo no cambia esta regla ni convierte el
   sibling canónico en baseline product-owned.

## Alternativas consideradas

- **Mover `Axiom/axiom.spec/` a `Axiom.Spec/` o fusionar ambas raíces.** Rechazada:
  rompería el boundary entre el repositorio canónico del workspace y el contenido
  que el producto entrega dentro de un repo adoptante, además de invalidar los
  consumidores y paths de baseline verificados.
- **Eliminar o renombrar `Axiom/axiom.spec/`.** Rechazada: los catálogos, el
  artifact store single-repo, `configure`, los adapters, el readiness smoke y
  los scaffolders dependen de la convención `axiom.spec`.
- **Mantener la estructura sin aclaración documental.** Rechazada: deja abierta
  la confusión que originó ACC-003 y permite tratar la ruta canónica como un
  destino de materialización.

## Consecuencias

- El workspace conserva dos superficies legítimas con grafías distintas y una
  responsabilidad explícita para cada una.
- Los consumidores single-repo pueden seguir usando `axiom.spec` sin migración
  de runtime; los proyectos multi-repo resuelven el spec repo por topología.
- `Axiom.Spec/` no debe aparecer como ruta fija de salida de scaffolding para un
  proyecto adoptante. La documentación operativa debe decir cuándo se refiere
  al repo canónico y cuándo a la carpeta product-owned.
- `increments/` y `plans/` permanecen como familias del baseline aunque el
  snapshot actual sólo contenga un probe de increment y ningún plan. No se
  infiere que sus contenidos sean la spec canónica de este workspace.
- El seguimiento queda limitado a mantener esta distinción en documentación y
  a revisar referencias futuras que mezclen `Axiom.Spec/` con `axiom.spec/`.
  No se autoriza en este incremento una migración de paths ni una modificación
  del runtime.

## Relación con specs/planes/código

- Incremento congelado: `Axiom.Spec/specs/increments/INC-20260803-r00-axiom-spec-boundary/README.md`.
- Plan de origen: `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`, ACC-003.
- Plantilla usada: `Axiom.Spec/templates/decision-template.md`.
- Contrato de topología: `Axiom/axiom.config/topology.yaml`.
- Contrato del producto: `Axiom/axiom.yaml`.
- Corrección documental puntual asociada: `Axiom/docs/overview.md`.
- No se editan `Axiom.Spec/specs/00_Resumen_Ejecutivo.md` a
  `08_Glosario.md`, `Axiom.Spec/context/**`, el plan R-00 ni el README
  congelado. La integración final de conocimiento estable y el cierre/archivo
  del incremento quedan para el orquestador.

## Fecha

2026-08-03.

## Responsable

Axiom Bootstrap Orchestrator — ejecución documental de ACC-003.
