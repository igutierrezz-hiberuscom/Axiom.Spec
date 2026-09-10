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

La superficie user-level usa `~/.axiom/install.json` como receipt/cache derivado de la instalación real y queda separada de `axiom upgrade` (project-scoped). En R13 las operaciones son:

- `axiom self-update status [--json]`: inspección local de identidad y receipt, sin mutación.
- `axiom self-update check [--json]`: comparación con la release publicada, sin persistir el resultado.
- `axiom self-update plan [--json] [--dry-run]`: plan declarativo read-only.
- `axiom self-update apply [--json] [--plan-file <path>]` y `axiom self-update recover [--json]`: motor transaccional; devuelven `installed`, `unchanged`, `failed` o `recovery-required` y no adoptan una instalación inicial.

La identidad real del entrypoint es la autoridad; `status`, `check` y `plan` no crean locks, temporales, receipts ni migraciones. `--dry-run` solo es válido con `plan`. La selección legacy mediante `--check`, `--apply`, `--recover` o `--target-version` se rechaza con error de uso. La implementación histórica de esos flags no forma parte de la gramática activa R13.

## Catálogo y entrada a proyectos

`axiom init` intenta catalogar el repo en `~/.axiom/projects.yml` de forma best-effort, salvo `--no-register`. El catálogo usa exclusivamente `schemaVersion: 2`; un `registry.json` residual se ignora y nunca se migra o modifica. Si `projects.yml` no existe, la primera alta parte de un catálogo vacío.

- `axiom projects add --path <dir>` cataloga un directorio existente sin exigir que sea Axiom-resoluble; `--id` permite aportar el slug canónico cuando el nombre no deriva a ASCII.
- `axiom projects join --path <dir>` exige que `resolveProject` reconozca el repo antes de catalogarlo. No confundirlo con `axiom join --member <id>`, que registra un miembro dentro del proyecto resuelto.
- `axiom projects list` ordena por recencia e informa disponibilidad física y resolubilidad por separado.
- `axiom projects use <id>` solo actualiza `lastUsedAt`: no activa contexto global. `--role` elige el repo y `--print-path` devuelve únicamente el path; PowerShell puede consumirlo con `Set-Location -LiteralPath (axiom projects use <id> --print-path)` y POSIX con `cd -- "$(axiom projects use <id> --print-path)"`.
- `--json` en `list|add|join|use` devuelve siempre un único envelope versionado, también en error; stdout queda libre de texto humano adicional y el exit code refleja el resultado.

Las altas concurrentes están serializadas por un lock local acotado y reemplazo atómico. Una colisión de identidad o de path no se fusiona ni sobrescribe; una entrada ausente se conserva para diagnóstico en vez de eliminarse automáticamente.

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

El motor compartido `runWorkspaceSetup` (`apps/cli/src/commands/workspace-setup.ts`) opera sobre una autoridad `axiomRepo`, repos `code` y fuentes `legacy` read-only. Lo consumen dos flujos:

- **`axiom workspace setup`**: crea o reconcilia un workspace sin depender del catálogo user-level para resolver la autoridad.
- **`axiom workspace adopt`** (`workspace-adopt.ts`): adopta fuentes legacy tras confirmación y preserva una identidad válida de otro proyecto como `skipped`; un documento inválido, ambiguo o conflictivo se rechaza.
- **`member install`** (`member-install.ts`): instala el overlay personal de un miembro sobre un workspace ya configurado.

Setup/adopt y las altas `repo add`/`role add` calculan primero un plan sin escribir. El preflight valida en conjunto identidad, paths, create flags, ownership y solapamientos; solo `ENOENT` representa ausencia. Apply mantiene locks de proyecto/registry/recursos, recupera journals incompletos, replantea y verifica precondiciones antes de publicar identidad, topología, bindings, `workspace.json`, init state y registro solicitado como una unidad. Un fallo termina en rollback o `recovery-required`, nunca en estructura parcial presentada como éxito; `--no-register` mantiene el catálogo fuera de la unidad persistida.

Adapters, reglas, MCP, skills, catálogos y base de spec se ejecutan después del commit y conservan política no-clobber dentro de sus owners. Una avería derivada se devuelve como warning tipada y no revierte la estructura válida. `WORKSPACE_STEP_CATALOG` es la matriz única entre setup y los comandos granulares de reparación.

El launcher web (`axiom app`) expone `/api/launcher/workspace/setup` y `/api/launcher/workspace/adopt`: preview no escribe y confirmación delega en los mismos runners/envelopes que la CLI. El resultado separa recursos estructurales, steps derivados, warnings, provenance y registry real; no existe una TUI pública equivalente.

## Fuera de la baseline inicial (no-goals explícitos del MVP)

Como contexto histórico, los overlays `standard`/`enterprise` no formaron parte del camino inicial de readiness; hoy están retirados. También quedan fuera de la baseline inicial `visual-studio-2026` como target de primer arranque, exigir `engram`/`cmm` como requisito de entrada, bridges externos/plugins/lanes paralelos avanzados e instalación user-level del binario como paso obligatorio. `cmm` es el reemplazo vigente de `codegraph`/`graphify` (ver `../integrations/01-capabilities-providers-y-toolchain.md`).


### Required-fields y confirmación launcher R-13 (2026-09-08)

Las superficies de install/join/setup/adopt, catálogo, plugins, roles/Git y ADO rechazan cuerpos incompletos antes de crear grants, escribir filesystem, invocar bridge o cambiar metadata. Las pruebas ACC-076 ejercitan esas superficies sobre el servidor real y comparan snapshots de no mutación; una preview nunca autoriza por sí sola el side effect.