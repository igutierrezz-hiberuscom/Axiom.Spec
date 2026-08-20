# 04 Interacciones UI

## Objetivo del documento

La investigación no añadió interfaz. Su salida observable fue un informe de clasificación para dirigir correcciones R-10.

## Superficie UI afectada

No se modificaron CLI, launcher ni MCP durante ACC-044. El correctivo posterior sí fijó la interacción pública de targets: los ocho IDs canónicos son aceptados y `copilot-vscode` se rechaza.

## Flujo de interacción

1. Se compara un claim documental con código, configuración y pruebas.
2. Se registra si es contrato activo, divergencia de implementación, intención futura o historia.
3. Cada divergencia de producto se deriva a un ACC o correctivo con alcance explícito.

## Estados visibles

ACC-044 está archivado y su informe es histórico. No constituye una confirmación de cierre del runtime; el estado actual se verifica en el correctivo y el código.

## Cascadas y comportamiento reactivo

No hay cascadas de UI. La corrección posterior de `configure` migra sólo datos persistidos legacy antes de sus dispatches; no expone una interacción alternativa al usuario.
