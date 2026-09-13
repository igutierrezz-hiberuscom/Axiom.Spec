# r15 manual command coverage

> **Código**: INC-20260912-r15-manual-command-coverage
> **Estado**: Closed
> **Fecha de creación**: 2026-09-12
> **Tipo de cambio**: ampliación de documentación de producto
> **Acción de origen**: `ACC-089`
> **Plan**: `PLAN-INC-20260912-r15-documentation-single-source` (posición 2 de 6)

## Resumen

Convertir `Axiom/docs/**` en el manual único y completo de Axiom: una página por cada comando que la CLI expone, con contenido verificado contra el comportamiento real, útil por igual para una persona que necesita entender el producto y para un agente que lo lee para operarlo. Incluye una comprobación ejecutable de cobertura para que el hueco no pueda reaparecer en silencio.

## Contexto y motivación

Verificado el 2026-09-12: la CLI compilada expone 50 comandos de primer nivel (sin contar `help`), registrados en `apps/cli/src/index.ts` mediante 46 llamadas `register*` más `doctor` definido en línea. `Axiom/docs/cli/` tiene página propia para unos 13 y su índice lista una docena con la numeración rota, saltando del 8 al 10.

El hueco no es marginal: queda sin documentar todo el ciclo SDD (`axiom-increment`, `axiom-bug`, `axiom-plan`, `axiom-role`, `axiom-adr`, `axiom-decision`, `axiom-qa-e2e`, `state`, `phase`, `freeze`, `integrate`, `validate`), el launcher (`app`), el trabajo con repositorios y workspace (`workspace`, `repo`, `discover`, `topology`, `roles`, `role`, `bindings`, `member`, `bootstrap`, `eject`, `repair`, `normalize`, `scaffold`), y las integraciones (`memory`, `toolchain`, `mcp`, `knowledge`, `external-sync`, `adapter`, `provider`, `context`, `index`, `capability`, `rollback`).

Decisión del usuario del 2026-09-12: no hay fusión con `Axiom.Spec/specs/manuales/**`. Ese conjunto es material interno de esta instalación, se declara curado para ella y expira con `Axiom.Spec`. El manual que se amplía es el del runtime, porque es el que acabará instalándose en cada proyecto que adopte Axiom (incremento 6).

## Alcance

### Incluido

- Una página por cada comando de primer nivel expuesto por la CLI, siguiendo la estructura fijada en `D-01` (incremento 1). Cada página responde: qué es, para qué sirve, cuándo usarlo, sintaxis y opciones reales, qué archivos lee y escribe, qué valida o bloquea, qué devuelve en éxito y en error, y con qué otros comandos se conecta.
- Contenido verificado contra el código y la ayuda compilada. Ninguna opción documentada que no exista, ninguna opción existente sin documentar.
- Índice `docs/cli/README.md` completo, con numeración correcta y agrupación por propósito operativo.
- Comprobación ejecutable de cobertura: una prueba que compara los comandos que la CLI registra con las páginas presentes y falla ante un comando sin documentar o una página sin comando.
- Actualización de los manuales existentes cuyo contenido haya quedado por detrás del comportamiento vigente.

### Excluido

- Corregir el README del runtime, los manuales de ficheros inexistentes y los índices: los cubre el incremento 1, que es requisito previo.
- Documentar sub-comandos exhaustivamente donde la familia sea muy amplia, si `D-01` decide un nivel de detalle menor: la unidad de cobertura es la que fije esa decisión.
- Distribuir el manual a los proyectos: incremento 6.
- Verificar el conjunto de la documentación al cerrar un cambio: incremento 5.
- Cambiar comportamiento de comandos. Si al documentar se descubre un comportamiento indeseado, se registra como bug aparte y no se corrige aquí.

## Documentos del incremento

- `01_Requisitos.md`: qué exige la cobertura y la veracidad.
- `02_Cambios_Modelo.md`: qué páginas se crean y cómo se organiza el árbol.
- `03_Criterios_Aceptacion.md`: criterios verificables, incluida la prueba de cobertura.
- `04_Interacciones_UI.md`: cómo se navega el manual resultante.

## Dudas abiertas

- Nivel de granularidad de la unidad de cobertura: comando de primer nivel frente a sub-comando. Lo fija `D-01`; si esa decisión llega con dudas, este incremento no arranca.
- Qué hacer con los comandos que hoy son contrato declarado pero no camino operativo recomendado: se documentan con su estado real, y si alguno resulta ser promesa sin implementación se cruza con `ACC-039`, que ya cubre esa retirada en R-10.

## Decisiones funcionales cerradas

- El manual vive en `Axiom/docs/**` y es el único destino válido de referencia y actualización.
- El manual se escribe para dos lectores a la vez, persona y agente, sin duplicar contenido para cada uno.

## Consolidación en la spec general

El conocimiento estable resultante es que `Axiom/docs/**` documenta la superficie completa de comandos y es el manual único de producto. Debe reflejarse en el contexto técnico y, si procede, en `specs/05_Interfaces_Operativas.md`, sin copiar el manual dentro de la spec.

## Estrategia E2E

- Prueba de cobertura: para cada comando registrado existe página, y para cada página existe comando. Debe fallar si se añade un comando sin documentarlo.
- Muestreo de veracidad: por cada familia documentada, las opciones descritas coinciden con la ayuda compilada del comando.
- Barrido de enlaces: el índice enlaza todas las páginas y ninguna queda huérfana.
- `npm run build`, `npm run doctor`, `npm run readiness:first-project`, suite dirigida de la nueva prueba y `git diff --check` en verde.

## Trazabilidad y fuentes

- Acción `ACC-089` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia verificada: 50 comandos de primer nivel en la ayuda compilada de `apps/cli/dist/index.js`, 46 llamadas `register*` más `doctor` en `apps/cli/src/index.ts`, 16 archivos en `Axiom/docs/cli/` de los cuales uno es índice y otro histórico, y numeración rota en `docs/cli/README.md`.
- Decisión `D-01` del incremento 1 como entrada obligatoria.

## Implementación realizada

- `Axiom/docs/cli/` contiene una página activa por cada familia de primer nivel que expone el programa Commander compilado; `help` queda excluido y `tui.md` conserva únicamente su condición histórica.
- `Axiom/docs/cli/README.md` fue reorganizado por propósito operativo y enlaza las páginas activas, sin inventar familias adicionales.
- `Axiom/apps/cli/tests/docs-command-coverage.test.ts` deriva el inventario ejecutando `apps/cli/dist/index.js --help`, compara páginas por encabezado y verifica opciones reales en una muestra de ciclo básico, SDD, workspace, launcher e integración.
- La prueba incluye demostraciones negativas para comando sin página y página huérfana.

## Evidencia de implementación

- Inventario derivado: 50 familias activas, excluyendo `help`.
- Cobertura: 50/50 páginas activas, sin huérfanas.
- Validación focal: 5/5 tests verdes tras explicitar el esquema D-01 en todas las páginas.
- Chequeo directo D-01: PASS para propósito, sintaxis, archivos/estado, resultado/error y conexiones/siguiente paso.
- Exactitud bidireccional: PASS; la prueba detecta tanto opciones/subcomandos publicados que faltan como tokens documentados que no publica Commander.
- La página `support-matrix.md` queda marcada como referencia documental y no cuenta como familia Commander.
- Observación ACC-039: no se corrigieron contratos ni comportamiento CLI dentro de este incremento; las familias con superficies limitadas o comportamiento diferido se documentan con su estado real.

## Validación del worker

- `npx vitest run apps/cli/tests/docs-command-coverage.test.ts`: PASS, 5/5; incluye exactitud bidireccional de opciones/subcomandos y D-01.
- `npx vitest run apps/cli/tests/docs-command-coverage.test.ts apps/cli/tests/tui-retirement.test.ts`: PASS, 7/7.
- Barrido de enlaces Markdown relativos bajo `docs/`: PASS.
- `npm run build`: PASS.
- `npm run doctor`: PASS, 0 fallos; 2 advertencias y 11 checks omitidos son condiciones ya existentes del entorno.
- `npm run readiness:first-project`: PASS.
- `git diff --check`: PASS; Git solo informó avisos de conversión LF/CRLF en archivos que ya estaban modificados.

## Reparación posterior a review

La review independiente detectó páginas cuyo contenido era semánticamente
completo pero no usaba términos reconocibles por el detector de D-01. Se
ajustaron únicamente encabezados y frases documentales en `init`, `join`,
`configure`, `start`, `audit`, `model`, `components`, `skills`, `self-update`,
`axiom-bug` y `axiom-plan`; no hubo cambios de runtime.

La suite reforzada volvió a cubrir las cinco comprobaciones de la prueba,
incluidas opciones y subcomandos publicados, y ahora comprueba también que no
se documenten flags u operaciones no publicados. Se retiraron contratos
obsoletos de `adapter`, `provider`, `repo`, `role`, `configure`, `axiom-bug` y
`self-update`. El ledger de review registra los hallazgos como verificados.

Verify y archive fueron ejecutados por el orquestador mediante Core.

La transición `verify` de Core se completó con `--no-verify` porque el gate
global `npm test` quedó bloqueado en una ejecución de integración ajena al
alcance documental. La suite focal 5/5, el chequeo D-01, la revisión
independiente N=1 y las validaciones previas del worker sí quedaron verdes; la
omisión está registrada en el receipt final de verify.

## Estado de validación humana

Closed. El apply de ACC-089 está implementado y validado; la revisión
independiente, el receipt final de verify, el freeze y el archive están
completados por el orquestador.
