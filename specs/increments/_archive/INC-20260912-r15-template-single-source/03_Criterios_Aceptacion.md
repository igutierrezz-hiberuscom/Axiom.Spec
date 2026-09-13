# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-090-01: Rescate documentado
Existe registro, archivo por archivo, de la comparación entre las dos copias de plantillas, con la elección tomada y su motivo para las siete divergentes.

### AC-090-02: Fuente única sin vocabulario retirado
`Axiom/axiom.spec/templates/` no contiene menciones a perfiles o capas de política seleccionables, `axiom-gateway`, `generated-snapshots`, `product-owner` ni `enterprise`, salvo dentro de notas marcadas explícitamente como históricas.

### AC-090-03: Copia canónica retirada
`Axiom.Spec/templates/` no existe y `Axiom.Spec/README.md` no declara plantillas entre el contenido del repositorio.

### AC-090-04: Procedencia declarada correctamente
Ningún comentario de código ni documento activo presenta `Axiom.Spec/templates/*` como fuente canónica de plantillas.

### AC-090-05: Test dorado eficaz
El test que protege las copias bundleadas compara la fuente única completa. Se demuestra su eficacia introduciendo una divergencia temporal y comprobando que falla, en al menos una plantilla que antes no estaba cubierta.

### AC-090-06: Generación sin regresión
Las pruebas de generación de `opencode`, `claude-code`, `codex` y `antigravity` pasan con override on-disk presente y ausente, con salida equivalente a la anterior al cambio.

### AC-090-07: Sin retiradas por sospecha
Las seis plantillas sin consumidor conocido siguen presentes en la fuente única.

### AC-GEN-01: Integridad general
`npm run build`, las suites de adapters y de workflow, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` pasan limpiamente.
