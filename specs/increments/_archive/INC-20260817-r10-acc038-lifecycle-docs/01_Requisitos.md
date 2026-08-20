# 01 Requisitos

## Objetivo del documento

Definir la corrección documental mínima para que el lifecycle activo no prometa semánticas contrarias al runtime.

## Requisitos del incremento

1. La documentación activa de `axiom upgrade` debe indicar que el primer uso puede crear `managed-state.json`.
2. El comentario activo de los gates debe permitir explícitamente lecturas de filesystem para evaluar precondiciones.
3. No se debe modificar la conducta observable de `upgrade` ni de ningún gate.

## Reglas de negocio relevantes

La documentación operativa y los comentarios de mantenimiento deben describir comportamiento ejecutable; una inicialización de estado gestionado no equivale a una migración funcional adicional.

## Fuera de alcance funcional

No se añaden archivos de estado, gates, flags ni rutas de migración.
