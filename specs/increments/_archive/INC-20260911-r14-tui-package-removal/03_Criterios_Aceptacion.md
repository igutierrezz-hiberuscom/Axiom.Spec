# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-081-01: Eliminación física en árbol de fuentes
`Axiom/packages/tui/` no existe en el sistema de archivos ni en `git ls-files`.

### AC-081-02: Purga de junction en node_modules
`node_modules/@axiom/tui` no existe en el árbol de dependencias.

### AC-081-03: Saneamiento de package-lock.json
`Axiom/package-lock.json` no contiene referencias al workspace `"packages/tui"`.

### AC-081-04: Prueba positiva de ausencia
`apps/cli/tests/tui-retirement.test.ts` comprueba de forma positiva la inexistencia física del paquete en disco y verifica que `axiom tui` devuelve error de comando desconocido.

### AC-081-05: Integridad general
`npm run build`, `npm run doctor`, `npm run readiness:first-project` y la suite de pruebas pasan limpiamente tras la eliminación.
