# 05 Interfaces Operativas

## Surfaces principales

1. CLI del runtime (`axiom`, entry point `apps/cli/dist/index.js`, instalado vía `scripts/install-global.mjs` como shim/`npm link` único en el PATH del usuario);
2. TUI operativa (`axiom tui`, `@axiom/tui`);
3. adapters que conectan Axiom con harnesses o IDEs (`@axiom/adapters-*`);
4. manifests/YAML que describen la configuración y topología del proyecto (`axiom.yaml`, `axiom.spec/config/*.yaml`, `topology.yaml`).

## CLI: comandos documentados en profundidad

`init`, `join`, `configure`, `sync`, `start`, `audit`, `doctor`, `upgrade`, `tui`, `model` (`show`/`set`/`unset`/`reset`/`validate`), `components` (`list`/`show`/`install`/`uninstall`/`restore`), `skills` (`list`/`refresh`/`drift`). Fuente: `Axiom/docs/cli/*.md`.

## CLI: comandos presentes en código sin documentación operativa equivalente

`apps/cli/src/commands/` tiene 36 ficheros (~16.400 líneas). Además de los documentados arriba existen: `app`, `app-api`, `app-plugins`, `app-plugins-azure-devops`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`. Estos se mencionan de forma dispersa en los documentos de incremento (`0019`–`0030`) pero no tienen una página propia bajo `docs/cli/`. Tratar su comportamiento como verificable en código, no como contrato documentado establecido, hasta que se audite comando por comando.

## TUI (`axiom tui`)

Menú de 6 items (configurar, sincronizar, diagnóstico, upgrade, model routing, salir). Para mutaciones (`configure`, `sync`, `upgrade`) muestra preview read-only, pide confirmación y muestra un post-run summary con restore point y follow-ups. Exige TTY. Fuente: `Axiom/docs/cli/tui.md`, package `@axiom/tui`.

## Adapters (surfaces por IDE/CLI externo)

6 adapters con package dedicado y contrato común `generate<Target>Config(args) → Promise<Result<GeneratorResult, AdapterGeneratorError>>`: `opencode` (`multi-mode`), `claude-code` (`single-mode`), `github-copilot`, `vscode`, `cursor`, `litellm` (los 4 últimos `fallback-only`). 3 targets adicionales declarados sin adapter dedicado (`copilot-vscode`, `antigravity`, `visual-studio-2026`) caen al default del IDE. Fuente: `Axiom/packages/adapters/README.md`.

## Documentación operativa navegable

`Axiom/docs/README.md` es el índice hacia manuales de instalación, configuración, uso diario, CLI, ficheros generados y troubleshooting — ya existe y está mantenido, no es un artefacto a construir desde cero.

## Regla

Las interfaces operativas se implementan en `Axiom/`, pero su comportamiento esperado se define primero en `Axiom.Spec/`. Regla vigente pero con brecha real: parte de la superficie de comandos (sección anterior) ya existe en código sin haber pasado primero por una spec formal en `Axiom.Spec/`.