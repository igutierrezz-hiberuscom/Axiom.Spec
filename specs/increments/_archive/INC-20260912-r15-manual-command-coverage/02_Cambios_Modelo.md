# 02 Cambios de Modelo

## Objetivo del documento

Detallar qué se crea y cómo queda organizado el manual, y qué código nuevo aparece para vigilar la cobertura.

## Documentación nueva

Páginas para las familias hoy sin cobertura, agrupadas por propósito:

- **Ciclo SDD**: `axiom-increment`, `axiom-bug`, `axiom-plan`, `axiom-role`, `axiom-adr`, `axiom-decision`, `axiom-qa-e2e`, `state`, `phase`, `freeze`, `integrate`, `validate`.
- **Workspace y repositorios**: `workspace`, `topology`, `roles`, `role`, `repo`, `discover`, `bindings`, `member`, `bootstrap`, `eject`, `repair`, `normalize`, `scaffold`.
- **Operador**: `app`, `rollback`, `projects`.
- **Integraciones y contexto**: `memory`, `toolchain`, `mcp`, `knowledge`, `external-sync`, `adapter`, `provider`, `context`, `index`, `capability`.

## Documentación revisada

Las páginas existentes (`init`, `join`, `configure`, `sync`, `start`, `audit`, `upgrade`, `self-update`, `model`, `components`, `skills`, `knowledge`, `doctor`, `support-matrix`) se revisan para ajustarlas a la estructura de `D-01` y al comportamiento vigente. `docs/cli/tui.md` permanece como página histórica.

## Índice

- `Axiom/docs/cli/README.md`: reescrito con el listado completo, numeración correcta y agrupación por propósito.
- `Axiom/docs/README.md`: enlaces actualizados a la nueva organización.

## Código nuevo

- Una prueba de cobertura documental en la suite del repositorio, que obtiene la lista de comandos registrados por la CLI y la compara con las páginas presentes en `Axiom/docs/`. Falla ante comando sin página y ante página sin comando, con mensaje accionable que nombre el elemento descubierto.
- La prueba debe obtener los comandos del programa Commander ya construido, no de una lista escrita a mano, para que no haya un segundo inventario que mantener.

## Contratos o estados afectados

- No cambia ningún contrato de runtime, esquema, estado persistido ni comportamiento de comando.
- Se añade un contrato de repositorio nuevo: añadir un comando obliga a documentarlo, y la suite lo comprueba.

## Estructura resultante esperada

`Axiom/docs/` conserva su árbol de secciones y `docs/cli/` pasa a contener una página por familia documentada, con el índice como única puerta de entrada a esa sección.
