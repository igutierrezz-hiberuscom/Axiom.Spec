# 02 Cambios de Modelo

## Objetivo del documento

Detallar qué se materializa, dónde se engancha y qué contratos cambian.

## Estructuras nuevas

- Un catálogo de distribución del manual, con la fuente declarada por página y su huella, siguiendo el patrón de `axiom.config/skills-catalog.yaml` y `agents-catalog.yaml`.
- El destino del manual dentro del proyecto, según la decisión `D-05` de este incremento.
- Un registro del estado de distribución en el proyecto, que permita detectar que el manual quedó atrás respecto a la versión instalada.

## Puntos de enganche

- Inventario de siembra de `apps/cli/src/commands/workspace-setup.ts`, que hoy declara los ficheros y carpetas que recibe el repositorio autoral y no incluye documentación de producto.
- Camino de adopción de `workspace-adopt.ts`, que crea el repositorio autoral `../<projectName>.axiom`.
- Camino de actualización, para que un proyecto ya instalado reciba el manual al actualizar.
- El materializador que ya usan skills y agentes, reutilizado en lugar de duplicado.

## Documentación afectada

- `Axiom/docs/generated-files.md`: debe declarar el manual entre las superficies que un proyecto recibe, con su política de edición.
- El propio manual: una página que explique qué es esa documentación en el proyecto, de dónde viene y qué ocurre si se edita.

## Contratos o estados afectados

- Cambia el contrato de lo que recibe un proyecto adoptante: se añade documentación de producto.
- Puede añadirse un chequeo de diagnóstico que compruebe la correspondencia entre manual distribuido y versión instalada, en línea con `TC-010` y `TC-011`.
- No cambia el comportamiento de ningún comando existente más allá de la superficie añadida.

## Riesgo estructural

Escribir documentación en un repositorio del equipo es una superficie nueva con riesgo de pisar contenido propio. La política de sobrescritura debe estar cerrada antes de implementar, y el modelo de bloque generado con zona propia es el precedente seguro del producto.
