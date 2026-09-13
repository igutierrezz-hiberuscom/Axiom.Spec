# 01 Requisitos

## Objetivo del documento

Fijar qué debe cumplir el manual único para considerarse completo y veraz.

## Requisitos del incremento

### RQ-089-01 Cobertura completa

Cada comando de primer nivel que la CLI expone tiene documentación en `Axiom/docs/**` conforme a la unidad de cobertura fijada en `D-01`. La lista verificada el 2026-09-12 es: `init`, `join`, `configure`, `sync`, `start`, `audit`, `upgrade`, `rollback`, `model`, `components`, `skills`, `projects`, `self-update`, `topology`, `roles`, `bindings`, `member`, `axiom-increment`, `axiom-bug`, `axiom-role`, `axiom-plan`, `axiom-adr`, `axiom-decision`, `axiom-qa-e2e`, `scaffold`, `normalize`, `integrate`, `state`, `external-sync`, `toolchain`, `memory`, `mcp`, `knowledge`, `app`, `capability`, `repo`, `discover`, `adapter`, `provider`, `role`, `workspace`, `context`, `index`, `validate`, `bootstrap`, `eject`, `repair`, `phase`, `freeze`, `doctor`.

La lista se recalcula en implementación contra la CLI del momento: si aparece o desaparece un comando, manda el runtime, no esta enumeración.

### RQ-089-02 Contenido mínimo por página

Cada página responde, como mínimo: qué es y para qué sirve, cuándo conviene usarlo, sintaxis y opciones reales, qué lee y qué escribe, qué valida o qué puede bloquear la operación, qué se ve en éxito y en error, y con qué comandos se relaciona antes y después.

### RQ-089-03 Veracidad verificable

Ninguna opción documentada puede ser inexistente y ninguna opción existente puede quedar sin documentar. La comprobación se hace contra la ayuda del comando y, cuando el comportamiento no se deduce de la ayuda, contra el código o una ejecución observada.

### RQ-089-04 Cobertura vigilada por ejecución

Existe una prueba que compara los comandos registrados por la CLI con las páginas del manual y falla ante un comando sin documentar o una página sin comando. La prueba forma parte de la suite del repositorio, no de un script suelto.

### RQ-089-05 Índice utilizable

`docs/cli/README.md` enlaza todas las páginas, con numeración correcta y agrupadas por propósito operativo, distinguiendo el ciclo básico de proyecto, el ciclo SDD, la operación de operador y las integraciones.

### RQ-089-06 Destino único

Toda referencia de consumo y de actualización apunta a `Axiom/docs/**`. No se crea, ni se conserva, un conjunto paralelo de manuales en ningún otro repositorio.

### RQ-089-07 Doble lector

El texto sirve a una persona y a un agente sin duplicarse: lenguaje directo, contratos explícitos y ninguna instrucción que dependa de conocimiento tácito del repositorio.

## Reglas de negocio relevantes

- Si documentar revela un comportamiento incorrecto o una promesa sin implementación, se registra aparte como bug o se cruza con la acción que ya lo cubre; no se corrige en este incremento.
- Lo histórico se marca, no se borra.
- La estructura la manda `D-01`. Este incremento no la redefine.

## Fuera de alcance funcional

- Correcciones de veracidad del incremento 1.
- Gate de verificación documental al cerrar cambios.
- Distribución del manual a proyectos adoptantes.
- Cambios de comportamiento en cualquier comando.
