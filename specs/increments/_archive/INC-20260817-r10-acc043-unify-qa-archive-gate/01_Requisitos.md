# 01 Requisitos

## Objetivo del documento

Definir el contrato observable que autoriza, advierte o bloquea un archive según la política y evidencia QA, para que todas las superficies produzcan la misma decisión.

## Requisitos del incremento

1. Debe existir una sola evaluación canónica de QA para archive de `increment` y `bug`.
2. El resultado debe incluir de forma tipada el modo aplicable, el estado de evidencia (`passed`, `failed`, `cancelled` o `pending`), la disposición (`allow`, `warn` o `block`) y un mensaje apto para CLI/launcher/MCP.
3. El runner debe usar el resultado antes de persistir, mover o emitir receipt. El preview debe devolver exactamente la misma decisión sin escritura.
4. Con `qaLane: parallel`, archive procede para los cuatro estados y comunica el estado observado; `pending` no se puede presentar como éxito QA.
5. Con QA inline o requerida por rol, archive solo procede con `passed`; `pending`, `failed` y `cancelled` retornan un bloqueo claro y no cambian workflow-state, metadata ni ubicación del artefacto.
6. Increment, bug, integrate, launcher y MCP deben atravesar el mismo punto de decisión. Ningún caller puede transformar un bloqueo en warning.
7. El modo por defecto debe respetar la configuración existente. Si la policy requerida no puede leerse o la evidencia requerida no puede evaluarse, el resultado debe bloquear en lugar de permitir archive silenciosamente.

## Reglas de negocio relevantes

- `passed` es la única evidencia correcta para una policy QA requerida.
- `failed` y `cancelled` son estados terminales distintos de `passed` y deben ser visibles al operador.
- `pending` incluye QA sin iniciar, con state ausente o en un estado no terminal/no satisfactorio.
- El contrato describe QA; no ejecuta QA ni modifica la máquina de estados.

## Fuera de alcance funcional

No se rediseña `workflows.yaml`, los roles ni la topología. No se modifica la policy de confirmación declarada, ni se habilitan efectos o integraciones externas.