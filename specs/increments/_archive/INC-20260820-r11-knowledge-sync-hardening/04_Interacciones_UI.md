# 04 Interacciones UI

## Objetivo del documento

Documentar los resultados CLI observables de sync/pull seguros.

## Superficie UI afectada

`axiom knowledge sync` y `axiom knowledge pull` en terminal.

## Flujo de interacción

El operador ejecuta sin `--confirm` para previsualizar. Repite con `--confirm` para aplicar localmente; añade `--push` solo cuando desea enviar al remoto. Pull no recibe identificador de incremento.

## Estados visibles

La salida distingue preview/aplicación, chunks/memorias importadas, entradas omitidas por privacidad/ausencia/secretos y chunks que siguen pendientes por corrupción o fallo parcial.

## Cascadas y comportamiento reactivo

No hay UI gráfica. El marker project-scoped evita que una importación local cambie el contenido compartido del repositorio.
