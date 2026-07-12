# Increment — Complete the Azure DevOps client's peripheral surface

> **Código**: INC-20260711-ado-peripheral
> **Continúa**: `INC-20260711-sdd-launcher-p2-tracker` (archivado) — porta la
> superficie ADO PERIFÉRICA que P2 dejó explícitamente diferida.
> **Implementa (parte de)**: epic `INC-20260711-sdd-launcher-core-port` (§5/§6 —
> "superficie ADO periférica" restante).
> **Estado**: archived (green gate; scope completo).
> **Fecha**: 2026-07-12
> **Fuente (read-only)**: `C:/repos/KVP25 Workspace/Kvp.Sdd/tools/sdd-launcher/src/services/adoWorkflowService.ts`
> (2211 LOC) — secciones Build, Git, sprint/iteration y attachments.

## Goal

P2 portó SÓLO la superficie WIT crítica del ciclo de vida detrás de
`IWorkItemTracker` (`@axiom/tracker`) + su impl real `@axiom/tracker-ado`. Este
incremento completa la superficie **PERIFÉRICA** que P2 dejó como STUB, **detrás
del mismo puerto**, de forma **aditiva** y **sin tocar ningún path del core**:

- El swap `tracker.kind: none|ado` sigue siendo **config-only** (invariante P2).
- `@axiom/tracker-ado` sigue **sin importar `vscode`** (RISK-002; guard test).
- Ningún test toca `dev.azure.com` — todo por el `FakeHttpTransport` de P2.

## Scope (las 4 áreas periféricas)

- **Build** (`_apis/build` v7.1) — listar definiciones por nombre, encolar
  (`queueBuild`) y leer (`getBuild`) una build, listar builds completadas de una
  definición, artifacts de una build, work items asociados a una build y changes.
  Portado de la sección Build de `adoWorkflowService.ts`
  (`getBuildDefinitionsByNames`, `getCompletedBuildsForDefinition`,
  `getBuildArtifacts`, `getBuildWorkItems`, `getBuildChanges`) + `queueBuild`/
  `getBuild` (endpoints ADO estándar, para cubrir "queue/get build").
- **Git traceability** (`_apis/git` v7.1) — work items de un commit
  (`commitsbatch`), pull requests por commit, work items de un PR, y el enlace
  **ArtifactLink** de un work item a una **branch / PR / commit** (vstfs URLs,
  con dedup). Portado de la sección Git (`getCommitWorkItems`,
  `getPullRequestsByCommit`, `getPullRequestWorkItems`, `addBranchArtifactLink`,
  `resolveGitRepository`), generalizando el link a los 3 tipos de artefacto.
- **Sprint/iteration catalog** (`_apis/wit/classificationnodes/Iterations`) —
  catálogo de sprints/iteraciones (flatten + detección de sprint activo/último
  cerrado + defaults increment/bug), y el helper de **role-close scheduling**
  (leer snapshot `OriginalEstimate/CompletedWork/RemainingWork` y aplicar las
  operaciones de horas al cerrar). Portado de `getProjectSprints`,
  `readSchedulingSnapshot`, `buildRoleCloseSchedulingOperations` + helpers de
  flatten/active/finished.
- **Attachments** (`_apis/wit/attachments`) — subida binaria (upload) de un
  adjunto y su asociación (`AttachedFile`) a un work item, más un helper
  `uploadAndAttach`. Portado de `uploadInlineAttachments` (el path binario que
  P2 dejó fuera; el texto `ReproSteps`/`Description` ya estaba portado).

## Non-goals (excluido)

- **Sin red**: todos los tests ADO usan el `FakeHttpTransport` in-memory de P2
  (fixtures grabadas); **nunca** `dev.azure.com`.
- No se reescribe la máquina de estados pura `@axiom/workflow`; el tracker sigue
  siendo un side-effect inyectado (invariante P2/RECONCILIATION).
- **Repro-image inline HTML** (`buildReproStepsHtml` con `<img src=…>` a partir
  de tokens `reproAttachments`) queda FUERA: es render de front (P3/P4). Este
  incremento entrega el **camino binario** upload+attach reutilizable; el HTML
  inline con imágenes se compondría encima cuando exista el consumidor de front.
- Catálogos de front puros (`getProjectTags`, `getProjectFeatures`,
  `getProjectUserStories`, `getBugSeverityValues`, `getWorkItemDetailsBatch`)
  siguen fuera — son form-population de P3/P4, no ciclo de vida.

## Design (aditivo, back-compat)

- `IWorkItemTracker` gana **4 accesores de capacidad OPCIONALES** (segregated
  sub-interfaces), no métodos sueltos: `build?: ITrackerBuild`,
  `git?: ITrackerGit`, `sprints?: ITrackerSprints`, `attachments?:
  ITrackerAttachments`. Al ser opcionales, cualquier impl/doble existente de
  `IWorkItemTracker` sigue satisfaciendo el tipo (back-compat total).
- `NullTracker` implementa las 4 capacidades como **no-ops concretos** (arrays
  vacíos / `undefined`), coherente con "local-only sin red".
- `AdoWorkItemTracker` implementa las 4 capacidades de verdad, detrás del
  `HttpTransport` inyectado. Se añade `requestBinary` (body `Buffer`,
  `application/octet-stream`) reutilizando `resolvePat` + la capa de request; los
  impls periféricos viven en módulos aparte (`ado-build.ts`, `ado-git.ts`,
  `ado-sprints.ts`, `ado-attachments.ts`) para mantener el diff acotado.

## Acceptance

- **Build**: `listDefinitions` filtra por nombre contra `_apis/build/definitions`;
  `listBuilds`/`getBuild`/`queueBuild` pegan a `_apis/build/builds…`;
  `getArtifacts`/`getWorkItemRefs`/`getChanges` parsean sus respuestas. Cada test
  asserta path+método+resultado parseado vía fake transport.
- **Git**: `getCommitWorkItems` resuelve el repo y postea a `commitsbatch`;
  `linkArtifact` genera la vstfs URL correcta para branch/PR/commit y hace dedup
  (no re-PATCHea si ya existe el `ArtifactLink`).
- **Sprints**: `listIterations` aplana el árbol de iteraciones, marca el sprint
  activo y calcula defaults; `applyRoleCloseScheduling` lee el snapshot y PATCHea
  `CompletedWork`/`RemainingWork`.
- **Attachments**: `upload` postea binario a `_apis/wit/attachments?fileName=…`
  con `Content-Type: application/octet-stream` y devuelve `{url}`; `attach`
  PATCHea la relación `AttachedFile`; `uploadAndAttach` encadena ambos.
- **NullTracker**: las 4 capacidades son no-ops seguros (arrays vacíos /
  `undefined`, sin throw).
- **Invariantes P2**: swap `none↔ado` sigue siendo config-only; `@axiom/
  tracker-ado` sin `vscode`; ningún test pega a `dev.azure.com`.

## Result

Ver la sección "Result" al final (FULL vs OUT y resultado del gate), rellenada
tras implementar.

---

### Result — qué queda FULL vs OUT

**FULL (portado y testeado, detrás del puerto, sin red):**

- **Build** — `ITrackerBuild`: `listDefinitions(names)`
  (`GET _apis/build/definitions`, filtro por nombre), `listBuilds({definitionId,
  minTime})` (`GET _apis/build/builds?definitions=…&statusFilter=completed…`),
  `getBuild(id)` (`GET _apis/build/builds/{id}`), `queueBuild({definitionId,
  sourceBranch?, parameters?})` (`POST _apis/build/builds`), `getArtifacts(id)`,
  `getWorkItemRefs(id)` (work items de la build), `getChanges(id)`.
- **Git** — `ITrackerGit`: `getCommitWorkItems(repo, commitId)` (resuelve repo +
  `POST commitsbatch`), `getPullRequestsByCommit(repo, commitId)`,
  `getPullRequestWorkItems(repo, prId)`, `linkArtifact({workItemId, repository,
  kind: branch|pullRequest|commit, ref, comment?})` (vstfs `Git/Ref|
  PullRequestId|Commit`, dedup vía lectura de relaciones).
- **Sprints** — `ITrackerSprints`: `listIterations()`
  (`GET _apis/wit/classificationnodes/Iterations?$depth=10`, flatten + activo/
  cerrado + defaults), `applyRoleCloseScheduling({workItemId, loggedHours})`
  (snapshot `OriginalEstimate/CompletedWork/RemainingWork` → PATCH horas).
- **Attachments** — `ITrackerAttachments`: `upload({fileName, content,
  contentType?})` (`POST _apis/wit/attachments`, binario), `attach(workItemId,
  {url, comment?})` (relación `AttachedFile`), `uploadAndAttach(...)`.
- `NullTracker` no-op para las 4 capacidades; `AdoWorkItemTracker` real; ambos
  expuestos como accesores opcionales en `IWorkItemTracker` (aditivo).

**OUT (intencional, con razón):**

- **Repro-image inline HTML** (`buildReproStepsHtml`/`renderReproBlock`/
  `parseReproImageToken`): render de front (P3/P4). El camino binario
  upload+attach SÍ está; el HTML con `<img>` se compone encima cuando exista el
  consumidor de front.
- Catálogos de form-population puros (`getProjectTags/Features/UserStories`,
  `getBugSeverityValues`, `getWorkItemDetailsBatch`): siguen siendo P3/P4.

**Gate (todo verde en `Axiom/`):**

- `npm run build` → PASS (tsc -b; `@axiom/tracker` + `@axiom/tracker-ado`).
- `npm test` → **2618 passing** (255 files); baseline 2588 + 30 tests nuevos
  (build 9 · git 8 · sprints 4 · attachments 4 · NullTracker periférico 5); 0
  test existente debilitado.
- `npm run typecheck` → PASS.
- `npm run doctor` → **PASS** (45/57 OK · 0 FALLO · 3 warn · 9 skip; IX-001 valida
  el nuevo `metadata.yml`; TC-009/PS-001/DF-001 sin cambios).

**Invariantes preservados:** swap `none↔ado` config-only (factory único punto que
lee `kind`); `@axiom/tracker-ado` sin `vscode` (guard test verde); ningún test
pega a `dev.azure.com` (todo por `FakeHttpTransport`, hostname aserotado).
