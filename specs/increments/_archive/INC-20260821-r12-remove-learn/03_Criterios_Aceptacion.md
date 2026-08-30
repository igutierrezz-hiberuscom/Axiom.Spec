# 03 Criterios de Aceptación

## Criterios de aceptación

1. La ayuda de la CLI compilada no lista `learn`.
2. El registro de comandos no importa ni registra código de aprendizaje continuo.
3. No existen fuentes, exports o pruebas activas de `learning.ts`, `runLearnCapture`, `runLearnList`, `extractLessons`, `persistLessons` o `recallLessons`.
4. `axiom context status` no muestra `recentLessons` ni resuelve el backend de memoria para ese bloque.
5. Las pruebas dirigidas de contexto, audit y memoria restante pasan, al igual que el build.
6. El cambio no elimina el audit trail ni modifica sus pruebas de integridad.

### Happy path

Un operador usa `axiom context status` y recibe el contexto normal sin un bloque de lecciones. `axiom audit` continúa verificando el audit trail.

### Validaciones y errores

`axiom learn` se rechaza por Commander como subcomando inexistente. No se genera memoria a partir de `audit.log`.

### Permisos y visibilidad

No se añaden permisos, red ni mecanismos de compartición.

### Estados y efectos observables

La retirada elimina comandos y salida visible; no escribe ni borra estado de memoria existente.
