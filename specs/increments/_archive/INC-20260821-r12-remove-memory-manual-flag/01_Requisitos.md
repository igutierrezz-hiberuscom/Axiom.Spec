# 01 Requisitos

## Objetivo del documento

Eliminar el flag `--manual` que no cambia la persistencia de `axiom memory add`.

## Requisitos del incremento

1. `axiom memory add` no acepta ni documenta `--manual`.
2. `MemoryAddArgs` y `runMemoryAdd` no reciben un argumento `manual`.
3. El comando guarda una entry válida de forma explícita sin depender del kind ni de un flag adicional.
4. La CLI rechaza un uso de `--manual` con el error estándar de opción desconocida.

## Reglas de negocio relevantes

Invocar `memory add` es la confirmación explícita de guardar memoria. No existe una segunda confirmación manual por kind.

## Fuera de alcance funcional

No se automatiza la captura de decisiones o bugs y no se cambia el backend de persistencia.
