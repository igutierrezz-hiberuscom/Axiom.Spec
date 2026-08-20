# R-10 ACC-044 active divergence investigation

> **Código**: INC-20260817-r10-acc044-active-divergence-investigation
> **Estado**: Archivado mediante Core
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: investigar

## Resumen

Esta investigación clasificó las divergencias R-10 contra código ejecutable, configuración, catálogos y documentación. No modificó producto. Su informe fue rectificado por el correctivo R-10 posterior para distinguir la fotografía original del contrato final.

## Clasificación histórica y destino

| Tema | Hecho investigado | Destino R-10 |
| --- | --- | --- |
| Sintaxis SDD y `sdd advance` | La CLI real usa entrypoints con guion y `axiom state`; los intents internos no son CLI. | ACC-039 retiró promesas públicas inexistentes. |
| Approval, archive e integrate | Las rutas divergían y `requiresApproval` no se aplicaba uniformemente. | ACC-041, ACC-042 y ACC-043 implementaron el runner/gates comunes. |
| YAML/default de workflows | Existían representaciones y fallbacks divergentes. | ACC-045 estableció YAML canónico, default empaquetado y error fail-closed. |
| Cavekit | El package no tenía consumidor productivo; 0015 era antecedente histórico. | ACC-040 retiró runtime; la Decision posterior sólo se enlaza por Core, sin supersesión formal. |
| Alias Copilot y LiteLLM | La investigación previa no detectó toda la compatibilidad pública activa. | El correctivo `INC-20260818-r10-closure-correction` define su retirada y verificación; este artefacto archivado no declara esa implementación ni su validación final. |

## Resultado del informe

El informe final caso a caso vive en `02_Cambios_Modelo.md`. Los artefactos ACC archivados y 0015 se preservan como historia; no sustituyen evidencia del HEAD ni del correctivo posterior.

## Cierre Core

El Core archivó esta investigación en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-10-00.081Z-increment-archive-success.json` lo confirma. Las cifras de validación que acompañaron al archivo son históricas y no declaran el cierre del correctivo R-10.
