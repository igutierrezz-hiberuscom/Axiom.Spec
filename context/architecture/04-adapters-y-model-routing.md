# Adapters y model routing

Fuente: `Axiom/packages/adapters/README.md`, `Axiom/docs/cli/support-matrix.md`, `@axiom/model-routing`, `@axiom/launcher`, `apps/cli/src/commands/native-mcp-config.ts`.

> Actualizado por la tanda `INC-20260726-*` (paridad de adapters + front de onboarding + doctor deep). Antes había 6 packages de adapter y 3 targets sin package; hoy hay **9 packages** y **10 targets canónicos**.

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

## Registro canónico de targets (10)

Fuente única reconciliada en `apps/cli/src/commands/init.ts#ADAPTER_TARGETS`, `Axiom/axiom.yaml#capabilities.adapters` (los 8 headline), `@axiom/model-routing#SUPPORT_MATRIX`/`MVP_TARGETS` e `@axiom/install-profiles`:

`opencode`, `copilot-vscode`, `claude-code`, `antigravity`, `visual-studio-2026`, `cursor`, `github-copilot`, `litellm`, `vscode`, `codex`.

`DEFAULT_TARGET` sigue siendo `opencode`. Los 8 **headline** (los que el launcher ofrece y el `axiom.yaml` lista) son: `claude-code`, `github-copilot`, `vscode`, `opencode`, `cursor`, `antigravity`, `visual-studio-2026`, `codex`. `copilot-vscode` y `litellm` quedan como targets internos reconocidos.

## Adapters con package dedicado (9, todos operativos)

| Target | Package | Nivel de soporte | Superficie de instrucciones |
|---|---|---|---|
| `opencode` | `@axiom/adapters-opencode` | `multi-mode` | `.opencode/AGENTS.md` + `skills-lock.yaml` (referencia) |
| `claude-code` | `@axiom/adapters-claude-code` | `single-mode` | `.claude/AGENTS.md` |
| `github-copilot` | `@axiom/adapters-github-copilot` | `fallback-only` | instrucciones Copilot |
| `vscode` | `@axiom/adapters-vscode` | `fallback-only` | superficie VS Code |
| `cursor` | `@axiom/adapters-cursor` | `fallback-only` | superficie Cursor |
| `litellm` | `@axiom/adapters-litellm` | `fallback-only` | router LiteLLM (P1 pendiente) |
| `codex` | `@axiom/adapters-codex` | `fallback-only` | `.codex/AGENTS.md` (INC-20260726) |
| `antigravity` | `@axiom/adapters-antigravity` | `fallback-only` | `.antigravity/AGENTS.md` (INC-20260726) |
| `visual-studio-2026` | `@axiom/adapters-visual-studio-2026` | `fallback-only` | `.vs/AXIOM.md` (INC-20260726) |

`codex`, `antigravity` y `visual-studio-2026` pasaron de placeholder fino (`writeThinCanonicalAgentsMd`, hoy código muerto) a **generadores de primera clase** que mirroran el patrón single-file de `claude-code`.

## Targets declarados sin adapter dedicado

Solo `copilot-vscode`: no tiene package propio pero comparte el schema nativo de VS Code con `github-copilot`/`vscode`. Cae a `fallback-only` en `SUPPORT_MATRIX`.

## Superficie portable de skills/agents (adapter-agnóstica)

`workspace-process-surfaces.ts#materializeProcessSurfaces` escribe SIEMPRE, independientemente del adapter, `.axiom/agents/<id>.md`, `.axiom/commands/<id>.md` y `.axiom/skills/<id>/SKILL.md`. Es el mínimo garantizado de descubribilidad: cualquier adapter (incluidos codex/antigravity/vs2026) encuentra sus skills/agents ahí, además de en su superficie nativa si la tiene. La materialización nativa por-comando (`.claude/commands`, `.opencode/agents`) existe hoy solo para claude-code y opencode; para el resto es un refinamiento diferido.

## Config MCP nativo (`native-mcp-config.ts`)

`writeNativeMcpConfig` proyecta los dos servers managed de Axiom (`sdd-mcp-server`, `spec-mcp-broker`) al schema nativo VERIFICADO de cada tool, merge-preservando lo que el usuario ya tenga:

| Target | Fichero nativo | Shape | Estado |
|---|---|---|---|
| `claude-code` | `.mcp.json` | `{ mcpServers }` | verificado |
| `cursor` | `.cursor/mcp.json` | `{ mcpServers }` | verificado |
| `copilot-vscode` / `github-copilot` / `vscode` | `.vscode/mcp.json` | `{ servers, type:'stdio' }` | verificado |
| `opencode` | `opencode.json` | `{ mcp, type:'local' }` | verificado |
| `visual-studio-2026` | `.vs/mcp.json` | `{ servers, type:'stdio' }` | **ASUNCIÓN DOCUMENTADA** override-able (VS 2022 17.14+/2026 lee MCP a nivel solución; no re-verificado contra VS real). Se eligió `.vs/mcp.json` y no `.mcp.json` raíz para no colisionar con `claude-code`. |
| `codex` / `antigravity` | — (ninguno) | nota informativa | config USER-GLOBAL (`~/.codex/config.toml` `[mcp_servers]` / `~/.gemini/config/mcp_config.json`); no se escribe fichero de proyecto |
| `litellm` | — | warning genérico | sin schema nativo verificado |

`NATIVE_MCP_TARGETS` = los 7 con fichero. `NATIVE_MCP_INFORMATIVE_TARGETS` = `codex`, `antigravity`. Disciplina invariante: **nunca se inventa un schema no verificado** — si no hay schema, se emite una nota, no un fichero.

## Routing de adapters en el launcher (`@axiom/launcher#AXIOM_ADAPTER_ROUTING`)

Tabla de 9 entradas (los 8 headline + `cli`). Cada headline rutea sus acciones al skill real (`axiom-sdd-orchestrator`, o `axiom-phase-reviewer` para `review-*`) y al mcp-tool `sdd.transitionApply`; `cli` rutea a la invocación `axiom <cmd>` real. `apiGetLauncherData` los ofrece todos en el selector del front con etiquetas amigables (`apps/cli/src/commands/_adapter-labels.ts#ADAPTER_LABELS`). Ningún headline cae al fallback de adapter-desconocido.

El prompt generado (`prompt-builder.ts`) incluye un bloque adapter-neutral **"Herramientas y ubicación"**: servers MCP disponibles, el mcp-tool de mutación confirmada, el skill a aplicar y las rutas de spec/metadata resueltas (`resolveArtifactDir`). El bloque es idéntico entre adapters skill-ruteados; `cli` omite la línea de skill.

## GATE 0031 / cobertura de adapters en doctor

`@axiom/doctor` corre `TC-009-adapter-runtime-coverage`: los **9** packages `@axiom/adapters-<target>` deben tener `src/generator.ts` y `dist/index.js` materializados; si falta alguno, el doctor falla. La support matrix refleja comportamiento real verificado, no aspiracional.

### Probes de runtime (opt-in, `axiom doctor --deep` / launcher `?deep=1`)

`runDoctorChecksDeep` (`packages/doctor/src/deep-checks.ts`) añade, sin modificar el doctor síncrono y **sin poder nunca hacer FAIL** (solo `pass`/`warn`/`skip`):

- **TC-018** (tool functional): corre `--version` para los tools con contrato de probe (`serena`, `cmm`→`codebase-memory-mcp`, `engram`); `skip` honesto para los skill-invoked sin binario (`rtk`, `caveman`).
- **TC-019** (MCP liveness): handshake `initialize` JSON-RPC REAL contra `sdd-mcp-server` / `spec-mcp-broker` (reutiliza `createStdioMcpClient` de `@axiom/providers`), leyendo `command`/`args` de `.axiom/mcp.yml`. `warn` si no hay `.axiom/mcp.yml`, si el comando no resuelve o si el handshake expira.

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
