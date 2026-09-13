# 04 Interacciones UI

## Objetivo del documento

Describir qué observa un equipo al instalar, adoptar o actualizar Axiom con la distribución del manual activa.

## Instalación y adopción

```bash
axiom workspace setup
axiom workspace adopt
```

La salida declara, entre las superficies creadas, el manual de producto y su destino. La previsualización previa a la mutación lo lista como cualquier otro archivo generado, de modo que nadie recibe documentación sin saberlo.

## Actualización

```bash
axiom upgrade
```

La salida indica si el manual se refrescó, si ya estaba al día o si quedó pendiente por la política de sobrescritura. Una segunda ejecución no produce cambios.

## Proyecto con el manual editado

El comportamiento coincide con la política declarada en la decisión del incremento. Si se eligió bloque generado con zona propia del equipo, la zona propia se conserva íntegra y solo se regenera el bloque de producto, igual que ya ocurre con `AGENTS.md`.

## Lo que ve un agente en el proyecto

Un agente que trabaje en el repositorio del equipo encuentra el manual en el destino declarado y puede resolver qué hace cada comando, qué escribe y con qué se conecta sin leer el código del producto ni salir del repositorio.
