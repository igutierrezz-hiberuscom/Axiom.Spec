# Requisitos

## RQ-001. Paridad demostrada

La retirada exige una matriz de todas las capacidades TUI y evidencia de su
sustituto en launcher o CLI.

## RQ-002. Ninguna capacidad exclusiva

No se elimina la TUI si una capacidad operativa sigue siendo exclusiva y no
hay una decision explicita de dejarla fuera.

## RQ-003. Limpieza de wiring

Registro, package, referencias, aliases, dependencias, tests y documentacion
activa se limpian de forma coherente.

## RQ-004. Preservar negocio

La retirada afecta superficies de interfaz, no commands/run functions que
usen otros consumidores.

## RQ-005. Regresion

La CLI construida, launcher y validaciones operativas deben continuar
funcionando tras el borrado.
