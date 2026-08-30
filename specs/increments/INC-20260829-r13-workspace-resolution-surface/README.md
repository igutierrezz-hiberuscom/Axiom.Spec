# Resolución local y superficie incremental del workspace R-13

> **Código**: INC-20260829-r13-workspace-resolution-surface
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-063, ACC-064, ACC-069
> **Dependencias**: A, B y C

## Objetivo

Hacer que la topología schema 2 del axiomRepo resuelva el workspace sin depender de `projects.yml`, retirar superficies repo ambiguas y dar a `workspace.json` un contrato único, estricto y seguro.

## Revalidación

`resolveExistingProject` exige el catálogo y reconstruye kinds mediante keys `sdd/spec`; `repo attach` crea una entrada de proyecto en vez de adjuntar; `repo add` acepta control/spec; `readWorkspaceJson` convierte cualquier error en arrays vacíos y los writers no coordinan concurrencia.

## Alcance

- ACC-063: resolución local desde axiomRepo o code repo por identidad/puntero; catálogo user-level opcional y reconciliable; `--no-register` plenamente operativo.
- ACC-064: retirar `repo attach` y `repo add --kind control|spec`; `repo/role add` incorpora solo code repos al proyecto local.
- ACC-069: schema tipado único de workspace state, loader fail-closed y updates atómicos/concurrentes/no-op estables.
- Proyección del catálogo con metadata estructural explícita, launcher/MCP/member install y documentación operativa afectada.

## No objetivos

- ACC-068 se separa a `INC-20260829-r13-workspace-step-reconciliation` después de E porque depende del preflight/coordinador estructural.
- Preflight total, journal y reconciliación seccional de axiom.yaml son E.
- No cambiar seguridad launcher ni R-13.5.

## Decisiones cerradas

1. Autoridad local: topology schema 2 en axiomRepo. `projects.yml` solo descubre/recuerda recencia y jamás reconstruye el grafo.
2. Cada `WorkspaceSetupSpec` contiene exactamente un `kind: axiom` y 0..N `kind: code`; legacy sources son referencias read-only de topology, no targets escribibles de setup incremental.
3. `axiom.yaml` de code repo aporta projectId/repoId/kind/puntero. El resolver canonicaliza el puntero, valida identidad contra topology y funciona sin home/catálogo.
4. Si catálogo existe y diverge, la operación local usa topology, devuelve warning tipado y solo reconcilia catálogo mediante acción explícita/operación estructural solicitada.
5. La proyección de registro usa `repoId`, `kind` y role/ownership explícitos; nunca keys convencionales.
6. `repo attach` desaparece. `projects add|join` son las únicas altas user-level. `repo add` y `role add` solo aceptan code repo + role declarable; segundo axiomRepo o legacy fuera de adopción falla.
7. `workspace.json` conserva `schemaVersion: 1` como único contrato (no existe legacy decidido), con shape cerrada: project identity, adapters, providers, `createdAt`, `updatedAt` y extensiones declaradas si existen.
8. Loader devuelve `Result`; ausente produce estado inicial válido, corrupto/futuro/tipos/unknown fields produce error visible y no overwrite.
9. Arrays se deduplican/ordenan; `createdAt` es inmutable, `updatedAt` cambia solo si cambian datos; no-op conserva bytes. Updates usan el lock/atomic writer común.

## Riesgos

Bootstrap circular axiomRepo/code repo, punteros relativos Windows, catálogo stale, consumidores de attach y cambios locales R-12 en workspace setup/MCP. Se cubren con fixtures sin registro, divergencia y barrido de CLI.

## Compatibilidad

No se preservan attach, kinds control/spec ni inferencias `sdd/spec`. Workspace JSON inválido no se migra ni resetea; debe repararse explícitamente.

## Validación prevista

Fixtures desde axiom/code repos con home ausente/stale; adopt→repo add; help/registro Commander; workspace JSON corrupto/futuro/concurrente/no-op; launcher/MCP; build y diff-check.

## Integración estable

Diferida al cierre del lote; no editar `specs/00..08` ni `context/**` durante apply.
