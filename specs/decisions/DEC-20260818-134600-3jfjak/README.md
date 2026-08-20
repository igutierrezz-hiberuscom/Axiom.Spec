# Retirada de Cavekit y antecedente 0015

## Estado verificable

La metadata Core mantiene esta Decision en estado `proposed`. El 2026-08-18, Core registró el único vínculo disponible: `axiom axiom-decision link-increment --id DEC-20260818-134600-3jfjak --target-id INC-20260818-r10-closure-correction`.

## Decisión operativa

El runtime retiró `@axiom/cavekit-discipline` porque no tenía consumidores productivos. El antecedente 0015 se conserva como historia y no debe presentarse como capacidad runtime vigente.

## Límite del modelo Core

Core ofrece `create`, `link-plan`, `link-increment`, `list` y `external-ref` para Decisions; no ofrece una transición de aceptación, cierre ni supersesión. Por ello este documento no afirma supersesión formal de 0015: la retirada es el hecho operativo verificable y el vínculo al correctivo R-10 aporta la trazabilidad disponible. El título estructural original que contiene «supera» no crea una relación Core adicional.
