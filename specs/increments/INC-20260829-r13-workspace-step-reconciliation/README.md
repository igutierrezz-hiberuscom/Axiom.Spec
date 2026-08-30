# Reconciliación de pasos granulares del workspace R-13

> **Código**: INC-20260829-r13-workspace-step-reconciliation
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acción**: ACC-068
> **Dependencias**: D y E

## Objetivo

Unificar setup y comandos granulares mediante un catálogo común de pasos/ownership, con validación total previa, reparación completa e idempotente y resultados tipados.

## Motivo de separación

ACC-068 depende expresamente de ACC-063/065/066. Se separa de D para evitar introducir un preflight o coordinador alternativo antes de E.

## Revalidación

`workspace adapters --path` acepta paths ajenos; `mcp-config` puede escribir antes de rechazar target; adapters omite process surfaces, rules omite AGENTS y config-scaffold cubre menos que setup.

## Alcance

- Catálogo único de pasos con owner, targets, outputs, preflight y executor.
- Matriz explícita setup↔comando granular.
- Adapters repara adapter outputs y process surfaces propias, sin invadir MCP.
- Rules reconcilia la sección Axiom de AGENTS.
- Config-scaffold cubre todas las declaraciones/políticas/catálogos reparables de setup.
- MCP valida target/path antes de escribir el MCP canónico.
- Distinción add/enable frente a repair/regenerate y resultados por paso.

## No objetivos

No ampliar setup con arquitectura nueva, no cambiar transacción E, no generar Workbench/MCP obligatorio/índices, no R-13.5.

## Decisiones

1. `WORKSPACE_STEP_CATALOG` es la única fuente declarativa; setup y granulares seleccionan entradas del catálogo.
2. Cada step declara `id`, owner, writable targets, read-only inputs, prerequisites, preflight y output classes.
3. Todos los targets del comando se prevalidan en conjunto con E antes del primer write.
4. `add/enable` puede cambiar workspace state para incorporar capacidad; `repair/regenerate` solo materializa lo ya declarado y no habilita nada.
5. Resultado por step usa `created|updated|unchanged|skipped|failed`; fallo estructural no se degrada a warning.
6. AGENTS se actualiza solo dentro de markers Axiom y preserva TEAM:CUSTOM/human content.
7. No se duplica lógica de generators; el catálogo apunta a executors existentes refactorizados.

## Riesgos

Ownership solapado entre adapters/process surfaces/MCP y cambios locales R-12 en comandos workspace. Se exige matriz exhaustiva y preservación de diffs previos.

## Compatibilidad

No hay fallback a los comandos parciales anteriores; instalaciones dañadas se reparan según catálogo actual.

## Validación

Fixtures parcialmente dañadas, target/path ajeno/desconocido con cero mutación, matriz completa, idempotencia desde axiom/code repos, build y diff-check.

## Integración estable

Diferida a la pasada final; sin cambios en specs 00..08/context durante apply.
