# Requisitos — INC-20260811-acc-031-r08-docs

## REQ-1: Conteo real del catálogo de agents

`Axiom/packages/agents/README.md` debe reflejar el conteo real del catálogo
(`axiom.config/agents-catalog.yaml`): **14 agents**, no "3 entries oficiales".

## REQ-2: Vía real de materialización de agents

El README de `@axiom/agents` debe describir correctamente que el materializado
real de agents en workspace lo hace `workspace-process-surfaces.ts`, y que
`materializeAgentSet` del paquete no es la vía principal invocada por comandos.

## REQ-3: Comentario de DEFAULT_DESIRED_SKILLS

El comentario de `DEFAULT_DESIRED_SKILLS` en `Axiom/packages/skills/src/apply.ts`
no debe afirmar una sincronización inexistente con `toolchain.yaml` (subset
`mvp: true`). Debe describir el set como lo que es: skills deseadas por defecto
en el scaffolding de workspace.

## REQ-4: Sin cambios de comportamiento

La corrección es exclusivamente documental. No se altera lógica, catálogos,
parámetro `mvp` ni materializadores.