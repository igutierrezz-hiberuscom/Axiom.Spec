# Adapters y model routing

Fuente: `Axiom/packages/adapters/README.md`, `Axiom/docs/cli/support-matrix.md`, `@axiom/model-routing`, `@axiom/launcher`, `apps/cli/src/commands/native-mcp-config.ts`.

> Reconciliado por ACC-025..ACC-029 (2026-08-09). El runtime mantiene 8
> targets activos con 8 packages dedicados; `copilot-vscode` es alias legacy de
> `github-copilot` y LiteLLM fue retirado.

## Contrato común de adapters

Cada adapter target expone `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`. `AdapterGeneratorError` es una unión discriminada con kinds `template-missing | io-error` (+ `invalid-skills-registry` para adapters que emiten lockfile). El generator **no** recalcula `ResolvedInstallProfile`: lo recibe ya materializado en `args.resolvedProfile`.

```ts
import { generateOpencodeConfig } from '@axiom/adapters-opencode';

const result = await generateOpencodeConfig({
  projectRoot: '/ruta/al/proyecto',
  projectName: 'mi-proyecto',
  resolvedProfile /* ResolvedInstallProfile */,
  skills: [],
});
```

## Registro activo de targets (8)

Fuente única reconciliada en `apps/cli/src/commands/init.ts#ADAPTER_TARGETS`, `Axiom/axiom.yaml#capabilities.adapters` (los 8 headline), `@axiom/model-routing#SUPPORT_MATRIX`/`MVP_TARGETS` e `@axiom/install-profiles`:

`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `codex`,
`antigravity`, `visual-studio-2026`.

`DEFAULT_TARGET` sigue siendo `opencode`. `copilot-vscode` solo se admite
durante la migracion de estados y se normaliza a `github-copilot`; LiteLLM no
se acepta como target activo.

## Adapters con package dedicado (8, todos operativos)

| Target | Package | Nivel de soporte | Superficie de instrucciones |
|---|---|---|---|
| `opencode` | `@axiom/adapters-opencode` | `multi-mode` | `.opencode/AGENTS.md` + `skills-lock.yaml` (referencia) |
| `claude-code` | `@axiom/adapters-claude-code` | `single-mode` | `.claude/AGENTS.md` |
| `github-copilot` | `@axiom/adapters-github-copilot` | `fallback-only` | instrucciones Copilot |
| `vscode` | `@axiom/adapters-vscode` | `fallback-only` | superficie VS Code |
| `cursor` | `@axiom/adapters-cursor` | `fallback-only` | superficie Cursor |
| `codex` | `@axiom/adapters-codex` | `fallback-only` | `.codex/AGENTS.md` (INC-20260726) |
| `antigravity` | `@axiom/adapters-antigravity` | `fallback-only` | `.antigravity/AGENTS.md` (INC-20260726) |
| `visual-studio-2026` | `@axiom/adapters-visual-studio-2026` | `fallback-only` | alias de `.github/copilot-instructions.md` |

`codex` y `antigravity` usan generators single-file dedicados. Visual Studio
conserva su package por compatibilidad, pero delega en el writer común de
Copilot y no genera `.vs/AXIOM.md`.

## Targets declarados sin adapter dedicado

Solo `copilot-vscode`: no tiene package propio pero comparte el schema nativo de VS Code con `github-copilot`/`vscode`. Cae a `fallback-only` en `SUPPORT_MATRIX`.

## Superficie común de instrucciones Copilot (ACC-024)

GitHub Copilot, Copilot CLI, VS Code y Visual Studio comparten el documento
general `.github/copilot-instructions.md`. `copilot-vscode` es alias legacy;
`vscode` reserva `.vscode/` para `settings.json`, `extensions.json` y
`mcp.json`. Las process surfaces por ruta usan
`.github/instructions/*.instructions.md`.

El writer común vive en `@axiom/document-bootstrap` y recibe el template
versionado del proyecto con fallback bundleado. Reemplaza solo el bloque
`AXIOM:GENERATED`, conserva byte a byte el preámbulo, la cola humana y
`TEAM:CUSTOM`, y migra `.vscode/copilot-instructions.md` solo después de
escribir y confirmar el destino. Si las zonas humanas divergen, conserva la
fuente legacy y devuelve una advertencia. La selección conjunta de aliases se
deduplica de forma determinista.

## Superficie portable de skills/agents (adapter-agnóstica)

`workspace-process-surfaces.ts#materializeProcessSurfaces` escribe SIEMPRE, independientemente del adapter, `.axiom/agents/<id>.md`, `.axiom/commands/<id>.md` y `.axiom/skills/<id>/SKILL.md`. Es el mínimo garantizado de descubribilidad: cualquier adapter (incluidos codex/antigravity/vs2026) encuentra sus skills/agents ahí, además de en su superficie nativa si la tiene. La materialización nativa por-comando (`.claude/commands`, `.opencode/agents`) existe hoy solo para claude-code y opencode; para el resto es un refinamiento diferido.

## Config MCP nativo (`native-mcp-config.ts`)

`writeNativeMcpConfig` proyecta servers managed de Axiom solo después de que
`filterProjectBoundMcpServers` confirme el proyecto, `mcp.yml`, el manifest,
`enabled` y `targetRepo`. Los writers reconcilian IDs gestionados y preservan
servidores custom:

| Target | Fichero nativo | Shape | Estado |
|---|---|---|---|
| `claude-code` | `.mcp.json` | `{ mcpServers }` | verificado |
| `cursor` | `.cursor/mcp.json` | `{ mcpServers }` | verificado |
| `github-copilot` / `vscode` | `.vscode/mcp.json` | `{ servers, type:'stdio' }` | verificado |
| `opencode` | `opencode.json` | `{ mcp, type:'local' }` | verificado |
| `visual-studio-2026` | — | — | sin schema MCP verificado; warning, no escritura |
| `codex` / `antigravity` | — (ninguno) | nota informativa | config USER-GLOBAL (`~/.codex/config.toml` `[mcp_servers]` / `~/.gemini/config/mcp_config.json`); no se escribe fichero de proyecto |

`NATIVE_MCP_TARGETS` = `claude-code`, `cursor`, `github-copilot`, `opencode` y
`vscode`. `NATIVE_MCP_INFORMATIVE_TARGETS` = `codex`, `antigravity`; no se
escribe ni recomienda configuración global sin binding seguro. Visual Studio
queda fuera por schema no verificado. Disciplina invariante: **nunca se inventa
un schema no verificado**.

## Routing de adapters en el launcher (`@axiom/launcher#AXIOM_ADAPTER_ROUTING`)

Tabla de 9 entradas (los 8 headline + `cli`). Cada headline rutea sus acciones al skill real (`axiom-sdd-orchestrator`, o `axiom-phase-reviewer` para `review-*`) y al mcp-tool `sdd.transitionApply`; `cli` rutea a la invocación `axiom <cmd>` real. `apiGetLauncherData` los ofrece todos en el selector del front con etiquetas amigables (`apps/cli/src/commands/_adapter-labels.ts#ADAPTER_LABELS`). Ningún headline cae al fallback de adapter-desconocido.

El prompt generado (`prompt-builder.ts`) incluye un bloque adapter-neutral **"Herramientas y ubicación"**: servers MCP disponibles, el mcp-tool de mutación confirmada, el skill a aplicar y las rutas de spec/metadata resueltas (`resolveArtifactDir`). El bloque es idéntico entre adapters skill-ruteados; `cli` omite la línea de skill.

## GATE 0031 / cobertura de adapters en doctor

`@axiom/doctor` corre `TC-009-adapter-runtime-coverage`: los **8** packages activos `@axiom/adapters-<target>` deben tener `src/generator.ts` y `dist/index.js` materializados; si falta alguno, el doctor falla. La support matrix refleja comportamiento real verificado, no aspiracional. (ACC-025 retiró `litellm`; el conteo pasó de 9 a 8.)

### Probes de runtime (opt-in, `axiom doctor --deep` / launcher `?deep=1`)

`runDoctorChecksDeep` (`packages/doctor/src/deep-checks.ts`) añade, sin modificar el doctor síncrono y **sin poder nunca hacer FAIL** (solo `pass`/`warn`/`skip`):

- **TC-018** (tool functional): corre `--version` para los tools con contrato de probe (`serena`, `cmm`→`codebase-memory-mcp`, `engram`); `skip` honesto para los skill-invoked sin binario (`rtk`, `caveman`).
- **TC-019** (MCP liveness): descubrimiento `server/discover` MCP `2026-07-28` REAL contra `axiom-mcp-broker` (reutiliza `createStdioMcpClient` de `@axiom/providers`), leyendo `command`/`args` de `.axiom/mcp.yml`. `warn` si no hay `.axiom/mcp.yml`, si el comando no resuelve o si el descubrimiento expira.

Ambos usan probers inyectables (los tests nunca lanzan procesos reales).

## Model routing (`@axiom/model-routing`)

Resuelve, por slot operativo (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`), qué clase de modelo (`cheap`, `medium`, `strong`, `local`) aplica, combinando policy base + overrides project-scoped + `SupportLevel` del target.

### Interpretación de support levels

- **`multi-mode`** (solo `opencode` hoy): respeta routing por slot; los 7 slots reciben su `ModelClass` propia; `axiom model set` funciona completo.
- **`single-mode`** (`claude-code`): no soporta per-slot routing; todos los slots caen a `medium` con `fallbackReason: per-slot-routing-unsupported`; `axiom model validate` lo reporta como `warn`, no `fail`.
- **`fallback-only`** (resto, incluidos `codex`/`vscode`): sin routing alguno; mismo fallback a `medium`; también `warn`, no `fail`.

### Checks de drift (`axiom model validate`)

| Check | Alcance | Falla cuando |
|---|---|---|
| MRC-001 | `model-routing-policy.yaml` | no existe |
| MRC-002 | YAML bien formado | malformado o schema incorrecto |
| MRC-003 | `model-assignments.json` | estado inválido (ausencia total es válida) |
| MRC-004 | `SupportLevel` del target | target no es `multi-mode` (**warn**, no fail) |

### Projection a Opencode

Si el target es `opencode`, `model validate` escribe `<root>/.opencode/model-routing.json` con `project`, `target`, `supportLevel`, `slots` (cada uno con `modelClass`, `fellBack`, `fallbackReason`), `projectedAt`. Opcional: si el archivo no existe, el adapter opencode cae a sus defaults internos.
