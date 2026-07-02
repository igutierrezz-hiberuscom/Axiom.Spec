# Adapters y model routing

Fuente: `Axiom/packages/adapters/README.md`, `Axiom/docs/cli/support-matrix.md`, `@axiom/model-routing`.

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

## Adapters con package dedicado (6, todos operativos)

| Target | Package | Nivel de soporte | Estado |
|---|---|---|---|
| `opencode` | `@axiom/adapters-opencode` | `multi-mode` | Operativo (referencia) |
| `claude-code` | `@axiom/adapters-claude-code` | `single-mode` | Operativo (0031/A) |
| `github-copilot` | `@axiom/adapters-github-copilot` | `fallback-only` | Operativo (0031/B) |
| `vscode` | `@axiom/adapters-vscode` | `fallback-only` | Operativo (0031/C) |
| `cursor` | `@axiom/adapters-cursor` | `fallback-only` | Operativo (0031/D.1; integración profunda P1 pendiente) |
| `litellm` | `@axiom/adapters-litellm` | `fallback-only` | Operativo (0031/D.2; router LiteLLM P1 pendiente) |

## Targets declarados sin adapter dedicado

`copilot-vscode`, `antigravity`, `visual-studio-2026` están declarados en `profiles.yaml#adapterTargets` con nivel `fallback-only`, pero SIN package de adapter propio: el runtime cae al default del IDE/CLI y el resolver de Axiom no puede influir en su configuración. Esto se honra explícitamente en `@axiom/model-routing#SUPPORT_MATRIX`.

## GATE 0031

La support matrix refleja comportamiento real verificado, no aspiracional. `@axiom/doctor` corre `TC-009-adapter-runtime-coverage`: los 6 packages de adapter deben tener `src/generator.ts` y `dist/index.js` materializados; si falta alguno, el doctor falla.

## Model routing (`@axiom/model-routing`)

Resuelve, por slot operativo (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`), qué clase de modelo (`cheap`, `medium`, `strong`, `local`) aplica, combinando policy base + overrides project-scoped + `SupportLevel` del target.

### Interpretación de support levels

- **`multi-mode`** (solo `opencode` hoy): respeta routing por slot; los 7 slots reciben su `ModelClass` propia; `axiom model set` funciona completo.
- **`single-mode`** (`claude-code`): no soporta per-slot routing; todos los slots caen a `medium` con `fallbackReason: per-slot-routing-unsupported`; `axiom model validate` lo reporta como `warn`, no `fail`.
- **`fallback-only`** (resto): sin routing alguno; mismo fallback a `medium`; también `warn`, no `fail`.

### Checks de drift (`axiom model validate`)

| Check | Alcance | Falla cuando |
|---|---|---|
| MRC-001 | `model-routing-policy.yaml` | no existe |
| MRC-002 | YAML bien formado | malformado o schema incorrecto |
| MRC-003 | `model-assignments.json` | estado inválido (ausencia total es válida) |
| MRC-004 | `SupportLevel` del target | target no es `multi-mode` (**warn**, no fail) |

### Projection a Opencode

Si el target es `opencode`, `model validate` escribe `<root>/.opencode/model-routing.json` con `project`, `target`, `supportLevel`, `slots` (cada uno con `modelClass`, `fellBack`, `fallbackReason`), `projectedAt`. Opcional: si el archivo no existe, el adapter opencode cae a sus defaults internos.
