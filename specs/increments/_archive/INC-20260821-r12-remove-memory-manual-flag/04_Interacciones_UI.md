# 04 Interacciones UI

## Objetivo del documento

Describir la simplificación de la interfaz CLI de guardado manual de memoria.

## Superficie UI afectada

- `axiom memory add --help`: desaparece `--manual`.
- Error de CLI: un uso posterior de `--manual` se informa como opción desconocida.

## Flujo de interacción

El operador o agente elige el texto, kind y metadatos opcionales y ejecuta `axiom memory add`. Esa llamada es suficiente para guardar explícitamente.

## Estados visibles

No se muestra una regla distinta por kind ni una vía alternativa para forzar la persistencia.

## Cascadas y comportamiento reactivo

No modifica las lecturas, consultas, auditoría, visibilidad ni el backend de memoria.
