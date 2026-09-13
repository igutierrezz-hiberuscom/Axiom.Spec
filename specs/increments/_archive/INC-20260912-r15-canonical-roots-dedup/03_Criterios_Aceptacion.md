# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-094-01: Formato único
[x] Existe un solo formato y una sola raíz de decisiones en el repositorio canónico; la migración Core y su motivo están registrados en README.

### AC-094-02: Sin colisión de numeración
[x] No existen dos artefactos gestionados con el mismo ID; `0032` conserva ADR-0032-toolchain-versioning y el documento legacy como DEC-20260913-091609-fuxoda.

### AC-094-03: Referencias intactas
[x] Ninguna referencia activa del workspace apunta a un documento de decisión inexistente tras la unificación; el barrido de `specs/00..08` no encontró destinos rotos.

### AC-094-04: Raíces legacy retiradas sin pérdida
[x] `increments/` y `bugs/` top-level fueron retiradas tras comparar sus contenidos; el R-10 real permanece archivado con receipts.
`Axiom.Spec/increments/` y `Axiom.Spec/bugs/` no existen. Se demuestra, antes de retirarlas, que su contenido no aportaba información ausente en los artefactos vigentes o archivados, y que el artefacto real de `INC-20260817-r10-acc038-lifecycle-docs` sigue archivado con sus receipts.

### AC-094-05: Carpeta vacía eliminada
[x] `Axiom.Spec/axiom.spec/` no existe tras comprobar que no contenía archivos.
`Axiom.Spec/axiom.spec/` no existe al cierre.

### AC-094-06: Raíces activas operativas
[x] Las tres raíces activas existen y el escenario hermético `create→archive` de `axiom-bug-receipts.test.ts` pasa; los 4 receipts se conservan bajo `specs/bugs/_archive/<id>/receipts/`.
`specs/increments/`, `specs/bugs/` y `specs/archive/` existen y siguen funcionando: se crea y archiva un artefacto de prueba en entorno hermético para demostrarlo.

### AC-094-07: Validación explicable
[x] `axiom index validate` pasa con 36 `metadata.yml` válidos; el recuento corresponde a los artefactos bajo la raíz resuelta `specs`, sin artefactos gestionados fuera de su alcance.

### AC-094-08: Gobierno respetado
[x] Ningún `metadata.yml`, índice ni receipt fue editado a mano; las nueve decisiones migradas fueron creadas/enlazadas mediante Core y los cambios de estado del incremento tienen receipts Core.

### AC-094-09: Estructura declarada correcta
[x] `Axiom.Spec/README.md` describe la estructura real resultante y declara las raíces top-level legacy como fuera del contrato activo.

### AC-GEN-01: Integridad general
[x] `git diff --check` limpio y el build del runtime en verde; ambos gates pasan según la validación registrada.
