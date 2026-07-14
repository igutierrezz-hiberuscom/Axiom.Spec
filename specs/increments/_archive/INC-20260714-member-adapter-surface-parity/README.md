# INC-20260714 — `member install` + `adapter add` surface/code-intel parity

> **Código**: INC-20260714-member-adapter-surface-parity
> **Estado**: archived
> **Fecha**: 2026-07-14
> **Batch**: cierre de brechas post-KVP25 (gap 3/5, orden 1→2→3→5→4)
> **Groundeo**: catálogo `INC-20260713-e2e-corner-case-catalog` (CC-ADP-02, CC-ADP-03; "Known gaps to build" P1 #3/#4); `INC-20260713-ns2-adapter-generation` (NS-2, `materializeProcessSurfaces`); `INC-20260713-ns3-codeintel-mcp` (NS-3, code-intel en configs MCP nativas); memoria `project-axiom-kvp25-integration-test`.

## Goal

`workspace setup` / `adopt` / `repo add` materializan las **process-surfaces por-rol** (NS-2:
`axiom-spec-author`/`axiom-role-planner`/`axiom-role-implementer`/`axiom-sdd-orchestrator`) y las
**entradas MCP de code-intel** (NS-3: serena/codegraph pinneadas al repo, cuando hay un proveedor
habilitado). `axiom member install` y `axiom adapter add` NO lo hacen: un miembro que clona y
corre `member install` obtiene brokers + activación de toolchain pero nunca sus superficies de
proceso ni code-intel; `adapter add` regenera AGENTS.md pero omite las surfaces y su llamada a
`writeWorkspaceNativeMcpConfigs` descarta el arg `codeIntelProviders`. Brechas **CC-ADP-02**
(member install) y **CC-ADP-03** (adapter add).

## Owner decision (el diseño)

- **Reusar, no reconstruir.** Ambos comandos reutilizan `materializeProcessSurfaces`
  (`workspace-process-surfaces.ts`, NS-2) y `writeWorkspaceNativeMcpConfigs` con el arg
  `codeIntelProviders` + el `kind` por-repo (`workspace-mcp.ts`, NS-3). El repo de referencia
  es `runRepoAdd` (`workspace-incremental.ts`, ~L668–754), que YA hace ambas cosas — se replica
  su patrón, no se inventa arquitectura nueva.
- **Señal de code-intel idéntica al resto del motor:**
  `codeIntelProviders = (workspaceJson.providers ?? []).filter(isCodeIntelProviderId)`. Default
  vacío ⇒ NADA de code-intel se emite ⇒ salida **byte-idéntica** a hoy cuando no hay proveedor
  habilitado (no-regresión, igual que NS-3 en `repo add`).
- **Rol/kind por-repo derivado de la topología:** `sddRepo`→`control`, `specRepo`→`spec`,
  cada `roleCodeRepositories[i]`→`role` con `assignments.find(a=>a.repoId===ref.id)?.roleId`.
  Sólo los repos `kind==='role'` (código) reciben code-intel (lo gatea
  `writeWorkspaceNativeMcpConfigs`).
- **`adapter add`** (`runAdapterAdd`, `workspace-incremental.ts`): para cada repo existente del
  proyecto (`project.repos`), materializar las process-surfaces del **nuevo target** (según el
  rol del repo) y añadir `codeIntelProviders` + `kind` a la llamada `writeWorkspaceNativeMcpConfigs`
  (hoy en ~L864 va sin ellos).
- **`member install`** (`runMemberInstall`, `member-install.ts`): para cada repo que el miembro
  tiene localmente (path bindeado que existe en disco), materializar sus process-surfaces por-rol
  y —en repos de código, si hay code-intel habilitado— emitir sus entradas MCP nativas de
  code-intel. Extender la emisión MCP nativa (hoy sólo `sdd`+`spec`) para cubrir también los
  repos de código que el miembro tenga.
- **Best-effort** por-repo (un fallo agrega warning, no aborta), igual que en `runRepoAdd`.

## Scope

- `apps/cli/src/commands/workspace-incremental.ts` — `runAdapterAdd`: + `materializeProcessSurfaces`
  por repo para el nuevo target; + `codeIntelProviders` + `kind` en `writeWorkspaceNativeMcpConfigs`.
- `apps/cli/src/commands/member-install.ts` — `runMemberInstall`: + materialización de
  process-surfaces por repo local bindeado; + code-intel en la config MCP nativa de los repos de
  código; extender el loop nativo más allá de sdd+spec.
- `apps/cli/src/commands/workspace-process-surfaces.ts` y `workspace-mcp.ts` — **reutilizados**;
  cambios sólo si hace falta un helper aditivo mínimo (p. ej. derivar rol/kind por-repo). No
  cambiar firmas públicas de forma incompatible.
- **Tests** — integración: tras `runMemberInstall` sobre un clon multi-repo, cada repo local
  tiene sus surfaces (`.axiom/agents/<surface>.md` por rol) y —con proveedor habilitado— sus
  entradas code-intel en la config nativa; idem `runAdapterAdd`. Regresión: sin code-intel
  habilitado, salida byte-idéntica (los tests positivos existentes de `member-install`/
  `workspace-incremental` no deben romperse; reemplazar los guard-tests que hoy fijan la
  AUSENCIA por asserts de la nueva presencia, documentando que el comportamiento cambió a
  propósito — NO debilitar).
- **e2e en sandbox aislado** (KVP25, HOME hermético): setup del multi-repo por el "arquitecto" →
  simular un "miembro" (clon) → `member install` con binds → verificar surfaces + (si habilitado)
  code-intel por repo; `adapter add <target>` → verificar surfaces del nuevo target + code-intel.

## Non-goals

- No forzar proveedores en proyectos que no opt-in (default vacío ⇒ no-op).
- No cambiar el motor `runWorkspaceSetup`/`runRepoAdd` (ya correctos; son la referencia).
- No cambiar firmas públicas de `materializeProcessSurfaces` / `writeWorkspaceNativeMcpConfigs`
  de forma incompatible (extensiones aditivas/opcionales sólo si imprescindibles).

## Acceptance

- Tras `member install` sobre un clon, cada repo que el miembro tiene → sus process-surfaces por
  rol + (si hay proveedor code-intel habilitado) sus entradas code-intel MCP.
- Tras `adapter add <target>`, cada repo → las surfaces del nuevo target + (si habilitado)
  code-intel; la llamada nativa YA no descarta `codeIntelProviders`.
- Sin proveedor code-intel habilitado ⇒ salida byte-idéntica a hoy (no-regresión).
- Gate verde en `Axiom/`: `npm run build` → `npm test` → `npm run typecheck` →
  `node apps/cli/dist/index.js doctor` (módulo el flake 5000ms).

## Result

Shipped, green on all gates (build / test / typecheck / doctor). Ambas brechas (CC-ADP-02
`member install`, CC-ADP-03 `adapter add`) cerradas replicando el patrón de `runRepoAdd`
(reuse, no arquitectura nueva) — cero cambios de firma pública en `materializeProcessSurfaces`
ni `writeWorkspaceNativeMcpConfigs`.

### Archivos cambiados

- `Axiom/apps/cli/src/commands/workspace-incremental.ts` — `runAdapterAdd`:
  - Reconstruye `repoSpecs = buildRepoSpecsFromResolvedProject(project)` (la MISMA derivación
    role/kind/functionalRoleId que `runRepoAdd` ya usaba para sus `existingSpecs`) y, para cada
    repo existente, llama `materializeProcessSurfaces` con `adapters:[target]` (NS-2).
  - La llamada a `writeWorkspaceNativeMcpConfigs` (antes sin `codeIntelProviders` ni `kind`) ahora
    pasa `repos: repoSpecs.map(r => ({roleKey, path, kind: r.kind}))` y
    `codeIntelProviders = (workspaceJson.providers ?? []).filter(isCodeIntelProviderId)` — desacoplado
    de si `.axiom/mcp.yml` existe, para que un proyecto con provider habilitado pero sin brokers
    wireados igual reciba code-intel en sus repos de código.
  - +1 import: `type McpServerEntry` de `@axiom/user-workspace`.
- `Axiom/apps/cli/src/commands/member-install.ts` — `runMemberInstall`:
  - Nuevo paso "2c": para cada entrada de `bound` (repo que el miembro tiene localmente, path
    existente en disco), deriva `repoRole`/`kind`/`functionalRoleId`/`role` desde la topología
    (`deriveRepoRoleAndKind`, nueva función local, D-002) y llama `materializeProcessSurfaces` con
    los adapters resueltos por `resolveTargetAdapters` (ya existente).
  - Paso "3" (MCP) reescrito: los `nativeServers` ahora son `McpServerEntry[]` completos
    (`type:'axiom', scope:'repo', enabled:true`, no sólo `{id,command,args}`); se agrega
    `resolveEnabledCodeIntelProviders` (nueva función local, mismo criterio best-effort/personal que
    `resolveTargetAdapters`, lee `.axiom-state/<projectId>/workspace.json#providers` filtrado por
    `isCodeIntelProviderId`); el loop ad-hoc `writeNativeMcpConfig` por repo×adapter fue
    REEMPLAZADO por una única llamada a `writeWorkspaceNativeMcpConfigs` (el mismo helper que
    `runRepoAdd`), con `repos` extendido más allá de sdd+spec para incluir cada repo de código
    (`kind:'role'`) que el miembro tenga bindeado, y `codeIntelProviders` threaded.
  - Nuevo import: `materializeProcessSurfaces`, `writeWorkspaceNativeMcpConfigs` +
    `WorkspaceMcpNativeRepo` (de `./workspace-mcp`), `relativeRef` (de `./workspace-setup`),
    `isCodeIntelProviderId` (de `@axiom/providers`), `type McpServerEntry` (de
    `@axiom/user-workspace`), `type RepoRole` (de `./init`).
  - `workspace-process-surfaces.ts` y `workspace-mcp.ts` — **sin cambios** (reuso puro; ya
    exponían todo lo necesario, incl. el `kind` opcional de NS-3 de un incremento previo).
- Tests:
  - `Axiom/apps/cli/tests/workspace-incremental.test.ts` — +1 describe (`runAdapterAdd — NS-2
    surfaces + NS-3 code-intel`), 3 tests nuevos.
  - `Axiom/apps/cli/tests/member-install.test.ts` — +1 describe (`runMemberInstall — NS-2
    surfaces + NS-3 code-intel`), 2 tests nuevos.

### Diseño as-built vs. spec

- **D-001/D-003 (reuse):** confirmado tal cual — `materializeProcessSurfaces` y
  `writeWorkspaceNativeMcpConfigs` se consumen sin tocar sus firmas.
- **D-002 (rol/kind por-repo):** implementado con DOS caminos distintos, cada uno el más simple
  disponible en su contexto (ninguno inventa arquitectura nueva):
  - `runAdapterAdd` opera sobre `ResolvedExistingProject` (vía el registry), así que reusa
    `buildRepoSpecsFromResolvedProject` (YA EXISTENTE, usado por `runRepoAdd` para sus
    `existingSpecs`) en vez de re-derivar desde `topology.yaml`. Esta derivación es
    estructuralmente EQUIVALENTE a "sddRepo→control, specRepo→spec, roleCodeRepositories→role +
    assignments.roleId": en TODO el código base, `functionalRoleId` para un repo de rol siempre se
    fija igual a su `roleKey`/`topologyId` (confirmado grep-eando cada sitio que setea
    `functionalRoleId:`), así que `assignments.find(a=>a.repoId===ref.id)?.roleId` y
    `r.roleKey` resuelven al MISMO valor. Se documenta como deviation menor y deliberada.
  - `runMemberInstall` NO usa el registry (opera directo sobre `topology.yaml` +
    `topology-bindings.yaml`, el diseño pre-existente del comando), así que sí se implementó la
    derivación EXACTA descrita en D-002 vía la nueva función local `deriveRepoRoleAndKind`.
- **Byte-identidad sin provider:** verificada con asserts explícitos (`Object.keys(mcpServers)`
  exacto) en ambos archivos de test — sin provider habilitado, la config nativa del repo de
  código sólo trae `sdd-mcp-server`/`spec-mcp-broker`, igual que antes del incremento.

### Tests — qué se agregó y por qué (no había guard-tests de AUSENCIA que "flipear")

Se auditó todo `apps/cli/tests/` buscando asserts que fijaran la AUSENCIA de surfaces/code-intel en
`member-install`/`adapter add` (mencionados en el catálogo CC-ADP-02/03) — **no existían**: ni
`member-install.test.ts` ni `workspace-incremental.test.ts` tenían ningún test que hiciera
`expect(fs.existsSync(...)).toBe(false)` sobre `.axiom/agents/*` o sobre entradas de code-intel para
estos dos comandos (el catálogo describía el GAP funcional, no un test que lo fijara). Por lo tanto
no se "flipeó" ningún test existente — se agregaron tests NUEVOS que fijan el comportamiento
correcto (presencia + gating + no-regresión):

- `workspace-incremental.test.ts` (`runAdapterAdd — NS-2 surfaces + NS-3 code-intel`):
  1. Materializa `.claude/agents/<surface>.md` role-diferenciado en control/spec/backend + la
     forma portable `.axiom/agents/...`; el backend NO recibe superficies fuera de su rol.
  2. Sin provider: `.mcp.json` del backend trae EXACTAMENTE `['sdd-mcp-server','spec-mcp-broker']`
     (no-regresión, byte-idéntico).
  3. Con `serena` habilitado (`runProviderAdd`): el backend recibe `serena`; control/spec NO.
- `member-install.test.ts` (`runMemberInstall — NS-2 surfaces + NS-3 code-intel`):
  1. Multi-repo (sdd+spec+backend) recién bindeado: surfaces por-rol en los 3 repos (`.axiom/
     agents/axiom-sdd-orchestrator.md`, `axiom-spec-author.md`+`axiom-role-planner.md`,
     `axiom-role-implementer.md`); sin provider habilitado, `.mcp.json` del backend trae
     EXACTAMENTE los 2 brokers.
  2. Con `.axiom-state/<projectId>/workspace.json#providers:['serena']` sembrado (simulando que
     el MIEMBRO también lo tiene local — ver nota de diseño abajo): el backend recibe `serena`;
     control/spec NO; y el adapter-scoping respeta `#adapters` (sólo `claude-code`, sin
     `.cursor/rules/*`).

No se debilitó ningún assert preexistente.

### Hallazgo notable (fuera de alcance, no se tocó)

`resolveTargetAdapters`/la nueva `resolveEnabledCodeIntelProviders` leen
`.axiom-state/<projectId>/workspace.json` (adapters/providers) — un archivo PERSONAL/gitignored del
ARQUITECTO. Un miembro que clona el repo (sin `.axiom-state/`) NUNCA hereda automáticamente la
selección de adapters/providers del arquitecto vía git; sólo la hereda si él mismo siembra su
propio `workspace.json` local (p. ej. corriendo `workspace setup`/`configure` localmente, o a mano).
Esto es un comportamiento PRE-EXISTENTE del motor (`resolveTargetAdapters` ya tenía este mismo
límite antes de este incremento, documentado en su propio comentario) — este incremento lo hereda
tal cual para `codeIntelProviders`, consistente con el Non-goal "no cambiar el motor". Confirmado en
el e2e: sin ese archivo sembrado, `member install` cae a TODOS los adapters nativos soportados
(comportamiento ya documentado/deseado, "escribir de más es inocuo") y a CERO providers de
code-intel (no-regresión); sembrando `workspace.json#providers` local, el NUEVO código de este
incremento wirea `serena` correctamente en los repos de código, no en control/spec.

Separadamente: `.mcp.json`/`opencode.json`/etc. NO están en `.gitignore` (a diferencia de `.axiom/`
y `.axiom-state/`), así que en un clon real SÍ viajan vía git con el `--project-root` ABSOLUTO del
arquitecto (stale para el miembro). `writeWorkspaceNativeMcpConfigs` es merge-preserving por
`id`, así que el `member install` del miembro los REFRESCA a su propio path absoluto para
sdd/spec/code-intel — comportamiento correcto, observado en el e2e, y preexistente (no introducido
por este incremento).

### e2e — sandbox aislado (KVP25, HOME hermético)

Sandbox: `scratchpad/sandbox/kvp25` (baseline, preservado pristine) → copiado a
`scratchpad/sandbox/work-gap3` (arquitecto) y `scratchpad/sandbox/work-gap3-member` (miembro,
simulando un clon: se removieron `.axiom-state/` y `.axiom/` de la copia, que SÍ están
gitignoreados). `HOME`/`USERPROFILE` apuntados al home hermético (`scratchpad/sandbox/.axiom-home`)
para AMBOS roles (mismo home, ya usado por incrementos previos de este mismo catálogo — projectId
`kvp25-gap3` para no colisionar con `kvp25-gap1`/`kvp25-gap2`).

1. **Arquitecto** — `workspace setup --name kvp25-gap3 --control-path . --spec-path ../Kvp.Spec
   --role backend:.../Kvp.Api --role frontend:.../Kvp.App --role e2e:.../Kvp.QA --adapters
   claude-code --providers serena` sobre `work-gap3/Kvp.Sdd` → `✓ Workspace "kvp25-gap3"
   inicializado` + `Registro: OK`. Verificado: `.axiom/agents/axiom-sdd-orchestrator.md` (control),
   `axiom-spec-author.md`+`axiom-role-planner.md` (spec), `axiom-role-implementer.md` (backend);
   `.mcp.json` del backend con `serena` (NS-3), del control SIN `serena` (comportamiento YA
   correcto de `runWorkspaceSetup`, la referencia).
2. **Miembro** — `member install --member bob --bind backend:<abs> --bind frontend:<abs> --bind
   e2e:<abs>` sobre `work-gap3-member/Kvp.Sdd` (sin `.axiom-state`/`.axiom` previos, real
   fresh-clone state):
   - Sin `workspace.json` local sembrado: `"superficies de proceso...: 70 archivo(s) en 5 repo(s)
     local(es)."` + `"config MCP nativo: 25 archivo(s)"` (5 repos × 5 adapters nativos, default sin
     preferencia declarada); el `serena` PRE-EXISTENTE heredado del clon (ver hallazgo arriba)
     quedó byte-idéntico, intacto (codeIntelProviders vacío ⇒ no tocado).
   - Sembrando `.axiom-state/kvp25-gap3/workspace.json` con `{adapters:['claude-code'],
     providers:['serena']}` (simulando que el miembro también optó localmente) y re-corriendo:
     `"superficies de proceso...: 35 archivo(s)"` (sólo claude-code) + `"config MCP nativo: 5
     archivo(s)"`; `backend/.mcp.json` ahora con `serena` apuntado al path ABSOLUTO del MIEMBRO
     (`work-gap3-member/Repos KVP25/Kvp.Api`); `control`/`spec` `.mcp.json` sin `serena` (gating
     `kind==='role'` confirmado en un proyecto real).
3. **`adapter add opencode`** sobre `work-gap3/Kvp.Sdd` (arquitecto) → `.opencode/agents/
   axiom-role-implementer/` NUEVO en backend (NS-2 del nuevo target) + `opencode.json` NUEVO con
   `serena` en backend, SIN `serena` en control (NS-3 del nuevo target, `codeIntelProviders`
   threaded correctamente).

Cero mutaciones git en todo el flujo (sólo escritura de archivos vía la CLI). Baseline `kvp25/`
confirmado pristine al final (sin `axiom.yaml`/`.axiom-state`/`axiom.config`). Real `~/.axiom/
projects.yml` confirmado sin `kvp25-gap3` (0 matches) — el home hermético nunca se filtró al real.

### Gate (`Axiom/`)

- `npm run build` → OK (tsc -b, sin errores).
- `npm run typecheck` → OK (tsc -b, sin errores).
- `npm test` → **279/279 archivos, 2810/2810 tests, 0 fallos** (corrida limpia final, sin carga
  concurrente de sandbox). Nota de flake observada DURANTE el desarrollo (bajo carga, con las
  copias del sandbox KVP25 corriendo en paralelo): dos corridas completas previas mostraron entre
  5 y 6 archivos fallando puramente por `Test timed out in 5000ms` (`launcher-panels.test.ts`,
  `workspace-command.test.ts`, `workspace-incremental.test.ts`, `workspace-setup.test.ts`,
  `member-install.test.ts`, `e2e/workspace-adopt.e2e.test.ts`) — CERO `AssertionError`. Re-corridos
  en aislamiento (`npx vitest run <files> --no-file-parallelism`): **94/94 verdes**. Una tercera
  corrida (bajo carga aún mayor) sumó `packages/memory/tests/engram-backend.test.ts` (assertion
  sobre selección de backend 'json' vs 'engram', dependiente del entorno/PATH real — archivo
  explícitamente nombrado como flaky-bajo-carga en las instrucciones del batch); re-corrido aislado:
  **15/15 verdes**. Ningún test fue debilitado ni el timeout committeado se tocó.
- `node apps/cli/dist/index.js doctor` → **PASS** (`45/57 OK · 0 FALLO · 3 ADVERTENCIA · 9
  OMITIDO`, mismas advertencias/omisiones pre-existentes del repo, ninguna nueva).

### Revisión contra acceptance criteria

- ✅ Tras `member install`, cada repo local del miembro recibe sus process-surfaces por-rol +
  (con provider habilitado) sus entradas code-intel — confirmado por tests + e2e.
- ✅ Tras `adapter add <target>`, cada repo recibe las surfaces del nuevo target + (si habilitado)
  code-intel; la llamada nativa ya no descarta `codeIntelProviders`/`kind` — confirmado por tests +
  e2e.
- ✅ Sin provider de code-intel habilitado, salida byte-idéntica (brokers únicamente) — confirmado
  por asserts explícitos de key-set exacto en ambos archivos de test.
- ✅ Gate verde en `Axiom/` (build/test/typecheck/doctor) — ver arriba.

### General spec integration

`Axiom.Spec` en este workspace **no tiene** un `general-spec.md` (evolucionó hacia
`specs/`+`context/`+`technical-context/`+`bugs/`+`plans/`, sin ese archivo — confirmado: no existe
en ningún nivel del repo). No hay conocimiento nuevo de arquitectura/producto que requiera un lugar
estable distinto de este propio increment: el patrón "reusar `materializeProcessSurfaces`/
`writeWorkspaceNativeMcpConfigs`, nunca reconstruirlos" ya estaba consolidado por los incrementos
NS-2/NS-3 referenciados (groundeo de este mismo archivo); este incremento sólo extiende sus
CONSUMIDORES (`member install`, `adapter add`) al mismo patrón, sin agregar una decisión de diseño
nueva que amerite un documento separado. Nada que integrar.

**Status: closed** — goal claro, acceptance criteria explícitos y verificados, cambios
implementados y validados (build/typecheck/test/doctor + e2e real en sandbox aislado), review
contra acceptance completo, y sin conocimiento estable pendiente de integrar.
