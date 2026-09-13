# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] AC-095-01: Manual presente tras instalar
- [x] AC-095-02: Manual presente tras adoptar
- [x] AC-095-03: Manual actualizado al actualizar
- [x] AC-095-04: Idempotencia demostrada
- [x] AC-095-05: Política de sobrescritura respetada
- [x] AC-095-06: Solo material distribuible
- [x] AC-095-07: Correspondencia verificable
- [x] AC-095-08: Mecanismo único
- [x] AC-GEN-01: Integridad general

### AC-095-01: Manual presente tras instalar
Una instalación en proyecto nuevo deja el manual completo en el destino declarado, y `docs/generated-files.md` lo documenta como superficie recibida.

### AC-095-02: Manual presente tras adoptar
Una adopción de proyecto existente deja el manual en el destino declarado sin pisar documentación propia del equipo.

### AC-095-03: Manual actualizado al actualizar
Un proyecto ya instalado recibe el manual actualizado al actualizar Axiom, según la política declarada.

### AC-095-04: Idempotencia demostrada
Dos ejecuciones consecutivas de la misma operación no producen cambios en la segunda, comparando por contenido.

### AC-095-05: Política de sobrescritura respetada
Con el manual editado localmente, el comportamiento coincide con la política declarada y no se pierde contenido en silencio. Si se eligió bloque generado con zona propia, la zona del equipo se conserva byte a byte.

### AC-095-06: Solo material distribuible
No se distribuye material específico de esta instalación. El conjunto interno del repositorio canónico no aparece en ningún proyecto.

### AC-095-07: Correspondencia verificable
Existe forma de comprobar que el manual del proyecto corresponde a la versión instalada, y un caso negativo demuestra que un manual atrasado se detecta.

### AC-095-08: Mecanismo único
La materialización reutiliza el patrón de catálogo con fuente y huella. No aparece un segundo mecanismo de escritura de documentación.

### AC-GEN-01: Integridad general
`npm run build`, las suites de instalación y de workspace, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` pasan limpiamente, con repositorios y homes herméticos.
