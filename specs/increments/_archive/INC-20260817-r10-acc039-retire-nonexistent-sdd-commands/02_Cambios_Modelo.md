# 02 Cambios de Modelo

## Objetivo del documento

Alinear las superficies SDD materializables con los entrypoints públicos que la CLI registra realmente.

## Entidades o estructuras afectadas

- Catálogos de agentes y skills, process surfaces y documentación/plantillas emitidas.
- Entrypoints públicos `axiom state`, `axiom-increment`, `axiom-bug`, `axiom-plan` y `axiom-role`.

## Contratos o estados afectados

Se retiran como contrato activo `axiom sdd advance`, `axiom plan` y `axiom role start`; no cambia ninguna transición del workflow. Los 19 intent commands internos de `@axiom/orchestrator` siguen siendo stubs `not-implemented` y no son entrypoints públicos.

## Notas de compatibilidad

Las instrucciones nuevas usan los comandos registrados. Las referencias históricas archivadas se conservan como historia y no constituyen una promesa operativa.
