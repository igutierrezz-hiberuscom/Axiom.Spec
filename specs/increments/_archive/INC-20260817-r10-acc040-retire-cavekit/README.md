# R-10 ACC-040 retire Cavekit runtime surface

> **Código**: INC-20260817-r10-acc040-retire-cavekit
> **Estado**: Archivado mediante Core
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: eliminar

## Resumen

Se retiró `@axiom/cavekit-discipline` del runtime y de las referencias activas porque no tenía consumidores productivos.

## Contexto y motivación

Cavekit era un paquete privado de funciones puras sin importadores en CLI, launcher, workflow, doctor ni configuración. Mantenerlo como capacidad operativa o dependencia directa añadía un contrato no ejecutado.

## Alcance realizado

- Se retiraron package, project references, pruebas propias, dependencia directa no usada, lockfile generado y referencias activas.
- Se conservó `zod` cuando continuaba siendo consumido fuera de Cavekit.
- Se preservó el antecedente histórico 0015 sin reescribirlo.
- La trazabilidad posterior usa exclusivamente la capacidad Core disponible: la Decision `DEC-20260818-134600-3jfjak` está enlazada al correctivo `INC-20260818-r10-closure-correction`.

## No alcance

No se elimina el antecedente 0015, no se introduce un sustituto y no se inventa una relación de supersesión de Decisions.

## Resultado verificable

El package y sus referencias runtime fueron retirados sin restaurar consumidores ni eliminar `zod` ajeno. El hecho operativo es verificable en el runtime y sus validaciones; 0015 queda como antecedente histórico.

## Límite Core y cierre documental

La Decision relacionada continúa `proposed`. El Core disponible permite enlazarla a un incremento, pero no ofrece aceptar, cerrar ni superseder una Decision. Por tanto ACC-040 no reclama «superación formal» de 0015: registra una retirada de runtime y la relación Core realmente existente.

## Cierre Core

El Core archivó este incremento en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-09-19.085Z-increment-archive-success.json` lo confirma. Las cifras de validación previas son fotografía histórica; el cierre correctivo R-10 conserva la evidencia vigente separada.
