# 01 Requisitos

## Objetivo del documento

Fijar la fuente y la semántica únicas de resolución del grafo de workflows.

## Requisitos del incremento

1. El YAML de `Axiom/axiom.config` es la única representación editable del grafo distribuible.
2. Todo consumer obtiene el mismo `WorkflowsConfig` mediante un resolvedor público compartido.
3. Ausencia de YAML usa el default derivado/empaquetado; YAML presente válido tiene precedencia.
4. YAML presente inválido o schema no soportado falla de forma tipada y no hace fallback.
5. Los subcomandos públicos alcanzables son verificables contra las transiciones del grafo.

## Reglas de negocio relevantes

El fallback cubre solo ausencia; no debe ocultar errores de configuración ni drift de schema.

## Fuera de alcance funcional

No se mueven effects, no se aplica confirmación, no se cambian estados ni se ejecutan transiciones en este incremento.
