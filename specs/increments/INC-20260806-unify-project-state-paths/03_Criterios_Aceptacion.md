# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] Existe un helper compartido para derivar `projectKey` y paths.
- [x] Ningún writer nuevo usa `.axiom-state/config/<projectName>/`.
- [x] Memoria, workflow y MCP bindings dejan de vivir bajo `local` al
	escribir.
- [x] `local` queda limitado a datos repo/operador-locales.
- [x] Lectura y migración legacy son deterministas, atómicas, idempotentes y
	no mantienen doble escritura.
- [x] Dos proyectos y dos ejecuciones no comparten rutas.
- [x] Build, doctor, readiness y pruebas focalizadas pasan.
- [x] Checkpoint restore, toolchain alias migration y worktree provider
	selection tienen regresiones explícitas.
- [x] La documentación activa coincide con las rutas emitidas.

### Happy path

Un proyecto v1 o v2 inicializa, configura, sincroniza y arranca bajo
`.axiom-state/<projectKey>/`. Dos proyectos v2 con `name` distinto no
colisionan, y cada ejecución usa su propio subtree de `executions/`.

### Validaciones y errores

Un estado legacy se puede leer y migrar sin excepción. Si existen copias
canonical y legacy, la canonical gana y se emite un warning de conflicto. Un
checkpoint legacy se puede listar y restaurar a destinos canónicos.

### Permisos y visibilidad

El cambio no amplía permisos ni copia `.axiom-state/local/` a worktrees.
Provider selection y toolchain detection se mantienen limitados al
`projectKey`/aliases explícitos del proyecto activo.

### Estados y efectos observables

Después de una migración exitosa el writer futuro solo toca el namespace
canónico. `config` sigue funcionando para callers existentes, pero no aparece
como segmento físico nuevo.
