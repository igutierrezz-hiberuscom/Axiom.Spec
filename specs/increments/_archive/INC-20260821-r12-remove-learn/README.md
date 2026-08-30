# r12-remove-learn

> **Código**: INC-20260821-r12-remove-learn
> **Estado**: Archivado
> **Fecha de creación**: 2026-08-21
> **Tipo de cambio**: eliminación de superficie opcional

## Resumen

Retirar `axiom learn` y el subsistema de lecciones derivadas del audit trail. La memoria general del proyecto permanece como capacidad independiente; el audit trail sigue siendo trazabilidad operativa.

## Contexto y motivación

El aprendizaje continuo añade comandos, heurísticas, almacenamiento y una presentación de contexto que no son necesarios para los primeros proyectos de Axiom. La retirada reduce la superficie operativa sin convertir la telemetría en una dependencia de memoria.

## Alcance

### Incluido

- Retirar `axiom learn capture` y `axiom learn list` del registro de la CLI y de la ayuda.
- Eliminar la lógica `learning.ts`, su contrato público y sus pruebas dedicadas.
- Retirar la captura de lecciones desde `audit.log` y el bloque `recentLessons` de `axiom context status`.
- Retirar referencias activas de código, pruebas y documentación del runtime que describan `learn` como capacidad vigente.
- Conservar `axiom audit`, el audit trail y la memoria genérica sin semántica de lección.

### Excluido

- No retirar ni modificar el audit trail, su integridad o su retención.
- No rediseñar la memoria general; ese trabajo pertenece al incremento de Engram obligatorio.
- No borrar referencias archivadas o históricas que documenten el origen de la funcionalidad retirada.

## Documentos del incremento

- `01_Requisitos.md`: comportamiento esperado y límites.
- `02_Cambios_Modelo.md`: contratos y superficies eliminadas.
- `03_Criterios_Aceptacion.md`: comprobaciones ejecutables.
- `04_Interacciones_UI.md`: impacto en CLI y contexto.

## Dudas abiertas

Ninguna. La decisión es retirar la funcionalidad, no sustituirla.

## Decisiones funcionales cerradas

- La repetición de eventos del audit trail no genera lecciones ni notas de memoria.
- La memoria continúa siendo una operación explícita.
- La ausencia de `learn` no cambia la emisión ni la auditoría de telemetría.

## Consolidación en la spec general

Al cerrar, reconciliar únicamente los claims activos de `specs/00..08` y `context/**` que presenten `learn` o `recentLessons` como capacidad vigente. La historia permanece en artefactos archivados.

## Estrategia E2E

Comprobar que la ayuda compilada no expone `learn`, que `context status` no lee ni muestra lecciones y que las rutas de audit y memoria no regresan.

## Trazabilidad y fuentes

- `Axiom/apps/cli/src/commands/{learn,context}.ts`
- `Axiom/apps/cli/src/index.ts`
- `Axiom/packages/memory/src/{learning,index}.ts`
- pruebas focalizadas de learn, context, memoria y audit.

## Estado de validación humana

Implementación y revisión independiente completadas. `npm run build` y las siete suites dirigidas de contexto, audit, launcher, memoria y Engram pasan (63 pruebas); `axiom learn` se rechaza como subcomando desconocido. La revisión independiente confirmó la retirada completa, la preservación del audit trail y de la memoria general, y corrigió dos comentarios residuales sin efecto runtime.

## Integración documental

Se retiraron los claims activos de aprendizaje continuo de `specs/01`, `04`, `05`, `06` y `08`, y de `context/operations/02-doctor-troubleshooting-y-telemetria.md`. La historia queda únicamente en los incrementos archivados. No hubo cambios estables que requieran crear o indexar contexto técnico nuevo.
