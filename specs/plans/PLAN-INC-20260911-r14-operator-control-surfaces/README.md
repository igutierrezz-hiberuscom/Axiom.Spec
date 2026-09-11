# R-14 control de operador e interfaces (ACC-081..ACC-086)

> **Código**: PLAN-INC-20260911-r14-operator-control-surfaces
> **Estado**: draft
> **Artefacto origen**: auditoría R-14 registrada en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md` (sesión 2026-09-11)
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

Este plan ordena la ejecución de las siete acciones abiertas por la auditoría de R-14 (`ACC-081`..`ACC-087`) en ocho incrementos. Tiene dos premisas.

La primera: **el lote se desbloquea a sí mismo**. El engine de workflow solo admite un artefacto en vuelo y liga el estado al último creado, así que el primer incremento es precisamente el que corrige esa limitación (`ACC-087`). Se creó en último lugar de forma deliberada para que sea el único que puede avanzar hoy; al cerrarse, el operador podrá elegir libremente el siguiente y el orden dejará de depender del orden de creación.

La segunda: **las decisiones se cierran antes de implementar**. Cinco acciones arrastran una duda que, sin resolver, obligaría a rehacer trabajo o a elegir en caliente. Por eso el segundo incremento no produce código: produce decisiones registradas, y ningún incremento posterior arranca sin la suya cerrada.

Alcance funcional del lote: permitir varios artefactos en vuelo con selección explícita, retirar el paquete residual de la TUI, sanear los artefactos compilados versionados, dejar una sola superficie de self-update, derivar la versión de runtime de una identidad única, dar recomendación de modelo y aviso por adapter donde el enrutado no se obedece, y exponer en el launcher las dos pantallas de operador que hoy no existen.

## Objetivo técnico

Cerrar el hueco entre los mandos de operador que ya existen en la CLI y lo que el operador puede ver y accionar, eliminando a la vez el residuo estructural que la zona arrastra. Al terminar el lote debe cumplirse:

0. Varios artefactos pueden estar en vuelo a la vez y el operador selecciona explícitamente sobre cuál opera, sin re-ligado implícito ni transiciones que se vuelvan legales por el cambio.
1. No queda paquete, enlace ni entrada de lockfile de la TUI, y la prueba de retirada comprueba la ausencia de forma positiva.
2. No hay artefactos compilados versionados dentro de `src/`, la exclusión cubre las salidas anidadas y un gate impide la reaparición.
3. Existe una sola ruta de self-update alcanzable, con «instalar» y «actualizar» separados si el análisis concluye que ambos verbos son necesarios.
4. La versión de runtime del `ManagedState` y el default de `--target-version` derivan de la misma identidad de release que el CLI, sin literales escritos a mano.
5. Donde el destino no enruta modelo, el operador ve el modelo recomendado, sabe que la selección es manual y el prompt copiado lo advierte según el adapter.
6. El launcher expone pantalla de model routing y pantalla de actualización de Axiom sobre los comandos y endpoints canónicos, con los gates de control plane vigentes.

## Alcance incluido

- `ACC-087` estado por instancia y selección explícita del artefacto en vuelo.
- `ACC-081` retirada física del paquete residual `@axiom/tui`.
- `ACC-084` higiene de artefactos compilados versionados, exclusión y gate anti-regresión.
- `ACC-083` decisión y consolidación de una sola superficie de self-update.
- `ACC-085` identidad única de versión de runtime.
- `ACC-086` recomendación de modelo y aviso dependiente del adapter al copiar prompt.
- `ACC-082` pantallas de operador de model routing y de actualización en el launcher.
- Las decisiones `D-01`..`D-05` que condicionan lo anterior, registradas como artefactos de decisión.

## Alcance excluido añadido por ACC-087

- Los carriles `qa-e2e` y `role` del engine de workflow conservan su semántica actual.
- No se habilita ejecución concurrente de dos transiciones: el objetivo es coexistencia y selección, no paralelismo de escritura.
- No se abre ninguna vía para editar el estado a mano ni para saltarse gates de aprobación.

## Alcance excluido

- Convertir un destino `fallback-only` en routing real: `ACC-086` entrega recomendación honesta y acción manual, no control efectivo sobre el IDE.
- Reabrir el contrato de self-update cerrado por `ACC-077`..`ACC-080`: este lote lo consume, no lo redefine.
- Reabrir `ACC-005`: la retirada funcional de la TUI ya está validada; aquí solo se cierra su residuo físico.
- Publicar el CLI como package npm o cambiar el canal de entrega.
- Cualquier cambio en el ciclo SDD, adapters, topología o memoria que no sea consecuencia directa de las seis acciones.

## Roles impactados

`builder` (rol único del producto). El repositorio objetivo de implementación es `Axiom`; la spec canónica y los artefactos de decisión viven en `Axiom.Spec`.

## Fase 0 — cierre de dudas antes de implementar

Esta fase es el propósito principal del plan en su estado actual. Se ejecuta en el incremento `INC-20260911-r14-operator-decisions-baseline` y su salida son decisiones registradas mediante los comandos de Axiom (`axiom-decision` o `axiom-adr` según corresponda), no prosa dentro de este plan.

Va en segunda posición, no en primera, por una razón mecánica: hasta que `INC-20260911-r14-workflow-instance-selection` esté cerrado, el único artefacto que puede avanzar es él. Sus propias dudas de diseño (`D-06`, listadas en su README: forma de la selección, historia de instancias terminales y alcance de la migración a `bug` y `plan`) se resuelven dentro de ese incremento porque no existe forma de precederlo. Ninguna de ellas condiciona a los otros seis.

### D-01 — ¿Qué self-update se conserva? (bloquea ACC-083, ACC-085, ACC-082)

Hecho verificado: `apps/cli/src/index.ts` solo registra `registerSelfUpdateContract`; `apps/cli/src/commands/self-update.ts` conserva `registerSelfUpdate` y 16 pruebas vivas, sin consumidor de producción. Ambas generaciones aterrizaron juntas en el commit del motor transaccional.

Qué hay que decidir, en este orden:

1. ¿La instalación inicial del shim (`scripts/install-global.mjs`, `runInstallGlobal`, backup/restore, `node --test scripts/install-global.test.mjs`) sigue siendo un camino soportado?
2. Si lo es, ¿queda como comando propio con nombre distinto de «actualizar», o como paso interno del contrato transaccional?
3. ¿Qué escenarios cubiertos hoy solo por las 16 pruebas antiguas deben migrarse antes de borrar, y cuáles mueren con el código?

Criterio de cierre: una sola ruta de actualización alcanzable desde la CLI y ninguna función de producción sostenida únicamente por sus propias pruebas.

### D-02 — ¿Qué pasa con los `dist/` de los adapters? (bloquea ACC-084)

Hecho verificado: hay 19 artefactos compilados versionados bajo rutas `src/` y 148 bajo `packages/adapters/*/dist/`; `.gitignore` solo declara `packages/*/dist/` y `apps/*/dist/`, un nivel por encima. `TC-009` (`runAdapterRuntimeCoverageCheck`) exige `src/generator.ts` y `dist/index.js` presentes para los ocho adapters y falla si alguno no está.

Qué hay que decidir: entre (a) dejar de versionar esos `dist/` y garantizar build previo al doctor, (b) relajar `TC-009` para distinguir «no construido» de «adapter ausente» con una señal honesta, o (c) conservarlos versionados de forma deliberada y documentada. Los 19 de `src/` se retiran en cualquiera de los tres casos.

Criterio de cierre: la opción elegida queda registrada con su efecto sobre un clone limpio sin build.

### D-03 — ¿De dónde sale la versión de runtime? (bloquea ACC-085)

Hecho verificado: `packages/versioning/src/version.ts` exporta el literal `RUNTIME_VERSION = '0.1.0'` y alimenta `ManagedState.runtime.version` y el default de `--target-version` de `axiom upgrade`; en paralelo `apps/cli/src/generated-build-metadata.ts` ya expone `version`, `releaseTag`, `commit`, `buildId` y `dirty` generados en build.

Qué hay que decidir: la fuente única, el comportamiento cuando no hay release publicada (fallo explícito frente a valor degradado declarado), el trato de los estados ya persistidos con `0.1.0` y si el default de `--target-version` sigue siendo la versión de runtime o pasa a ser la identidad de producto. La separación que fijó R-13.5-A entre versión del CLI instalado y versión de compatibilidad project-scoped se mantiene: unificar la fuente no colapsa los dos conceptos.

Criterio de cierre: una sola fuente declarada, con la política de ausencia y de migración escrita.

### D-04 — Contrato exacto de recomendación y aviso (bloquea ACC-086 y ACC-082)

Decisión de producto ya tomada por el usuario: donde el destino no enruta, se muestra el modelo recomendado y el usuario lo selecciona a mano; al copiar el prompt, la salida avisa según el adapter seleccionado.

Qué queda por decidir, y es lo que esta fase debe cerrar:

1. Qué es exactamente «modelo recomendado»: hoy la política resuelve una **clase** (`cheap`/`medium`/`strong`/`local`), no un modelo concreto. Hay que decidir si el aviso nombra la clase, un modelo concreto por clase y destino, o ambos.
2. Redacción y forma del aviso para los tres niveles del `SUPPORT_MATRIX`: `multi-mode` (sin aviso), `single-mode` (aviso de alcance global) y `fallback-only` (aviso de selección manual).
3. Dónde se inserta: toda copia de prompt del launcher, o solo las acciones con slot asociado; y si la CLI (`axiom model show`) muestra el mismo texto.
4. Dónde queda «registrada» la limitación que declara `whenPerSubagentRoutingUnsupported: use-default-and-record-limitation`, hoy sin superficie verificada.

Criterio de cierre: contrato redactado y su punto de inserción identificado en `prompt-builder.ts` / `adapter-routing.ts`, sin duplicar el criterio de soporte.

### D-05 — Barrido previo a la retirada de la TUI (bloquea ACC-081)

Hecho verificado: `packages/tui/` conserva `package.json`, `tsconfig.json` y el huérfano `src/flows/preview.ts`; `node_modules/@axiom/tui` es un junction al paquete; `package-lock.json` mantiene su entrada de workspace con ocho dependencias internas; no está en las project references ni en los alias de Vitest, y `src/index.ts` se borró en `c4df64c`.

Qué hay que decidir: si la regeneración del lockfile se acepta en el mismo incremento, y con qué comprobación se sustituye la aserción indirecta de `tui-retirement.test.ts`, que hoy pasa porque no existe `dist/`.

Criterio de cierre: barrido de dependencias declarado limpio y forma de la nueva prueba acordada.

## Secuencia de incrementos

| # | Incremento | Acciones | Depende de | Por qué en esta posición |
| --- | --- | --- | --- | --- |
| 1 | `INC-20260911-r14-workflow-instance-selection` | ACC-087 | — | Desbloquea el lote entero. Se creó en último lugar a propósito para quedar ligado al estado de workflow y ser el único que puede avanzar hoy. Al cerrarse, el resto se puede seleccionar libremente. Sus propias dudas se resuelven dentro del incremento porque no hay forma de precederlo. |
| 2 | `INC-20260911-r14-operator-decisions-baseline` | fase 0 | #1 | Cierra D-01..D-04 y las registra como decisiones. Sin código de producto. Ningún incremento posterior arranca antes de que esté archivado. |
| 3 | `INC-20260911-r14-tui-package-removal` | ACC-081 | D-05 | Trabajo aislado y de bajo riesgo. Retirarlo antes de tocar higiene de artefactos elimina ruido estructural del árbol. |
| 4 | `INC-20260911-r14-build-artifact-hygiene` | ACC-084 | D-02 | Cambia qué se versiona y qué gates existen. Va antes que los incrementos de código para que los diffs posteriores nazcan sobre el árbol ya saneado. |
| 5 | `INC-20260911-r14-single-self-update-surface` | ACC-083 | D-01 | Borra o reubica código antes de que nadie más lo consuma; en particular antes de la pantalla de actualización. |
| 6 | `INC-20260911-r14-runtime-version-identity` | ACC-085 | D-03, #5 | Comparte identidad de release y manifest con la superficie superviviente de self-update, y alimenta lo que la pantalla mostrará. |
| 7 | `INC-20260911-r14-model-recommendation-notice` | ACC-086 | D-04 | Entrega el contrato de recomendación y aviso en la construcción del prompt; la pantalla de modelos lo consume después. |
| 8 | `INC-20260911-r14-launcher-operator-panels` | ACC-082 | #5, #6, #7 | Último por definición: consume la superficie de self-update superviviente, la identidad de versión y el contrato de recomendación. |

Los ocho incrementos existen ya como artefactos, en `specifying` y enlazados a este plan.

## Restricción de ejecución verificada y cómo se neutraliza

El workflow `increment` es hoy un singleton por repositorio. Verificado en ejecución real durante la preparación de este lote:

- `workflow-state.json` mantiene un único registro para `increment`, ligado a un `metadataId`.
- El wrapper de la CLI rechaza cualquier subcomando distinto de `create` cuando el id recibido no coincide con el ligado y el estado no es terminal: `[axiom increment] ID mismatch: se recibió 'A', pero workflow-state.json está ligado a 'B'`.
- `create` es legal solo desde `draft`; una vez que el artefacto está en `specifying`, `create` se rechaza con `No transition for command 'increment-create' from state 'specifying'`.
- `axiom state` es un inspector read-only y no existe comando soportado para reasignar el singleton.

La estrategia adoptada convierte la restricción en la palanca de arranque:

1. Los ocho incrementos se crearon de una vez, dejando en último lugar `INC-20260911-r14-workflow-instance-selection`. El estado quedó ligado a él, verificado en `workflow-state.json`.
2. Por tanto es el único que puede avanzar ahora, y es exactamente el que corrige la limitación.
3. Al archivarse, el estado queda terminal. A partir de ahí el resto del lote se ejecuta en el orden de la tabla: con la corrección aplicada, por selección explícita; y aun sin ella, el bypass por estado terminal permite retomar cualquier artefacto desde su propio `metadata.yml#status`, uno a uno.
4. En ningún caso se edita `workflow-state.json` a mano: el estado solo se mueve con comandos Axiom.

## Estrategia E2E

- **Incremento 1**: dos y tres instancias simultáneas en vuelo; selección explícita; avance de una sin afectar a las otras; id inexistente; instancia terminal; migración del schema con y sin record previo; ausencia de herencia de `vars`; y regresión de que ninguna transición ilegal pasa a ser legal.
- **Incremento 3**: prueba positiva de ausencia del paquete y del subcomando; build, suite completa, doctor y readiness en verde tras regenerar el lockfile.
- **Incremento 4**: gate anti-regresión que falle ante cualquier artefacto compilado versionado; comprobación explícita del comportamiento de `TC-009` en un árbol sin build, según lo decidido en D-02.
- **Incremento 5**: conservar o migrar la cobertura de los escenarios que hoy solo cubren las pruebas de la implementación antigua; regresión de que la ruta borrada ya no es alcanzable.
- **Incremento 6**: identidad de versión coherente entre `axiom --version`, manifest instalado, `ManagedState.runtime.version` y default de `--target-version`; caso de checkout sin release publicada.
- **Incremento 7**: los tres niveles de soporte del `SUPPORT_MATRIX`, con el snapshot por adapter demostrando cuerpo de prompt idéntico y variación solo en cabecera, verbo y aviso.
- **Incremento 8**: extender la matriz hermética existente del launcher con las dos pantallas, incluidos negativos, grants caducados y ausencia de mutación; no crear una matriz paralela.

## Riesgos y dependencias

| Riesgo | Efecto | Mitigación |
| --- | --- | --- |
| El incremento 1 toca el estado de todos los workflows de artefacto | Una regresión ahí bloquea el ciclo SDD completo, no solo este lote | Migración atómica e idempotente, fail-closed intacto, y matriz de pruebas de coexistencia antes de cualquier consumidor |
| El incremento 1 se complica y el lote queda parado detrás de él | Los otros siete no pueden avanzar | El bypass por estado terminal ya permite ejecutar uno a uno; si el incremento 1 se alarga, se puede archivar y continuar el lote en modo secuencial |
| `TC-009` falla en clone limpio si se dejan de versionar los `dist/` de adapters | Doctor rojo sin causa real | Cerrar D-02 antes de tocar el árbol y hacer explícito el comportamiento elegido |
| Cablear la pantalla de actualización contra la superficie que se va a borrar | Retrabajo completo del incremento 8 | Orden fijado: #5 antes de #8 |
| Regenerar el lockfile arrastra cambios no relacionados | Diff ruidoso y difícil de revisar | Regeneración aislada en el incremento 1, revisada aparte |
| Presentar en la pantalla un mando sin efecto | El operador cree que controla el modelo | ACC-086 antes de ACC-082 y honestidad del aviso por nivel de soporte |
| Cobertura DOM del launcher (riesgo residual conocido de R-13.4) | Regresión de UI no detectada | Extender la matriz hermética; no inventar cobertura que no existe |
| Fatiga de secuencia por el singleton mientras el incremento 1 no está cerrado | Tentación de editar `workflow-state.json` a mano | Prohibido: el estado se mueve solo con comandos Axiom |

## Cambios en contexto técnico

Al cierre del lote deben reconciliarse las afirmaciones activas sobre: contrato del estado de workflow y selección de instancia, superficie única de self-update, identidad de versión de producto y de runtime, higiene de artefactos versionados, ausencia del paquete de la TUI, y el contrato de recomendación de modelo con aviso por adapter. Los archivos propietarios se determinan al integrar cada incremento; el índice lo regenera Core, nunca a mano.

## Consolidación y archivado

Cada incremento consolida su conocimiento estable en la spec canónica y el contexto técnico antes de archivarse, y actualiza el estado de su acción en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md` de `propuesto` a `validado` con la evidencia observada. Los receipts y el estado lifecycle los gobierna Core.

## Validaciones y gates

Por incremento: `npm run build`, suites dirigidas del área tocada, `npm run doctor`, `npm run readiness:first-project` y `git diff --check`. Al cierre del lote: suite completa de Vitest y repetición de la evidencia focal de R-14 (35 archivos / 385 tests fueron el punto de partida verificado el 2026-09-11).

## Fuentes y supuestos

- Registro de acciones `ACC-081`..`ACC-087` y sesiones R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia de la restricción de instancia única: `Axiom/packages/workflow/src/{state-store,governed-transition-runner}.ts`, `Axiom/apps/cli/src/commands/{axiom-increment,state-cmd}.ts`, `Axiom/axiom.config/workflows.yaml` y los rechazos reproducidos en vivo.
- Contrato de self-update cerrado por `ACC-077`..`ACC-080` (R-13.5), consumido sin redefinirse.
- Gates de control plane del launcher cerrados por `ACC-070`..`ACC-076` (R-13.4), reutilizados sin redefinirse.
- Supuesto pendiente de confirmación: ninguna de las seis acciones requiere ejecutar un `upgrade` ni un `self-update apply` reales contra una instalación de producción durante la implementación; la verificación se hace con repositorios y homes herméticos, como ya hizo R-13.5.
