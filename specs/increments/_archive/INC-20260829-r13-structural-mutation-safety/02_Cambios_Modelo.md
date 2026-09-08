# 02 Cambios de Modelo

## Plan

`StructuralMutationPlan` contiene operationId, project identity, expected ownership, desired validated documents, ordered resource changes y derived steps. Su cálculo no escribe.

## Resultados

`StructuralMutationResult` es unión `committed | rejected | failed | recovery-required`, con `ResourceOutcome[]`, warnings y error tipado. Exit 0 solo para committed sin fallo estructural.

## Journal

Intent + hashes/backups + estado por recurso se persiste antes del primer replace. Estados: `prepared`, `committing`, `committed`, `rolling-back`, `rolled-back`, `recovery-required`. Recovery verifica hashes antes de tocar destinos y nunca reemplaza contenido humano cambiado fuera de la transacción.

## Axiom YAML

El bloque gestionado representa identidad/puntero mínimo. El renderer/merger conserva bytes fuera de markers; extensiones humanas no se parsean y reserializan innecesariamente. La preflight detecta markers múltiples, YAML inválido o conflicto semántico.

## Compatibilidad

Conversión única del shape actual solo con identidad inequívoca; no schema topology v1 ni migrador general.
