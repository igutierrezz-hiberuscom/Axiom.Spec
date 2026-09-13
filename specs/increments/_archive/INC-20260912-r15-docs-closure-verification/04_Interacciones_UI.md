# 04 Interacciones UI

## Objetivo del documento

Describir qué ve el operador al cerrar un artefacto con la verificación documental activa.

## Interacción en CLI

Previsualización antes de cerrar:

```bash
axiom integrate --kind increment --id <id> --dry-run
```

La previsualización incluye, además de la transición y los efectos declarados, la lista de documentación en alcance y su estado de revisión.

Cierre confirmado:

```bash
axiom integrate --kind increment --id <id> --confirm
```

En modo aviso, un documento en alcance sin revisar produce una advertencia que lo nombra y explica por qué entró en alcance, y el cierre continúa. En modo bloqueo, el cierre se rechaza con el mismo mensaje y una indicación de qué hacer, sin dejar el artefacto en estado inconsistente.

## Interacción en el launcher

La pantalla de ciclo SDD muestra el mismo resultado que la CLI, porque consume el mismo ejecutor. La confirmación de cierre presenta la lista de documentación en alcance antes de mutar.

## Cambio sin impacto documental

Cuando el alcance calculado está vacío, la salida lo declara explícitamente en lugar de omitir la sección. La ausencia de impacto documental queda registrada, no supuesta.
