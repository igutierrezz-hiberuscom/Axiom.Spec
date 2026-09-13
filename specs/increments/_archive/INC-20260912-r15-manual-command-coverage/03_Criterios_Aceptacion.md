# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-089-01: Cobertura completa
Para cada comando de primer nivel que la CLI registra existe documentación en `Axiom/docs/**` conforme a la unidad de cobertura fijada en `D-01`. No queda ningún comando sin página.

### AC-089-02: Sin páginas huérfanas
No existe página de comando en el manual que no corresponda a un comando registrado, salvo las marcadas explícitamente como históricas.

### AC-089-03: Contenido mínimo presente
Cada página incluye propósito, momento de uso, sintaxis y opciones reales, archivos que lee y escribe, validaciones o bloqueos, salida esperada en éxito y en error, y relación con los comandos anterior y siguiente.

### AC-089-04: Opciones veraces
Para una muestra que cubra al menos una familia de cada grupo del índice, las opciones documentadas coinciden exactamente con las que expone la ayuda del comando: ni sobra ninguna ni falta ninguna.

### AC-089-05: Cobertura vigilada
Existe una prueba en la suite que deriva los comandos del programa Commander construido y falla ante un comando sin documentar o una página sin comando, con mensaje que nombra el elemento afectado. Se demuestra su eficacia añadiendo temporalmente un comando de prueba y comprobando que la prueba falla.

### AC-089-06: Índice completo y navegable
`docs/cli/README.md` enlaza todas las páginas, sin numeración rota, agrupadas por propósito operativo, y `docs/README.md` apunta a la organización resultante.

### AC-089-07: Destino único
No existe conjunto paralelo de manuales en ningún otro repositorio del workspace, y ninguna referencia activa apunta a uno.

### AC-GEN-01: Integridad general
`npm run build`, `npm run doctor`, `npm run readiness:first-project`, la prueba de cobertura y `git diff --check` pasan limpiamente.
