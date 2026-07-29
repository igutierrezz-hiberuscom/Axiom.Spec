# Instalación y onboarding

Fuente: `Axiom/docs/installation.md`, `Axiom/docs/first-project-readiness.md`, `Axiom/docs/configuration/files/onboarding.md`, `Axiom/scripts/verify-first-project-readiness.mjs`, `Axiom/apps/cli/src/commands/workspace-setup.ts`, `workspace-adopt.ts`, `member-install.ts`.

> Reconciliado 2026-07-29: sección "Nota crítica de estado real" (más abajo) actualizada — la brecha que describía está RESUELTA — y se añadió el flujo de onboarding multi-repo (`axiom workspace setup`/`adopt` + `member install`) que no existía en el baseline 2026-07-02.

## Instalación user-level del binario

El binario `axiom` se instala una vez por operador (no por proyecto) y queda disponible desde cualquier terminal. Contrato de home user-level: `@axiom/user-workspace` (`UserWorkspacePaths.installPath`).

Convenciones de PATH:
- macOS/Linux: `npm prefix -g` o `$HOME/.local/bin`.
- Windows: `%USERPROFILE%\.local\bin`.

Script oficial: `scripts/install-global.mjs`.
- `--install`: materializa shim (Windows) o `npm link` (macOS/Linux); idempotente.
- `--uninstall`: revierte; idempotente.
- `--install-tool <toolId>`: **siempre rechazado** — las tools se gestionan por proyecto vía `axiom toolchain add`.

Comportamiento Windows: crea `%USERPROFILE%\.local\bin\` si falta; si `axiom.cmd` ya existe, no lo sobreescribe (exit 0, "ya instalado"); si no existe, lo crea apuntando a `node "<repo>\Axiom\apps\cli\dist\index.js" %*`; corre un smoke post-install y hace rollback (borra el shim, exit 1) si falla.

## `axiom self-update`

Gestiona la versión user-level, separada de `axiom upgrade` (project-scoped). Manifest: `~/.axiom/install.json`.

- `--check`: reporta versión, persiste manifest (sin mutar binario).
- `--target-version <X.Y.Z>`: preview puro, sin mutación.
- `--target-version <X.Y.Z> --apply`: ejecuta update real vía `install-global.mjs --install`.
- Sin flags: placeholder que fuerza ser explícito (no-op).

GATE crítico: el update NO se aplica sin `--apply`. En Windows, antes de aplicar hace backup del shim (`axiom.cmd.bak`); si el install falla, restaura el backup.

## Flujo feliz de onboarding de un proyecto

```bash
npx axiom init --yes --name mi-proyecto --profile builder --overlay local-only --target opencode
npx axiom join --member user:alice
npx axiom configure
npx axiom sync
npx axiom start
npx axiom doctor
```

## `onboarding.yaml` (contrato declarativo)

- `init`: preguntas, defaults, valores permitidos, documentos generados.
- `join`: qué lee y qué puede escribir localmente; prohíbe mutación documental compartida.
- `doctor`/`configure`/`start`: requisitos, validaciones, superficies refrescables.
- `generatedDocs`: mapea outputs documentales a templates.
- `repairPlaybooks`: qué hacer ante proyecto ambiguo, provider faltante, docs generadas ausentes.

## First-project readiness

La readiness inicial no es solo "compila y pasan los tests unitarios": valida la secuencia operativa completa.

Script ejecutable: `npm run readiness:first-project` (`Axiom/scripts/verify-first-project-readiness.mjs`). Siembra un proyecto temporal con la baseline canónica copiada, vía `seedCanonicalBaseline()`, desde `axiom.config/` (renombrado; antes `axiom.spec` + subcarpeta `config`), `axiom.spec/templates/`, `axiom.spec/target-axiom-skills/`, `axiom.spec/target-axiom-agents/`, `AGENTS.md` y `axiom.skills.lock` del propio repo `Axiom/` (`_builder/` se crea vacío, no se copia), y ejecuta:

```
init → configure → toolchain repair → sync → start → gateway start → gateway status → audit → doctor
```

Falla si algún paso devuelve exit code ≠ 0, falta un artefacto esperado, o `gateway status` no reporta `state: active`.

**Nota de estado real (RESUELTA, actualizada 2026-07-29)**: el baseline 2026-07-02 de este documento advertía que `axiom.config/`, `axiom.spec/templates/`, `axiom.spec/target-axiom-skills/`, `axiom.spec/target-axiom-agents/`, `AGENTS.md` y `axiom.skills.lock` no existían en la raíz de `Axiom/` y que el script fallaría con `ENOENT`. Verificado por listado directo 2026-07-29: `Axiom/axiom.config/`, `Axiom/axiom.spec/`, `Axiom/AGENTS.md` y `Axiom/axiom.skills.lock` **ya existen** (`INC-20260708-product-repo-self-bootstrap` + reconciliaciones posteriores). Único gap residual menor: `Axiom/_builder/` sigue sin existir (el script lo crea vacío en el proyecto temporal, no depende de que exista en `Axiom/`). Detalle en [../references/03-riesgos-y-brechas-conocidas.md](../references/03-riesgos-y-brechas-conocidas.md).

## Baseline recomendada y checklist manual

Triple recomendado: `functionalProfile: builder`, `operationalOverlay: local-only`, `adapterTarget: opencode` (motivo: perfil más cubierto en runtime, menor fricción, target más estable).

Checklist manual de equipo:
1. `npm run build` verde en `Axiom/`.
2. `npm test` verde en `Axiom/`.
3. `node ../axiom.spec/scripts/doctor-validate-contracts.mjs` verde desde la raíz del repo. **Gap residual verificado 2026-07-29**: `Axiom/axiom.spec/` ya existe (contiene `increments/`, `plans/`, `target-axiom-agents/`, `target-axiom-skills/`, `templates/`), pero NO tiene subcarpeta `scripts/` — este paso del checklist sigue sin script real que ejecutar; no confundir con la brecha más amplia ya resuelta del punto anterior.
4. `npm run readiness:first-project` en `PASS`.
5. Documentación operativa navegable (instalación, uso diario, CLI, troubleshooting, esta guía).

## Onboarding multi-repo (`axiom workspace setup` / `axiom workspace adopt` / `member install`)

Añadido desde el baseline 2026-07-02 (varias oleadas post-0030, consolidadas por `INC-20260727-adoption-config-scaffolding` y anteriores). `runWorkspaceSetup` (`apps/cli/src/commands/workspace-setup.ts`) es el motor CORE compartido, sin TUI ni MCP, detrás de dos flujos:

- **`axiom workspace setup`**: scaffolding aditivo de un proyecto/repo de control ya existente (no requiere legacy repos previos).
- **`axiom workspace adopt`** (`workspace-adopt.ts`): adopción de un repo/proyecto legacy hacia el modelo Axiom; llama a `runWorkspaceSetup` tras confirmación explícita para la parte de scaffolding (Subject B), y añade por separado la migración basada en detectores (Subject A) — no cambia la firma de `runWorkspaceSetup` (parametrización puramente aditiva).
- **`member install`** (`member-install.ts`): instalación por miembro de equipo en un repo multi-rol ya adoptado/configurado.

Ambos flujos de `workspace setup`/`adopt` siembran, best-effort y no-clobber, los 4 artefactos de config que antes ningún comando producía automáticamente: `axiom.config/integrations.yaml` (PC-001), `axiom.config/policy-as-code.yaml` (PC-002), `axiom.config/agents-catalog.yaml` (TC-011) y el `axiom.skills.lock` raíz (GC-001/GC-002/GC-007) — ver `../architecture/02-modelo-de-datos-y-configuracion.md`.

## Fuera de la baseline inicial (no-goals explícitos del MVP)

Overlays `standard`/`enterprise` como camino inicial obligatorio; `visual-studio-2026` como baseline de primer arranque; providers post-MVP (`engram`, `cmm`) como requisito de entrada; `cmm` es el reemplazo vigente de `codegraph`/`graphify` (ver `../integrations/01-capabilities-providers-y-toolchain.md`); bridges externos/plugins/lanes paralelos avanzados; instalación user-level del binario como paso obligatorio.
