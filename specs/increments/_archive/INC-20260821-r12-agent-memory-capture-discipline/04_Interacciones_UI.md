# 04 Interacciones UI

## Objetivo del documento

Definir la interacción explícita de los agentes con la memoria y las nuevas opciones visibles de CLI.

## Superficie UI afectada

- `axiom memory add --help` expone `--visibility`, `--rationale` y `--source`.
- Las skills/agents Kiro incluyen las plantillas de captura de decisión y bug.

## Flujo de interacción

Al confirmar una decisión o bug, el agente escoge una visibilidad y ejecuta una de estas formas:

```text
axiom memory add --kind decision --text "<hecho confirmado>" --visibility project-shared --rationale "<por qué>" --source "<artefacto o evidencia>"
axiom memory add --kind bug --text "<síntoma, causa y resolución confirmada>" --visibility private --rationale "<por qué conservarlo>" --source "<artefacto o evidencia>"
```

Elige `project-shared` solo para conocimiento confirmado y reutilizable por el equipo. Elige `private` para contexto local y temporal que no se debe exportar; nunca guarda secretos en ninguno.

## Estados visibles

La ayuda muestra la semántica de las options. Un valor inválido o un error del backend se comunica al agente; no existe éxito implícito.

## Cascadas y comportamiento reactivo

La captura no ejecuta Knowledge Sync, Git, lifecycle ni telemetría. La exportación sigue siendo una operación posterior y explícita.
