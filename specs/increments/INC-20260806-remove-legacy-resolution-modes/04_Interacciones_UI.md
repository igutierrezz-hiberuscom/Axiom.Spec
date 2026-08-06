# 04 Interacciones UI

## Objetivo del documento

Registrar el impacto de la retirada del contrato legacy en superficies
operativas.

## Superficie UI afectada

No hay una pantalla nueva ni un selector nuevo. La CLI, doctor, launcher y
MCP consumen la resolución normalizada existente.

## Flujo de interacción

Al abrir un proyecto con un manifiesto raw antiguo, el resolver normaliza la
política a local-only y los comandos continúan por el mismo flujo local.

## Estados visibles

Las salidas pueden mostrar `local-only`. No deben mostrar `gateway` o `hybrid`
como estado efectivo; solo pueden aparecer como valores legacy explicados en
un diagnóstico o migración.

## Cascadas y comportamiento reactivo

La normalización no inicia procesos remotos, no cambia la selección de
providers y no modifica la topología. Los efectos quedan limitados al shape
de `ProjectResolution`.
