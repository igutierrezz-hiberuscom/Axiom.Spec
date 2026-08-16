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
npx axiom init --yes --name mi-proyecto --target opencode
npx axiom join --member user:alice
npx axiom configure
npx axiom sync
npx axiom start
npx axiom doctor
```

`axiom init` escribe `axiom.yaml`, `AGENTS.md` canónico (aditivo, best-effort),
`.gitignore`, `.axiom-state/local/` y `.axiom-state/<projectKey>/init.json`, y
además intenta registrar el proyecto en el registry user-level de forma
best-effort con opt-out `--no-register`.

`axiom configure` usa `DEFAULT_PROFILES` únicamente cuando
`axiom.config/profiles.yaml` está ausente. Si el archivo existe pero no se
puede leer, contiene YAML inválido o no cumple `InstallProfilesYamlSchema`,
devuelve `invalid-profiles-yaml` y no continúa con una configuración por
defecto silenciosa. Un override válido se usa sin modificar el archivo de
origen.

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
init → configure → toolchain repair → sync → start → audit → doctor
```

Falla si algún paso devuelve exit code ≠ 0 o falta un artefacto esperado.

**Nota de estado real (RESUELTA, actualizada 2026-07-29)**: el baseline 2026-07-02 de este documento advertía que `axiom.config/`, `axiom.spec/templates/`, `axiom.spec/target-axiom-skills/`, `axiom.spec/target-axiom-agents/`, `AGENTS.md` y `axiom.skills.lock` no existían en la raíz de `Axiom/` y que el script fallaría con `ENOENT`. Verificado por listado directo 2026-07-29: `Axiom/axiom.config/`, `Axiom/axiom.spec/`, `Axiom/AGENTS.md` y `Axiom/axiom.skills.lock` **ya existen** (`INC-20260708-product-repo-self-bootstrap` + reconciliaciones posteriores). Único gap residual menor: `Axiom/_builder/` sigue sin existir (el script lo crea vacío en el proyecto temporal, no depende de que exista en `Axiom/`). Detalle en [../references/03-riesgos-y-brechas-conocidas.md](../references/03-riesgos-y-brechas-conocidas.md).

## Baseline recomendada y checklist manual

Configuración efectiva: `builder` + `local-only` implícitos y `adapterTarget: opencode` como target inicial de menor fricción.

Checklist manual de equipo:
1. `npm run build` verde en `Axiom/`.
2. `npm test` verde en `Axiom/`.
3. `npm run doctor` verde desde la raíz de `Axiom/`. El script histórico `../axiom.spec/scripts/doctor-validate-contracts.mjs` ya no forma parte del árbol actual; la validación operativa vigente usa el comando del package.
4. `npm run readiness:first-project` en `PASS` como criterio de aceptación del flujo.
5. Documentación operativa navegable (instalación, uso diario, CLI, troubleshooting, esta guía).

La checklist de readiness debe interpretarse junto con el estado actual de
Doctor: `CC-004` cubre 13/16 capabilities provider-routed y las tres opcionales
restantes producen warning no bloqueante. Las capabilities MCP-only se validan
por su superficie MCP.

**Registro histórico de ejecución (2026-07-30; superado el 2026-08-02):** el
flujo no alcanzó `PASS` porque su paso `doctor` falló en `TC-011` por un
`bundleHash` stale de `axiom-reviewer`. La causa era ajena al versionado de
toolchain; los checks TC-020..TC-023 no fallaron por el incremento.

**Registro histórico verificado el 2026-08-02:** `npm run doctor` terminó en
`PASS` (46/61 OK, 0 fallos, 3 advertencias, 12 omitidos) y
`npm run readiness:first-project` terminó en `PASS`. Esta fotografía queda
superada por R-04: la cobertura canónica actual de `CC-004` sirve 13/16 y deja
warning por las tres capabilities opcionales sin provider.

## Onboarding multi-repo (`axiom workspace setup` / `axiom workspace adopt` / `member install`)

Añadido desde el baseline 2026-07-02 (varias oleadas post-0030, consolidadas por `INC-20260727-adoption-config-scaffolding` y anteriores). `runWorkspaceSetup` (`apps/cli/src/commands/workspace-setup.ts`) es el motor CORE compartido, sin interfaz TUI y sin lógica de negocio de MCP, detrás de dos flujos:

- **`axiom workspace setup`**: scaffolding aditivo de un proyecto/repo de control ya existente (no requiere legacy repos previos).
- **`axiom workspace adopt`** (`workspace-adopt.ts`): adopción de un repo/proyecto legacy hacia el modelo Axiom; llama a `runWorkspaceSetup` tras confirmación explícita para la parte de scaffolding (Subject B), y añade por separado la migración basada en detectores (Subject A) — no cambia la firma de `runWorkspaceSetup` (parametrización puramente aditiva).
- **`member install`** (`member-install.ts`): instalación por miembro de equipo en un repo multi-rol ya adoptado/configurado.

Ambos flujos de `workspace setup`/`adopt` siembran, best-effort y no-clobber, los 4 artefactos de config que antes ningún comando producía automáticamente: `axiom.config/integrations.yaml` (PC-001), `axiom.config/policy-as-code.yaml` (PC-002), `axiom.config/agents-catalog.yaml` (TC-011) y el `axiom.skills.lock` raíz (GC-001/GC-002/GC-007) — ver `../architecture/02-modelo-de-datos-y-configuracion.md`.

El launcher web (`axiom app`) expone estos flujos con endpoints server-level
`/api/launcher/workspace/setup` y `/api/launcher/workspace/adopt`. La preview
no escribe; la confirmación delega en los mismos runners. Antes de adoptar se
rechazan destinos con `axiom.yaml` de otro proyecto o identidad desconocida y
paths de roles iguales/anidados. El resultado devuelve paths, created/skipped,
warnings, provenance, conformance y registry real; una adopción parcial se
representa como resultado parcial, no como error opaco. `axiom app` es la
interfaz guiada vigente; no existe una TUI pública equivalente.

## Fuera de la baseline inicial (no-goals explícitos del MVP)

Como contexto histórico, los overlays `standard`/`enterprise` no formaron parte del camino inicial de readiness; hoy están retirados. También quedan fuera de la baseline inicial `visual-studio-2026` como target de primer arranque, exigir `engram`/`cmm` como requisito de entrada, bridges externos/plugins/lanes paralelos avanzados e instalación user-level del binario como paso obligatorio. `cmm` es el reemplazo vigente de `codegraph`/`graphify` (ver `../integrations/01-capabilities-providers-y-toolchain.md`).
