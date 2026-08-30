# 02 Cambios de Modelo

## Objetivo del documento

Reducir el contrato de entrada de la operación de alta de memoria a los datos que afectan el resultado.

## Entidades o estructuras afectadas

- `MemoryAddArgs`: se elimina `manual`.
- Registro Commander de `memory add`: se elimina la opción `--manual` y su texto de ayuda.

## Contratos o estados afectados

No cambia `MemoryEntry`, `MemoryKind`, el scope del proyecto ni la semántica de `saveMemory`. La persistencia sigue ejecutándose al completar la acción explícita `memory add`.

## Notas de compatibilidad

Los callers programáticos deben dejar de enviar `manual`. Los invocadores CLI con `--manual` reciben un error accionable de opción desconocida en lugar de una falsa configuración aceptada.
