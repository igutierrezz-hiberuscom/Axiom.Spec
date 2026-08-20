# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] El manual activo de `axiom upgrade` refleja la posible inicialización de `managed-state.json` en el primer uso.
- [x] El comentario de lifecycle de gates ya no afirma que los gates carecen de I/O cuando comprueban ficheros.
- [x] No se modifica código funcional de upgrade ni de los gates.
- [x] Las pruebas focalizadas relevantes pasan (2 archivos, 19 tests).

### Happy path

Un operador puede leer el manual de upgrade y anticipar el archivo gestionado que puede aparecer en una primera ejecución.

### Validaciones y errores

La validación confirma que el comportamiento del runtime se conserva y que los textos coinciden con él.

### Permisos y visibilidad

Sin cambios.

### Estados y efectos observables

Sin nuevos estados ni efectos; solo se alinean descripciones existentes con efectos ya vigentes.
