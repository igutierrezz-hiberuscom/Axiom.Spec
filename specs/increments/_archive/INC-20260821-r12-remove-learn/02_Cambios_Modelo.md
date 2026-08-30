# 02 Cambios de Modelo

## Objetivo del documento

Identificar los contratos que desaparecen sin alterar el modelo general de memoria ni telemetría.

## Entidades o estructuras afectadas

Se retiran los contratos específicos `AuditEventSample`, `LessonCandidate`, `LessonRecord`, `LessonSource`, `PersistLessonsScope` y `RecallLessonsOptions`, junto con la etiqueta reservada `lesson` y las funciones de extracción, persistencia y recall de lecciones.

## Contratos o estados afectados

- Se retira el subcomando `axiom learn`.
- Se retira el bloque `recentLessons` de la salida de contexto.
- El contrato de `MemoryEntry` genérico permanece; una entrada histórica con tags arbitrarios no se elimina por esta acción.
- El formato de `audit.log` y el contrato de `axiom audit` permanecen sin cambios.

## Notas de compatibilidad

Una invocación futura de `axiom learn` debe fallar como comando desconocido. No se implementa un alias, migración silenciosa ni compatibilidad que mantenga la superficie retirada.
