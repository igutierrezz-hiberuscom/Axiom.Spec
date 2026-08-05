# Incremento 0030 — Operator App Plugins and External Bridge

> **Estado**: archived 2026-06-30
> **Plan**: `axiom.spec/plans/PLAN-INC-0030-operator-app-plugins-and-external-bridge.md`
> **Archive**: `openspec/changes/2026-07-01-0030-operator-app-plugins/`

## Resumen ejecutivo

Incremento 0030 completa el Lote B de `0025` con un sistema de
plugins project-scoped para la app portable y el bridge Azure DevOps
declarativo.

## Mini-lotes

### D1 — Plugin loader

- `loadAppPlugins(projectRoot, projectName)`: discovery en
  `.axiom-state/<project>/app-plugins/*.json`.
- Schema guard tolerante: plugins malformados → warnings, no
  abortan la app.
- ID único por proyecto; duplicados se rechazan con warning.

### D2 — Plugin `azure-devops` declarativo

- 2 tabs (`work-items`, `mapping`).
- 4 actions: 1 read (`list-work-items`), 3 external-mutation
  (`create-work-item`, `register-mapping`).
- 9 fields cubriendo work items, asignación, iteration path, tags.

### D3 — API integration

- `GET /api/projects/:id/plugins`: retorna los plugins cargados
  del filesystem + `azure-devops` si no está en disco.
- 11 GET endpoints en total (10 + plugins).

### D4 — Docs y openspec

- Roadmap, plan, openspec.

## Métricas

- **1000/1000 tests verde** (102/102 files). +8 tests del plugin.
- **`tsc -b` verde**.
- **GATE 0025 honrada**: la app NO introduce lógica de negocio
  paralela; el bridge declara el contrato, NO ejecuta.
- **GATE 0030 honrada**: plugins project-scoped; la mutación
  externa exige `confirmed: true`.

## Siguiente paso

Iniciar incremento 0028 (workflow UX and archive safety completion).
