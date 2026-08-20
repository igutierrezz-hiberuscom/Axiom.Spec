# 01 Requisitos

## Objetivo del documento

Eliminar un paquete privado sin consumidor runtime y reconciliar las referencias que lo promueven como operativo.

## Requisitos del incremento

1. No queda package, project reference, test propio ni dependencia directa de Cavekit en el runtime.
2. Un barrido de imports confirma ausencia de consumidores productivos antes y después de la retirada.
3. `zod` permanece mientras otros packages lo requieran.
4. La decisión 0015 se preserva y queda registrada como superada mediante Core.
5. Specs, contexto e inventarios activos no presentan Cavekit como capacidad vigente.

## Reglas de negocio relevantes

Una decisión histórica se conserva como antecedente; el estado de superación no autoriza reescritura ni borrado de historia.

## Fuera de alcance funcional

No se implementa disciplina de reemplazo ni se desinstalan dependencias transitivas o compartidas.
