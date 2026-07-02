# Instalación y onboarding

Fuente: `Axiom/docs/installation.md`, `Axiom/docs/first-project-readiness.md`, `Axiom/docs/configuration/files/onboarding.md`.

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

Script ejecutable: `npm run readiness:first-project` (`Axiom/scripts/verify-first-project-readiness.mjs`). Siembra un proyecto temporal con la baseline canónica copiada desde `axiom.spec/config/`, `axiom.spec/templates/`, `axiom.spec/target-axiom-skills/`, `axiom.spec/target-axiom-agents/`, más `_builder/`, `AGENTS.md` y `axiom.skills.lock` del propio repo `Axiom/`, y ejecuta:

```
init → configure → toolchain repair → sync → start → gateway start → gateway status → audit → doctor
```

Falla si algún paso devuelve exit code ≠ 0, falta un artefacto esperado, o `gateway status` no reporta `state: active`.

**Nota crítica de estado real**: este script asume que `axiom.spec/config/`, `axiom.spec/templates/`, `axiom.spec/target-axiom-skills/`, `axiom.spec/target-axiom-agents/`, `AGENTS.md` y `axiom.skills.lock` existen en la raíz de `Axiom/`. Verificado: ninguno existe en este checkout → el script fallaría hoy con `ENOENT` en `seedCanonicalBaseline()`. Detalle en [../references/03-riesgos-y-brechas-conocidas.md](../references/03-riesgos-y-brechas-conocidas.md).

## Baseline recomendada y checklist manual

Triple recomendado: `functionalProfile: builder`, `operationalOverlay: local-only`, `adapterTarget: opencode` (motivo: perfil más cubierto en runtime, menor fricción, target más estable).

Checklist manual de equipo:
1. `npm run build` verde en `Axiom/`.
2. `npm test` verde en `Axiom/`.
3. `node ../axiom.spec/scripts/doctor-validate-contracts.mjs` verde desde la raíz del repo (mismo riesgo de ausencia que arriba).
4. `npm run readiness:first-project` en `PASS`.
5. Documentación operativa navegable (instalación, uso diario, CLI, troubleshooting, esta guía).

## Fuera de la baseline inicial (no-goals explícitos del MVP)

Overlays `standard`/`enterprise` como camino inicial obligatorio; `visual-studio-2026` como baseline de primer arranque; providers post-MVP (`engram`, `codegraph`, `graphify`) como requisito de entrada; bridges externos/plugins/lanes paralelos avanzados; instalación user-level del binario como paso obligatorio.
