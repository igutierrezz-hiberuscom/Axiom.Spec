# r15 canonical roots dedup

> **Código**: INC-20260912-r15-canonical-roots-dedup
> **Estado**: pending (ejecución completada; pendiente de review/orquestador)
> **Fecha de creación**: 2026-09-12
> **Tipo de cambio**: unificación de formato y limpieza de raíces
> **Acción de origen**: `ACC-094`
> **Plan**: `PLAN-INC-20260912-r15-documentation-single-source` (posición 4 de 6)

## Resumen

Dejar un único formato y una única raíz de decisiones en el repositorio canónico, y retirar solo lo que ninguna operación usa: las raíces legacy de primer nivel, la copia suelta de un incremento que nació en la raíz equivocada y una carpeta vacía. Las raíces activas se conservan, porque quedarse vacías es su estado normal.

## Contexto y motivación

Verificado el 2026-09-12:

- Conviven dos formatos de decisión: nueve documentos numerados a mano en `Axiom.Spec/decisions/` y siete artefactos `DEC-*` gestionados con `metadata.yml` en `specs/decisions/`. Además hay un ADR gestionado en `specs/adr/ADR-0032-toolchain-versioning/` cuyo número choca con un documento distinto, `decisions/0032-axiom-spec-boundary-and-runtime-baseline.md`.
- Las raíces de primer nivel no son las que usa el runtime: `resolveSpecArtifactRelPath` devuelve `specs` para esta autoridad schema-2, y la transición de archive mueve la carpeta a `_archive/` dentro de esa misma raíz.
- `increments/INC-20260817-r10-acc038-lifecycle-docs` es una copia suelta con metadata gestionada, `status: specifying`, sin receipts y con `links.planId: null`, commiteada el 2026-08-20 en `4661172`. El artefacto real de esa acción completó su ciclo y está archivado en `specs/increments/_archive/INC-20260817-r10-acc038-lifecycle-docs` con sus receipts, por lo que retirar la copia no pierde historia.
- Esa copia es invisible para la validación: `index-cmd` escanea solo la raíz resuelta, lo que explica que R-14 reportara 21 metadatas válidos cuando el repositorio tiene 22 fuera de `_archive`.
- `bugs/` de primer nivel conserva dos READMEs legacy sin metadata y `Axiom.Spec/axiom.spec/` es una carpeta sin ningún archivo, con el nombre que el artifact-store usa como proyección por defecto.
- `Axiom.Spec/README.md` sigue afirmando que la raíz `increments/` está vacía.

## Alcance

### Incluido

- Unificar el formato de decisiones: un solo formato y una sola raíz. Los nueve documentos numerados a mano se migran al formato gestionado mediante comandos de Axiom, o se declaran históricos y se congelan de forma explícita; la elección debe quedar registrada con su motivo.
- Resolver el choque de numeración 0032 entre el ADR gestionado y el documento numerado a mano.
- Retirar las raíces legacy `increments/` y `bugs/` de primer nivel, incluida la copia suelta de `INC-20260817-r10-acc038-lifecycle-docs`, cuyo borrado autorizó el usuario el 2026-09-12.
- Eliminar la carpeta vacía `Axiom.Spec/axiom.spec/`.
- Reconciliar `Axiom.Spec/README.md` con la estructura resultante.
- Declarar la frontera con `ACC-035` y `ACC-037`, que cubren residuos bajo `specs/increments/INC-20260809-*`, para no solapar trabajo ni dejar hueco.

### Excluido

- Tocar las raíces activas `specs/increments/`, `specs/bugs/` y `specs/archive/`. Son las que usan los comandos y las que reciben el histórico al archivar; quedarse vacías es su estado normal.
- Resolver los residuos que el archivado deja en la raíz activa. Es una duda registrada en R-15 sin acción, con evidencia de que la reescritura ocurre después del archive, y su naturaleza es de fallo de comportamiento, no de limpieza documental.
- Editar identificadores, estados o índices a mano, y borrar historia con receipts.

## Documentos del incremento

- `01_Requisitos.md`: qué exige la unificación y qué se puede retirar.
- `02_Cambios_Modelo.md`: qué artefactos y carpetas cambian.
- `03_Criterios_Aceptacion.md`: criterios verificables.
- `04_Interacciones_UI.md`: efecto en la operación de artefactos.

## Decisión de formato y tratamiento de 0032

Se migra el contenido de los nueve documentos numerados a `specs/decisions/` como decisiones gestionadas `DEC-*`, mediante `axiom-decision create` de Core. No se conserva `decisions/` como segunda raíz activa. Los IDs emitidos son: `0015 -> DEC-20260913-091553-rvx61p`, `0019 -> DEC-20260913-091603-o23mav`, `0026 -> DEC-20260913-091604-oanqpw`, `0027 -> DEC-20260913-091605-3yq7ef`, `0028 -> DEC-20260913-091606-293kxd`, `0029 -> DEC-20260913-091606-9fj08b`, `0030 -> DEC-20260913-091607-nmmu9u`, `0031 -> DEC-20260913-091608-9mnt1m` y `0032 -> DEC-20260913-091609-fuxoda`.

El choque `0032` se resuelve sin renumeración manual: `ADR-0032-toolchain-versioning` permanece en `specs/adr/` con su ID gestionado, y el documento legacy de boundary queda en `specs/decisions/DEC-20260913-091609-fuxoda`. Ambos READMEs explicitan su procedencia y relación con ACC-094.

## Comparación de contenido antes de retirar raíces

La comparación previa verificó que los dos READMEs legacy de `bugs/` son copias narrativas de bugs ya presentes como artefactos gestionados bajo `specs/bugs/`, sin metadata ni receipts propios. La copia `increments/INC-20260817-r10-acc038-lifecycle-docs/` contiene solo un README/metadata en estado `specifying`, sin receipts; su artefacto real completo está archivado en `specs/increments/_archive/INC-20260817-r10-acc038-lifecycle-docs/` con receipts. Se retira únicamente la copia suelta.

Para `decisions/`, se conservaron los nueve cuerpos útiles dentro de los READMEs gestionados anteriores, con banner `AXIOM:MIGRATED`, nombre histórico y el ID Core nuevo. Los documentos legacy eran nueve registros narrativos distintos de los nueve artefactos gestionados preexistentes de R-14, por lo que no se sobreescribió ninguno.

## Dudas abiertas

- Ninguna para la ejecución. La review debe comprobar la cobertura de referencias activas y los resultados de `axiom index validate`.

## Decisiones funcionales cerradas

- La copia suelta de `INC-20260817-r10-acc038-lifecycle-docs` se borra.
- Las raíces activas se conservan.
- Toda transición de artefacto gestionado se ejecuta con comandos de Axiom.

## Consolidación en la spec general

Conocimiento estable a integrar: el repositorio canónico tiene un único formato y una única raíz de decisiones, y las raíces activas de artefactos son las que resuelve la topología. Afecta a `Axiom.Spec/README.md`, al contexto técnico de modelo de datos y a la descripción del gobierno de artefactos.

## Estrategia E2E

- `axiom index validate` sobre el repositorio canónico antes y después: el recuento de metadatas debe explicarse por completo, sin artefactos invisibles fuera de la raíz resuelta.
- Comprobación de que ningún artefacto gestionado cambió de estado por edición manual: los receipts y el estado los mueve Core.
- Barrido de referencias a los documentos de decisión afectados por la unificación: ninguna referencia activa queda rota.
- Comprobación de que las raíces activas siguen operativas creando y archivando un artefacto de prueba en un entorno hermético.
- `git diff --check` limpio y build del runtime en verde, aunque el cambio sea de repositorio canónico.

## Trazabilidad y fuentes

- Acción `ACC-094` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia verificada: `packages/cli-commands/src/commands/_spec-scope.ts`, `packages/workflow/src/artifact-store.ts` (`DEFAULT_SPEC_REL_PATH = 'axiom.spec'`), `packages/cli-commands/src/commands/index-cmd.ts`, `Axiom.Spec/increments/INC-20260817-r10-acc038-lifecycle-docs/metadata.yml`, `Axiom.Spec/specs/increments/_archive/INC-20260817-r10-acc038-lifecycle-docs/`, commit `4661172`, `Axiom.Spec/decisions/`, `Axiom.Spec/specs/{decisions,adr}/`, `Axiom.Spec/README.md`.
- Acciones relacionadas: `ACC-035` y `ACC-037` de R-07.

## Estado de validación humana

OK. La review inicial encontró y dejó resueltos los faltantes de evidencia de
AC-094-06 y el conteo del escenario de archive; la re-review independiente N=1
confirmó el alcance completo sin blockers.

## Archivos y decisiones emitidas

Se afectaron `specs/decisions/DEC-20260913-091553-rvx61p/` a `DEC-20260913-091609-fuxoda/`, este README y las raíces legacy autorizadas. No se modificó runtime Axiom ni metadata, índices o receipts a mano. Las relaciones con este incremento se emitirán mediante el comando Core `axiom-decision link-increment`.

## Integración general

Pendiente para el orquestador: actualizar únicamente referencias activas y consolidar el hecho estable de raíz/formato único en el archivo propietario correspondiente. No se tocaron `specs/00..08` ni `context/**` en este worker.

## Validación ejecutada

- Candidate freeze conservado sin edición; hash recibido: `dfbc0dc60db5bc775d8bacb49f6d6eeb8182b43e9d4ee8ffbf1fae1c4029b6ae`.
- `axiom index validate` antes y después: OK, 36 `metadata.yml` escaneados, ninguno inválido.
- `npx vitest run packages/workflow/tests/artifact-store.test.ts -t archiveArtifactDir`: 3/3 tests focalizados pasan.
- `npx vitest run apps/cli/tests/axiom-bug-receipts.test.ts --testNamePattern='create→archive emiten 4 receipts success'`: 1/1 escenario pasa (4 tests omitidos por filtro).
- `npm run build` desde `Axiom/`: pasa con exit 0; no hubo cambios de runtime solicitados.
- `git diff --check`: sin errores de whitespace; solo warnings informativos de conversión LF/CRLF en cambios preexistentes del worktree.
- Referencias: no quedan rutas legacy activas en las specs canónicas; solo permanece una mención histórica explícita a `Axiom.Spec/templates/` en el registro de la evolución de templates dentro de `specs/06_Integraciones_y_Capacidades.md`.
- La comparación de referencias activa confirmó que los destinos canónicos de decisiones, ADR y bugs existen; los cuatro tipos de raíz top-level retirados no tienen consumidores activos.
