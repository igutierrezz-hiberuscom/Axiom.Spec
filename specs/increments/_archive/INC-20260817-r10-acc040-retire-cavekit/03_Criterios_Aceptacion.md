# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] `@axiom/cavekit-discipline` deja de existir como package y project reference runtime.
- [x] Sus tests y dependencia directa no usada se retiran sin eliminar `zod` necesario por otros consumidores.
- [x] No quedan referencias activas que lo presenten como operativo.
- [x] ADR-0015 y artifacts archivados se preservan; la superación queda registrada por Core.
- [x] Build y pruebas focalizadas pasan cuando el estado base permita cargarlas.

### Happy path

El runtime se construye sin Cavekit y la documentación activa solo lo conserva como antecedente histórico.

### Validaciones y errores

El package graph confirma que `zod` continúa disponible para consumidores no Cavekit.

### Permisos y visibilidad

Sin cambios.

### Estados y efectos observables

La retirada no altera ningún comando porque Cavekit no tenía consumidor runtime.
