# 04 Interacciones UI

## Objetivo del documento

Definir la respuesta visible cuando la única memoria requerida, Engram, no puede atender una operación.

## Superficie UI afectada

- `axiom memory add|show|query|inventory`.
- Consumers de contexto/recall que usan memoria.
- `axiom doctor` y su salida JSON/humana de checks de memoria.

## Flujo de interacción

Con Engram disponible, el usuario opera los comandos de memoria sin conocer un backend alternativo. Si no está instalado o falla, el comando informa el error de Engram y no persiste ni devuelve resultados inventados. Doctor muestra el check fallido y la remediación de instalar o reparar Engram localmente.

## Estados visibles

No se presentan mensajes de «fallback», «JSON local» o memoria degradada aparentemente válida. La ausencia de backend es un fallo explícito.

## Cascadas y comportamiento reactivo

La indisponibilidad de memoria no instala herramientas, no borra JSON histórico, no ejecuta Git ni cambia el lifecycle del artefacto. Solo afecta la operación de memoria solicitada y el diagnóstico de Doctor.
