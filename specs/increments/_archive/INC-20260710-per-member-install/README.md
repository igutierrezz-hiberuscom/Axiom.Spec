# Increment: separación de responsabilidades ARQUITECTO vs MIEMBRO (per-member install)

Status: closed
Date: 2026-07-10

## Goal

Auditar y corregir la separación entre configuración SHARED (que el
ARQUITECTO define y commitea una vez: qué repos existen, qué MCPs
existen, qué utilidades están habilitadas) y configuración
PERSONAL/per-user (que cada MIEMBRO del equipo materializa en SU propia
máquina tras clonar: dónde viven esos repos en disco, cómo se lanza el
server MCP desde esta máquina, y el estado local de activación de las
utilidades). Cerrar los gaps encontrados con un comando de onboarding
headless (`axiom member install`) y un comando de reparación puntual
(`axiom bindings`).

## Context

`@axiom/topology` ya separaba `topology.yaml` (shared, versionado) de
`topology-bindings.yaml` (personal, gitignored) desde
`INC-20260702-axiom-redesign-roadmap`. Lo que faltaba era: (1) un
comando de onboarding que un MIEMBRO recién clonado pudiera correr para
materializar TODO lo personal en un solo paso, headless; (2) verificar
que el `.gitignore` generado realmente cubre TODO lo personal (no sólo
los bindings); (3) que el launch command MCP generado sea REALMENTE
ejecutable en la máquina de cualquier miembro, no sólo en la del
arquitecto.

## Scope

- Comando nuevo `axiom member install` (`apps/cli/src/commands/
  member-install.ts`): registra al miembro (reusa `join`), bindea repos
  lógicos a paths reales, materializa config MCP nativo con un launch
  command resolvable, activa estado local de toolchain, imprime guía de
  instalación externa honesta.
- Comando nuevo `axiom bindings show|set|remove` (`apps/cli/src/
  commands/bindings.ts`): reparación puntual de un binding sin
  re-correr todo el install.
- Fix de `buildGitignore()` (`init.ts`): `.axiom-state/` completo, no
  sólo `.axiom-state/local/`.
- Fix de resolución del launch command MCP (`resolveMcpLaunchCommand`,
  `workspace-mcp.ts`): `axiom` si resuelve en PATH, si no `node
  <cliEntryPath>`, wireado en `runWorkspaceSetup` y en `member install`.
- Fix descubierto en la validación viva (no en el brief original, pero
  bloqueante para el goal): el gate del orchestrator (`hasInitJson`)
  impedía que `axiom join`/`axiom member install` completaran sobre
  proyectos bootstradeados vía `axiom workspace setup` — `member
  install` ahora sintetiza un `init.json` local mínimo antes de
  invocar `join`.
- Documentación de la separación de responsabilidades en
  `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` (tabla shared vs
  personal + hallazgos del audit) y `05_Interfaces_Operativas.md`
  (superficie CLI de los dos comandos nuevos).

## Non-goals

- No se gitignorean los configs MCP nativos por adapter (`.mcp.json`,
  `.cursor/mcp.json`, `.vscode/mcp.json`, `opencode.json`) — aunque
  embeben paths absolutos per-máquina, viven en la raíz del repo (no
  bajo `.axiom-state/`/`.axiom/`) y decidir si un equipo quiere
  commitear una versión base de estos (algunos SÍ lo hacen a propósito,
  por convención de IDE compartida) es una decisión de producto
  separada, no un bug de este incremento. Documentado como hallazgo del
  audit, no implementado (ver AGENTS.md: documentar future
  considerations, no especular).
- No se toca `@axiom/orchestrator`'s `STATE_MACHINE`/gates
  compartidos. El fix del gate `hasInitJson` se resolvió LOCALMENTE en
  `member-install.ts` (sintetizando el `init.json` que el gate espera,
  antes de invocar `join`), no relajando la precondición compartida —
  cero riesgo para `configure`/`sync`/`start`/`audit`/`upgrade`, que
  también dependen de `hasInitJson` y no fueron tocados.
- No se unifica `axiom.config/toolchain-catalog.yaml` (catálogo global)
  con `axiom.config/toolchain.yaml` (subset habilitado del proyecto) —
  ya eran conceptos distintos desde `INC-20260710-schema-reconciliation`
  y este incremento los consume tal cual, sin cambiarlos.
- No se instalan binarios reales de terceros (serena, codegraph,
  graphify, etc.) — Axiom nunca lo hizo y este incremento no cambia esa
  postura; sólo mejora la HONESTIDAD de la guía impresa.
- No se expone `--home-dir` como flag de `axiom workspace setup` (no
  lo tenía antes de este incremento); la validación viva usa el
  override estándar de `os.homedir()` (`USERPROFILE`/`HOME`) para
  sandboxear sin tocar ese comando.

## Acceptance criteria

- [x] Step 0 (audit) documentado con hallazgos concretos y verificados
      contra el `.gitignore` real, a mano, del propio repo `Axiom/`.
- [x] `axiom member install` existe, es idempotente, soporta `--bind`
      repetible, `--no-register`, `--home-dir`/`$AXIOM_HOME_DIR`,
      `--json`.
- [x] `axiom bindings show|set|remove` existe y opera sobre
      `topology-bindings.yaml`.
- [x] `resolveMcpLaunchCommand()` reemplaza la asunción incondicional
      `axiom` en PATH, wireada en `runWorkspaceSetup` y `member
      install`, con test dedicado.
- [x] `buildGitignore()` ignora `.axiom-state/` completo; test
      actualizado para reflejarlo.
- [x] Responsabilidad SHARED/committed vs PERSONAL/gitignored
      documentada en `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
      y `05_Interfaces_Operativas.md`.
- [x] `npm run build` limpio.
- [x] Simulación viva de dos actores (arquitecto + miembro) en TEMP
      dirs, con `HOME`/`USERPROFILE` sandboxeados — ver "Validation".
- [x] `npx vitest run` (suite completa): 211 archivos / 2239 tests,
      cero fallos (baseline previo: 209/2221 — +2 archivos de test
      nuevos, +18 tests nuevos).

## Open questions

Ninguna bloqueante.

## Assumptions

- El "clone fresco" de un miembro se simula copiando, en la validación
  viva, todo lo que NO está gitignoreado hoy (equivalente a "todo lo
  que el arquitecto commiteó antes de que existiera nada
  gitignoreado") — es la simulación más honesta posible sin un repo git
  real de por medio (esta tarea prohíbe explícitamente correr comandos
  `git`).
- `axiom.config/mcp-manifest.yaml` sólo declara los ids canónicos `sdd`
  (→ `sdd-mcp-server`) y `spec` (→ `spec-mcp-broker`) — los únicos dos
  que `axiom mcp serve --kind <sdd|spec>` sabe lanzar. Un id
  desconocido en el manifest genera un warning explícito en `member
  install`, nunca una excepción ni un launch command inventado.
- `axiom.config/toolchain.yaml` (no `toolchain-catalog.yaml`) es la
  fuente de "qué utilidades activar" para un miembro — ya era la
  interpretación correcta según `INC-20260710-schema-reconciliation`
  (`toolchain.yaml` es el manifest project-scoped que el arquitecto
  puebla vía `axiom toolchain add`; el catálogo global sólo restringe
  qué ids son válidos).

## Implementation notes

### Step 0 — AUDIT (hallazgos)

Verificado contra el `.gitignore` real, a mano, del propio repo
`Axiom/` (`# Axiom local overlay (per-repo runtime state; never
versioned)` → `.axiom-state/`) y contra el comportamiento real de
`buildGitignore()`/`runWorkspaceSetup`/`join`/`workspace-mcp.ts`:

| Artefacto | Antes de este incremento | Rol correcto | Estado tras el fix |
|---|---|---|---|
| `axiom.yaml`, `axiom.config/topology.yaml`, `axiom.config/mcp-manifest.yaml`, `axiom.config/toolchain-catalog.yaml`, `axiom.config/toolchain.yaml`, spec repo | Committeados, no gitignoreados | SHARED (arquitecto) | Sin cambios — ya correcto |
| `.axiom-state/local/topology-bindings.yaml` | Gitignoreado (`.axiom-state/local/` explícito) | PERSONAL (miembro) | Sin cambios — ya correcto |
| `.axiom-state/<projectId>/` (`members.yaml`, `init.json`, `workspace.json`, checkpoints, telemetry) | **NO gitignoreado** (`buildGitignore()` sólo cubría `local/`) | PERSONAL | **FIX #4**: `.axiom-state/` completo ahora gitignoreado |
| `.axiom/mcp.yml` | Gitignoreado (`.axiom/` explícito) | PERSONAL/derivado | Sin cambios — ya correcto |
| `.mcp.json`/`.cursor/mcp.json`/`.vscode/mcp.json`/`opencode.json` | **NO gitignoreado** | Ambiguo — embeben paths absolutos per-máquina pero viven fuera de `.axiom-state/`/`.axiom/` | **Documentado, no corregido** (ver Non-goals #1) |
| `MCP_LAUNCH_COMMAND` (launch command del server MCP) | Constante fija `'axiom'`, asumía PATH incondicionalmente | Debe resolverse por máquina | **FIX #3**: `resolveMcpLaunchCommand()` |
| Onboarding de un miembro recién clonado | No existía ningún comando dedicado; `axiom join` por sí solo no bindea repos ni materializa MCP/toolchain, y además (hallazgo en validación viva) su gate bloqueaba proyectos `workspace-setup` | Debe existir un comando headless de un solo paso | **FIX #1**: `axiom member install` (+ fix del gate, ver abajo) |
| Reparación de UN binding sin re-correr todo | No existía | Debe existir | **FIX #2**: `axiom bindings show/set/remove` |

**Hallazgo adicional descubierto en la validación viva** (no estaba en
el audit original, pero es bloqueante para el goal): el gate del
orchestrator `hasInitJson` (`@axiom/orchestrator`, `state-machine.ts`)
exige `.axiom-state/<projectName>/init.json` para `join-command`. Sólo
`axiom init` lo escribe; `axiom workspace setup` (el flujo multi-repo
que usa el ARQUITECTO) no — y aunque lo escribiera, `init.json` vive
bajo `.axiom-state/`, que es PERSONAL/gitignored, así que el del
arquitecto nunca llegaría al clone de un miembro vía git. Sin
corregirlo, `axiom join`/`axiom member install` no podían completarse
NUNCA sobre un proyecto bootstradeado vía `workspace setup` — exactamente
el escenario multi-repo que este incremento existe para soportar. Fix:
`member-install.ts#ensureInitJsonForJoin` sintetiza un `init.json` local
mínimo (best-effort, hereda `profile`/`overlay`/adapter real de
`workspace.json` si existe) ANTES de invocar `runJoin`, nunca clobberea
uno preexistente. Deliberadamente LOCAL a `member-install.ts` — no se
tocó el gate compartido del orchestrator (ver Non-goals).

### `axiom member install`

`apps/cli/src/commands/member-install.ts`. Pasos, todos best-effort
salvo el 0 (resolución de proyecto, que si falla lanza con mensaje
claro):

0. `ensureInitJsonForJoin` (ver arriba).
1. `runJoin` (reuso literal, `--member` pasa tal cual).
2. Bindings: `--bind repoId:path` (repetible, parseo tolerante a
   `:` de drive letters de Windows) tiene prioridad; si no, un binding
   previo persistido; si no, auto-detección vía `resolveRepoPath` (ref
   fallback) + `fs.existsSync` — sólo se PERSISTE lo explícito o lo
   previamente persistido, nunca lo auto-detectado (ya cubierto por el
   fallback de `resolveRepoPath`, persistirlo sería redundante). Repos
   sin resolver quedan en `unbound` con la remediación exacta.
3. MCP: lee `axiom.config/mcp-manifest.yaml` (committeado), mapea
   `id: 'sdd'`/`id: 'spec'` a `SDD_MCP_SERVER_ID`/`SPEC_MCP_BROKER_ID`
   (los únicos dos kinds que `axiom mcp serve` sabe lanzar; cualquier
   otro id genera un warning, nunca una excepción), construye el
   launch command vía `resolveMcpLaunchCommand()` (override inyectable
   para tests/operadores avanzados), y escribe el config nativo
   (`writeNativeMcpConfig`) en control + spec para cada adapter
   declarado en `workspace.json#adapters` si existe localmente, o para
   los 5 `NATIVE_MCP_TARGETS` completos si no (best-effort, inocuo:
   merge-preserving).
4. Toolchain: `loadToolchain` + `repairTool` por cada tool de
   `axiom.config/toolchain.yaml`; guía de instalación externa honesta
   (`EXTERNAL_INSTALL_GUIDANCE`, sólo comandos/URLs YA documentados en
   el codebase — `serena` → `uv tool install -p 3.13 serena-agent`,
   sourced de `packages/providers/src/code-intel/serena-client.ts`;
   para el resto, mensaje explícito de "no se conoce un comando
   automatizado").
5. Resumen humano (`summaryLines`) distinguiendo SHARED de PERSONAL.

Soporta `--json`, `--home-dir <path>` + `$AXIOM_HOME_DIR`,
`mcpLaunchCommandOverride` (programático, para tests deterministas).

### `resolveMcpLaunchCommand` (BUG-2)

`workspace-mcp.ts`: chequeo real de PATH (`isAxiomOnPath`, con
extensiones ejecutables de Windows) → si no resuelve, `node
<process.argv[1]>` (el entrypoint REALMENTE en ejecución, sin adivinar
dónde se instaló) → último recurso `axiom` (preserva el comportamiento
previo). `buildWorkspaceMcpServers` ganó un 4to parámetro OPCIONAL
`launchCommand` con default `{command: MCP_LAUNCH_COMMAND, baseArgs:
[]}` — preserva EXACTAMENTE el comportamiento previo para los ~15
call sites de test existentes que no pasan uno explícito.
`runWorkspaceSetup` (`WorkspaceSetupSpec.mcpLaunchCommandOverride`,
default = resolver real) y `member-install.ts` lo usan.

### `buildGitignore()` (BUG, fix #4)

`init.ts`: `.axiom-state/local/` → `.axiom-state/` (dir completo),
alineado con el `.gitignore` real, a mano, del propio repo `Axiom/`.

### `axiom bindings`

`apps/cli/src/commands/bindings.ts`, patrón idéntico a
`topology.ts`/`roles.ts` (sin `runOrchestrated`, `runX` separado de
`registerX`). `show` es siempre exit 0; `set`/`remove` son
idempotentes.

## Validation

- `npm run build` (`tsc -b`): limpio, exit 0.
- `npx vitest run` (suite completa): **211 archivos / 2239 tests, 0
  fallos** (baseline previo: 209/2221 — el delta son los 2 archivos de
  test nuevos, `bindings.test.ts` (6 tests) y `member-install.test.ts`
  (6 tests), más 6 tests nuevos en `workspace-mcp.test.ts` para el
  resolver/el nuevo parámetro).
  - Tests existentes que hardcodeaban `command: 'axiom'` a través de
    `runWorkspaceSetup` (`workspace-mcp.test.ts`'s `makeBaseSpec`, el
    e2e `workspace-mcp.e2e.test.ts`) se actualizaron para pasar
    `mcpLaunchCommandOverride` explícito — determinismo, no
    encubrimiento: el resolver en sí tiene su propia cobertura
    dedicada e inyectable.
  - `init.test.ts` actualizado: `.gitignore` ahora se verifica contra
    `.axiom-state/` (no `.axiom-state/local/`).
- **Simulación viva de dos actores** (scratchpad, `HOME`/`USERPROFILE`
  sandboxeados a directorios TEMP — nunca se tocó el `~/.axiom` real):
  - **(a) ARQUITECTO**: `axiom workspace setup --name demo-app
    --control-path <sdd> --spec-path <spec> --role backend:<path>
    --role frontend:<path> --adapters claude-code,opencode` con
    `USERPROFILE`/`HOME` apuntando a un home sandboxeado → éxito,
    4 repos creados, `topology.yaml` con multi-repo real. Se copiaron
    manualmente `mcp-manifest.yaml`/`toolchain-catalog.yaml` (del repo
    `Axiom/` real) + un `toolchain.yaml` con `serena` habilitado, al
    repo de control (simulando el bootstrap de producto del
    arquitecto). Verificado: `.gitignore` generado contiene
    `.axiom-state/` (no sólo `local/`) y `.axiom/`; `axiom.config/`
    contiene todos los artefactos SHARED sin ninguno cubierto por el
    `.gitignore`.
  - **(b) MEMBER**: clonado simulado copiando, de cada uno de los 4
    repos, todo EXCEPTO `.axiom-state/`/`.axiom/`/`node_modules/`/
    `dist/` (lo que un `git clone` real traería, dado el `.gitignore`
    generado) a una ubicación NUEVA. Corrido `axiom member install
    --member alice --bind backend:<path> --bind frontend:<path>` con
    `USERPROFILE`/`HOME` sandboxeados a un home DISTINTO del
    arquitecto, y con `PATH` restringido a excluir `axiom` (para
    forzar y probar el fallback de BUG-2). Resultado real:
    - `mcp.launchCommand` = `{command: 'node', baseArgs:
      ['C:\...\apps\cli\dist\index.js']}` (fallback en acción — con
      `axiom` en PATH, en una corrida separada, resolvió a `{command:
      'axiom', baseArgs: []}`).
    - `.mcp.json` escrito en control Y spec con ese launch command y
      `--project-root` apuntando al path REAL de cada repo en la
      máquina del miembro.
    - `.axiom-state/local/topology-bindings.yaml` con `backend`/
      `frontend` apuntando a los paths del `--bind`.
    - `members.yaml` con `alice` agregada.
    - `.axiom-state/demo-app/toolchain/serena/` creado (estado local
      activado); guía impresa: `serena: uv tool install -p 3.13
      serena-agent`.
    - Re-corrida del MISMO comando: `join.added: false`
      ("ya estaba registrado"), mismos archivos, sin duplicar nada —
      **idempotente confirmado en vivo**.
  - **(c) `axiom bindings show`** en el clone del miembro: listó los 4
    repos con `source`/`exists` correctos. Simulado un "miembro movió
    el repo backend" (mover el directorio) + `axiom bindings set --repo
    backend --path <nueva-ubicación>` → `bindings show` posterior
    refleja el nuevo path como `local-override`, `exists: true`.
  - `--json` verificado en ambos comandos: shape estable, parseable.

## Result

Cerrado. Los tres fixes pedidos explícitamente en el brief
(`buildGitignore`, `resolveMcpLaunchCommand`, `axiom member
install`/`axiom bindings`) están implementados, testeados (unitarios +
vivos) y documentados. Un cuarto fix, no anticipado en el brief pero
descubierto en la validación viva y bloqueante para el goal (el gate
`hasInitJson` del orchestrator impedía completar `join`/`member
install` sobre proyectos `workspace-setup`), se resolvió localmente en
`member-install.ts` sin tocar código compartido del orchestrator. Un
hallazgo del audit (configs MCP nativos no gitignoreados) se documentó
explícitamente como no corregido, con justificación, en vez de
implementarse especulativamente.

## General spec integration

Integrado en `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
("Separación de responsabilidades: instalación del ARQUITECTO vs.
MIEMBRO") — tabla shared/personal + los 5 hallazgos del audit — y en
`05_Interfaces_Operativas.md` (superficie CLI de `axiom member
install`/`axiom bindings`). No se copió el detalle de implementación
línea por línea; sólo el modelo de datos/responsabilidad estable y la
superficie de comandos, consistente con el resto de esos dos ficheros.
