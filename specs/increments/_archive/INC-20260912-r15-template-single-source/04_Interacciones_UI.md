# 04 Interacciones UI

## Objetivo del documento

Describir qué observa el operador cuando Axiom genera superficies a partir de plantillas, antes y después del cambio.

## Comportamiento observable

Sin cambios funcionales esperados. Los comandos que materializan superficies a partir de plantillas siguen produciendo el mismo resultado:

```bash
axiom configure
axiom sync
```

La resolución de la plantilla mantiene su precedencia: si existe `axiom.spec/templates/agents-md-template.md` en el proyecto, gana sobre la copia bundleada; si no existe, se usa la bundleada.

## Diferencia perceptible

La única diferencia observable para quien trabaja en el workspace es que `Axiom.Spec/templates/` deja de existir. Cualquier flujo que abriera esa carpeta debe pasar a `Axiom/axiom.spec/templates/`.

## Señal ante divergencia futura

Si alguien modifica una plantilla de la fuente única sin actualizar la copia bundleada del código, la suite falla nombrando la plantilla afectada. Antes esa divergencia podía pasar inadvertida.

## Sin interacciones nuevas de CLI ni de launcher

No se registra, modifica ni retira ningún comando, y no cambia ninguna pantalla del launcher.
