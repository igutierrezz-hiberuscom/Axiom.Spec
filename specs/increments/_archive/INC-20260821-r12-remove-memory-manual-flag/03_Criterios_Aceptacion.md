# 03 Criterios de Aceptación

## Criterios de aceptación

1. La ayuda de `axiom memory add` no contiene `--manual` ni describe una regla que lo requiera.
2. Los tipos y llamadas a `runMemoryAdd` no incluyen el campo `manual`.
3. Una llamada válida a `memory add` sin `--manual` persiste una entry y conserva los metadatos admitidos.
4. El binario compilado rechaza `--manual` como opción desconocida.
5. Las pruebas dirigidas de memoria y el build pasan.

### Happy path

Un agente ejecuta `axiom memory add --text "Decisión confirmada" --kind decision` y la entry se guarda sin otro flag.

### Validaciones y errores

`--manual` ya no es una opción compatible; Commander comunica el error y el proceso falla.

### Permisos y visibilidad

No cambia la visibilidad, el aislamiento project-scoped ni la política de exportación de memoria.

### Estados y efectos observables

La única diferencia observable es la desaparición del flag redundante y de la explicación falsa sobre su necesidad.
