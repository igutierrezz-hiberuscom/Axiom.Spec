# 04 Interacciones UI

## Objetivo del documento

Describir el único impacto de interfaz: desaparición de una familia CLI y de un bloque informativo de contexto.

## Superficie UI afectada

- Ayuda raíz de `axiom`.
- Ayuda de subcomandos: deja de existir `axiom learn`.
- Salida humana y JSON de `axiom context status`: deja de incluir lecciones recientes.

## Flujo de interacción

No hay un flujo sustitutivo. Las notas de memoria explícitas continúan por la superficie de memoria general definida por el runtime.

## Estados visibles

El usuario no ve candidatos, capturas o listas de lecciones. La auditoría conserva sus estados previos.

## Cascadas y comportamiento reactivo

La retirada de `learn` no debe cambiar la ejecución de `audit`, `doctor`, `context` fuera del bloque eliminado ni la persistencia general de memoria.
