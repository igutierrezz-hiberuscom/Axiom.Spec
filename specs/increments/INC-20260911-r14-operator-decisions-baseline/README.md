# r14 operator decisions baseline

> **Código**: INC-20260911-r14-operator-decisions-baseline
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: decisiones, sin código de producto
> **Acciones de origen**: dudas de `ACC-082`..`ACC-086`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 2 de 7, fase 0)

## Resumen

Cerrar las cuatro decisiones transversales que bloquean el resto del lote de R-14 y registrarlas como artefactos de decisión mediante los comandos de Axiom. Este incremento no cambia código de producto: su entregable son las decisiones y el contrato que los incrementos 3 a 7 consumen sin volver a discutirlo.

## Contexto y motivación

La auditoría de R-14 dejó cinco dudas. Cuatro son transversales y, sin resolver, obligarían a decidir en caliente en medio de una implementación o a rehacer trabajo ya hecho. La quinta (`D-05`, barrido previo a la retirada de la TUI) se resuelve dentro de su propio incremento y no forma parte de este.

## Alcance

### Incluido

- **D-01 — ¿Qué self-update se conserva?** Decidir si la instalación inicial del shim sigue siendo un camino soportado y, en su caso, separarla del verbo «actualizar»; decidir qué escenarios de las pruebas antiguas se migran antes de borrar. Bloquea `ACC-083`, y con él `ACC-085` y `ACC-082`.
- **D-02 — ¿Qué pasa con los `dist/` de los adapters?** Elegir entre dejar de versionarlos garantizando build previo al doctor, relajar `TC-009` con una señal honesta que distinga «no construido» de «adapter ausente», o conservarlos versionados de forma deliberada y documentada. Bloquea `ACC-084`.
- **D-03 — ¿De dónde sale la versión de runtime?** Fijar la fuente única, el comportamiento sin release publicada, el trato de los estados ya persistidos con `0.1.0` y si el default de `--target-version` sigue siendo la versión de runtime. Bloquea `ACC-085`.
- **D-04 — Contrato de recomendación y aviso.** Definir qué es «modelo recomendado» cuando la política resuelve una clase y no un modelo concreto, la redacción del aviso para los tres niveles del `SUPPORT_MATRIX`, dónde se inserta y dónde queda registrada la limitación que hoy solo se declara. Bloquea `ACC-086` y condiciona `ACC-082`.
- Registro de cada decisión con su alternativa descartada y su efecto verificable.

### Excluido

- Cualquier cambio en `Axiom/` (código, configuración, pruebas o documentación operativa).
- `D-05`, que pertenece a `INC-20260911-r14-tui-package-removal`.
- Reabrir contratos ya cerrados por `ACC-070`..`ACC-080`.
- Decidir detalles de implementación que cada incremento puede resolver dentro de su propio alcance.

## Dudas abiertas

Las cuatro anteriores son, por definición, el contenido del incremento. La duda de segundo orden a vigilar: si `D-01` concluye que instalación y actualización son verbos distintos, hay que decidir si eso abre un incremento propio en lugar de ampliar `ACC-083`.

## Decisiones funcionales cerradas

Ninguna todavía. La decisión de producto que sí está tomada y entra como premisa, no como pregunta: donde el destino no enruta modelo, se muestra el recomendado y el usuario lo selecciona a mano, con aviso al copiar el prompt según el adapter.

## Consolidación en la spec general

Las decisiones se registran como artefactos de decisión. La spec canónica solo se toca cuando cada incremento posterior integre su conocimiento estable; este incremento no consolida comportamiento porque no lo cambia.

## Estrategia E2E

No aplica: sin cambios ejecutables. La validación es documental y de coherencia. Se comprobará que cada decisión referencia hechos verificados de la auditoría y no supuestos, y que los criterios de cierre de la fase 0 del plan quedan satisfechos uno a uno.

## Trazabilidad y fuentes

Sesión R-14 del 2026-09-11 y acciones `ACC-082`..`ACC-086` en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`; fase 0 de `PLAN-INC-20260911-r14-operator-control-surfaces`.

## Estado de validación humana

Pendiente. El incremento está en `specifying`. Queda bloqueado hasta que `INC-20260911-r14-tui-package-removal` alcance estado terminal, por la restricción de instancia única del workflow `increment` documentada en el plan.
