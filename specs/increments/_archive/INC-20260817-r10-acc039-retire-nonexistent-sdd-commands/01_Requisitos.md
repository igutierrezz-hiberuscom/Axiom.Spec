# 01 Requisitos

## Objetivo del documento

Eliminar instrucciones activas que lleven a un operador o agente a comandos que la CLI no registra.

## Requisitos del incremento

1. Ninguna superficie activa debe prometer `axiom sdd advance`, `axiom plan` o `axiom role start`.
2. Cada sustitución debe usar el comando real y adecuado: `axiom state`, `axiom-increment`, `axiom-bug`, `axiom-plan` o `axiom-role`.
3. Los 19 intents internos `not-implemented` se conservan intactos.
4. Las referencias históricas se conservan sin reescribir.

## Reglas de negocio relevantes

La sintaxis pública de workflow es parte del contrato materializable; el catálogo no debe anunciar aliases inexistentes.

## Fuera de alcance funcional

No se implementan aliases, comandos nuevos ni cambios en la máquina de estados.
