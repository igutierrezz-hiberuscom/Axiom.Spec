# 04 Interacciones UI

## Objetivo del documento

Describir el efecto de este incremento en la navegación de la documentación, única superficie de interacción implicada. No hay cambios en CLI ni en el launcher.

## Recorrido esperado del lector

1. Entra por `Axiom/README.md` y distingue en el primer barrido qué describe el estado vigente y qué es historia conservada.
2. Salta al índice `Axiom/docs/README.md` y encuentra todo el contenido existente, con lo histórico identificado como tal en lugar de ausente del índice.
3. Al buscar un fichero de configuración, entra en `docs/configuration/files/README.md` y ve separados los manuales de ficheros que existen y los contratos históricos no materializados.
4. Ningún enlace que siga lo lleva a una ruta inexistente.

## Recorrido esperado de un agente

Un agente que lea `Axiom/docs/**` para operar el producto obtiene contratos vigentes. Si encuentra una página histórica, la marca explícita del encabezado le permite descartarla como instrucción operativa, igual que hoy ocurre con `docs/cli/tui.md`.

## Sin interacciones de CLI ni de launcher

Este incremento no registra, modifica ni retira comandos, y no altera pantallas del launcher. La única salida observable es documentación.
