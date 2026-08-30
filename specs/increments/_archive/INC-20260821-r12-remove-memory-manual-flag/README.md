# r12-remove-memory-manual-flag

> **Código**: INC-20260821-r12-remove-memory-manual-flag
> **Estado**: Archivado
> **Fecha de creación**: 2026-08-21
> **Tipo de cambio**: simplificación de interfaz CLI

## Resumen

Retirar la opción redundante `--manual` de `axiom memory add`. El propio subcomando ya es la acción explícita de guardar memoria y hoy el flag no altera el comportamiento.

## Contexto y motivación

La ayuda y los tipos actuales sugieren una regla de curación que requiere `--manual` para algunos kinds, pero `runMemoryAdd` persiste siempre y descarta el valor recibido. Mantener el flag introduce una decisión inexistente y complejidad innecesaria.

## Alcance

### Incluido

- Eliminar `--manual` de la CLI, su ayuda y el argumento interno de `runMemoryAdd`.
- Retirar las descripciones de auto-persistencia/manual que contradicen el comportamiento real.
- Actualizar pruebas dirigidas para invocar el guardado explícito sin el campo eliminado.
- Verificar que `memory add` continúa guardando cualquier kind válido y que Commander rechaza `--manual`.

### Excluido

- No cambiar los `MemoryKind`, el esquema de `MemoryEntry` ni los comandos `show`, `query` o `inventory`.
- No rediseñar la política de captura de decisiones y bugs; ese contrato corresponde al incremento de disciplina de skills.
- No migrar backends de memoria; Engram obligatorio corresponde a su incremento independiente.

## Documentos del incremento

- `01_Requisitos.md`: comportamiento y límites.
- `02_Cambios_Modelo.md`: contrato de CLI y API interna retirado.
- `03_Criterios_Aceptacion.md`: comprobaciones ejecutables.
- `04_Interacciones_UI.md`: impacto visible en help y errores.

## Dudas abiertas

Ninguna. `memory add` es siempre un guardado manual y explícito.

## Decisiones funcionales cerradas

- La eliminación es de interfaz y argumento interno; no se añade un sustituto.
- `memory add --text ... --kind <kind>` sigue guardando por acción explícita del operador o agente.
- Un uso de `--manual` debe fallar como opción desconocida, no ignorarse silenciosamente.

## Consolidación en la spec general

Al cerrar, actualizar solo claims activos de `specs/00..08` o `context/**` que describan el flag o una curación que lo requiera. No crear contexto técnico nuevo si no hay conocimiento estable adicional.

## Estrategia E2E

Comprobar la ayuda compilada, un guardado válido sin `--manual` y el rechazo de la opción retirada.

## Trazabilidad y fuentes

- `Axiom/apps/cli/src/commands/memory.ts`
- `Axiom/apps/cli/tests/memory.test.ts`
- `Axiom/packages/memory/src/curation.ts`

## Estado de validación humana

Implementación, validación dirigida y revisión independiente completadas. `npx vitest run apps/cli/tests/memory.test.ts --reporter=dot` pasó 1 archivo y 7 pruebas; `npm run build` pasó; la ayuda compilada no expone `--manual` y el binario lo rechaza como opción desconocida. La revisión independiente identificó un comentario de curación residual, se corrigió sin cambiar su API y la re-revisión aprobó el resultado sin blockers.

## Integración documental

No existían claims activos sobre `--manual` en `specs/00..08` ni en `context/**`; la búsqueda dirigida confirmó que no hace falta una actualización canónica ni crear contexto técnico nuevo.
