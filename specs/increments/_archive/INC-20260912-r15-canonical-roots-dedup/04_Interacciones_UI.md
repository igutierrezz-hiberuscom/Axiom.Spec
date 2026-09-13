# 04 Interacciones UI

## Objetivo del documento

Describir qué cambia para quien opera artefactos con la CLI y qué debe seguir funcionando igual.

## Operación de artefactos

Sin cambios en los comandos. La creación, avance y archivado siguen operándose igual:

```bash
axiom axiom-increment create --id <id> --slug <slug>
axiom axiom-increment list
axiom index validate
```

La raíz de artefactos sigue siendo la que resuelve la topología (`specs`), y el archivado sigue moviendo la carpeta a `_archive/` dentro de esa misma raíz.

## Diferencia perceptible

- Las carpetas `increments/` y `bugs/` de primer nivel del repositorio canónico desaparecen. Quien las abriera debe usar `specs/increments/` y `specs/bugs/`.
- Las decisiones se consultan en una sola raíz y con un solo formato.
- `axiom index validate` deja de tener un artefacto invisible fuera de su alcance de escaneo, así que su recuento coincide con lo que hay.

## Sin interacciones nuevas de CLI ni de launcher

No se registra, modifica ni retira ningún comando, y no cambia ninguna pantalla del launcher.
