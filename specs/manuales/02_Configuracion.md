# 02. Configuración

Dónde vive la configuración de Axiom y cómo cambiarla, capa por capa.

## Las cuatro capas

La configuración de Axiom no es un único archivo, se reparte en:

- **`axiom.yaml`** (raíz del repo): manifiesto autoral del proyecto — identidad (`projectId`/`name`/`repoId`/`role`), `paths` recíprocos hacia repos hermanos, y el profile triple. Es la fuente única de verdad; `topology.yaml` se deriva de él.
- **`axiom.config/*.yaml`** (versionado): profiles/overlays/capabilities/providers/políticas — p. ej. `profiles.yaml`, `providers.yaml`, `topology.yaml`.
- **`~/.axiom/projects.yml`** (registro de usuario, machine-level): el registro v2 de proyectos conocidos por esta máquina, un `repos: { <role>: { role, path } }` por proyecto. Sustituyó al legado `~/.axiom/registry.json` (migración automática, ver `08_Glosario.md`).
- **`.axiom-state/<project>/`** (estado runtime, no versionado): `init.json`, `install-profile.json`, checkpoints, `workspace.json`, `tracker.json` (ver [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md)), y el overlay local en `.axiom-state/local/` (bindings, provider overrides).

## Cómo cambiar la configuración

- **`axiom configure`**: relee el profile triple desde `init.json`, recompone el install profile completo y persiste `install-profile.json`, materializando las surfaces derivadas del target activo (p. ej. `.github/copilot-instructions.md` para Copilot). Es single-shot: reaplica TODO el perfil persistido, no admite cambios incrementales de un solo campo. Repetilo cuando cambies `profiles.yaml`, el target activo, o templates/manifests usados para surfaces derivadas.
- **Operaciones incrementales ADD** (`axiom repo add`, `axiom adapter add <target>`, `axiom provider add <id>`, `axiom role add <roleId> --path <path>`): añaden UN elemento nuevo a un workspace ya inicializado, de forma idempotente y no-clobber, sin re-correr todo el setup. Los REMOVE equivalentes aún no existen (diferido).
- **`axiom workspace <granular>`** (`spec-base`, `adapters`, `skills`, `rules`, `mcp-config`, `config-scaffold`): re-aplica/repara UNA PARTE de un install ya existente (nunca añade algo nuevo al registro).

## Roles (team/code roles)

`axiom roles list` lista tanto los perfiles funcionales canónicos (`profiles.yaml`) como los team/code roles registrados en `topology.yaml#roles` — son dos ejes distintos. Para agregar y asignar un rol de equipo/código a un repo:

```bash
axiom roles register --id <roleId> --path <path>
axiom roles assign --repo <repoId> --role <roleId>
```

Los nombres de rol son texto libre (no un enum fijo) — cualquier nombre sanitizado (lowercase, guiones) es válido, no solo `backend`/`frontend`/`qa-e2e`. Lo mismo se puede hacer desde el launcher, pestaña **Instalar/Unirse** → tarjeta "Roles" (ver [11_Launcher_Visual.md](11_Launcher_Visual.md)), que internamente llama a los mismos `runRolesRegister`/`runRolesAssign`.

## Profiles, overlays, targets

- **`functionalProfile`**: `builder` (habilita `code.*` + roles de implementación) o `product-owner` (limitado a spec/SDD, `code.*` bloqueado).
- **`operationalOverlay`**: `local-only` (todo local, sin gateway, recomendado para desarrollo), `standard` (gateway opcional), `enterprise` (gateway requerido + gobierno/compliance ampliado).
- **`adapterTarget`**: el IDE/CLI de destino primario (uno de los 9 targets soportados). Un proyecto puede tener MÚLTIPLES adapters instalados a la vez (selección multi-select en el wizard de setup, persistida en `workspace.json#adapters`); `adapterTarget`/`spec.target` es el primario derivado (`adapters[0]`).

## Tuning de agente por adapter (verbosity/personality)

Cada adapter en la tabla de ruteo (`AXIOM_ADAPTER_ROUTING`, `@axiom/launcher`) puede llevar un `agentTuning` opcional: `{ model?, verbosity?: 'low'|'medium'|'high', personality?: 'pragmatic'|'balanced'|'thorough' }`. Cuando está presente, el prompt pregenerado incluye un bloque "Ajustes del agente" que le indica al agente ser conciso, pragmático y económico en tokens (`verbosity: low`, `personality: pragmatic` es el default en los tres adapters shippeados: `claude-code`, `github-copilot`, `cli`). Es puramente una instrucción de estilo del prompt — no afecta ruteo de modelo ni selección de provider. Se ve reflejado como una etiqueta junto a cada adapter en el selector del launcher (ver [11_Launcher_Visual.md](11_Launcher_Visual.md)).

## Relacionado

- [01_Que_Es_Cada_Cosa.md](01_Que_Es_Cada_Cosa.md)
- [03_Actualizar_Versiones.md](03_Actualizar_Versiones.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
- [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md)
- [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md)
