# R-15 documentación, spec e historial (ACC-088..ACC-095)

> **Código**: PLAN-INC-20260912-r15-documentation-single-source
> **Estado**: plan-approved
> **Artefacto origen**: auditoría R-15 registrada en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md` (sesión 2026-09-12)
> **Versión de spec**: v1
> **Versión de plan**: p1

## Resumen ejecutivo

Este plan ordena la ejecución de las ocho acciones abiertas por la auditoría de R-15 (`ACC-088`..`ACC-095`) en seis incrementos.

La premisa de origen fue que la documentación de Axiom se había desalineado
del producto porque no existía una comprobación ejecutable. Esa fotografía del
2026-09-12 queda resuelta por el lote: `Axiom/docs/**` tiene cobertura derivada
de Commander, el gate documental está integrado en el runner común y el manual
se distribuye mediante un writer con fuente y hashes verificables.

Alcance funcional: corregir las afirmaciones falsas y los índices, completar el manual hasta cubrir los 50 comandos que la CLI expone, dejar una sola fuente de plantillas, unificar el formato de decisiones y limpiar las raíces legacy del repositorio canónico, hacer verificable el cierre documental, y distribuir el manual a los proyectos que instalan o actualizan Axiom.

## Objetivo técnico

Al terminar el lote debe cumplirse:

1. Ninguna afirmación falsa ni enlace muerto en la documentación activa del runtime, con lo histórico marcado en lugar de mezclado.
2. Cada comando que la CLI expone tiene documentación veraz, y una prueba falla si aparece un comando sin documentar.
3. Existe una sola fuente de plantillas, la del runtime, con el test dorado comparando la fuente completa.
4. El repositorio canónico tiene un único formato y una única raíz de decisiones, sin raíces legacy ni carpetas vacías.
5. Cerrar un incremento o un bug incluye una verificación documental ejecutable, con alcance declarado y salida auditable.
6. Un proyecto que instala, adopta o actualiza Axiom recibe el manual de producto en su repositorio autoral.

## Alcance incluido

- `ACC-091`, `ACC-092`, `ACC-093`: corrección de veracidad e índices, más la decisión de estructura del manual único.
- `ACC-089`: cobertura completa del manual y prueba de cobertura.
- `ACC-090`: fuente única de plantillas en el runtime.
- `ACC-094`: formato único de decisiones y limpieza acotada de raíces.
- `ACC-088`: verificación documental ejecutable en el cierre.
- `ACC-095`: distribución del manual a los proyectos.

## Alcance excluido

- Fusionar o conservar `Axiom.Spec/specs/manuales/**`. Decisión del usuario del 2026-09-12: es material interno de esta instalación y expira con `Axiom.Spec`.
- Retirar plantillas por sospecha de desuso. Las seis sin consumidor conocido se revisan al instalar Axiom sobre Axiom.
- Resolver la coherencia de nomenclatura entre `axiom.spec/`, `<proyecto>.axiom` y `<proyecto>.spec`. Duda registrada en R-15 sin acción; encaja en la instalación de Axiom sobre Axiom.
- Corregir los residuos que el archivado deja en la raíz activa. Duda registrada con evidencia, de naturaleza distinta: es comportamiento, no documentación.
- Redefinir la semántica de archive, el contrato de QA o la resolución de workflows. `ACC-041`, `ACC-043` y `ACC-045` los cubren; este lote los consume.
- Cambiar comportamiento de comandos. Si documentar revela un comportamiento indeseado, se registra aparte.

## Roles impactados

`builder`, rol único del producto. El repositorio de implementación es `Axiom` para los incrementos 1, 2, 3, 5 y 6; el incremento 4 opera sobre artefactos del repositorio canónico `Axiom.Spec` mediante comandos de Axiom. Los artefactos de spec y decisión viven en `Axiom.Spec`.

## Secuencia de incrementos

| # | Incremento | Acciones | Depende de | Por qué en esta posición |
| --- | --- | --- | --- | --- |
| 1 | `INC-20260912-r15-docs-truth-baseline` | ACC-091, ACC-092, ACC-093 + D-01 | — | Primero dejar de afirmar cosas falsas y fijar la estructura. Escribir contenido nuevo sobre una base que miente multiplica el problema, y sin `D-01` el incremento 2 elegiría formato en caliente. Riesgo bajo, sin código de producto. |
| 2 | `INC-20260912-r15-manual-command-coverage` | ACC-089 | #1 | El trabajo grande del lote. Necesita la estructura decidida y los índices limpios. Entrega además la prueba de cobertura, que es la primera red de seguridad real. |
| 3 | `INC-20260912-r15-template-single-source` | ACC-090 | — (independiente) | Independiente del manual, pero toca código y tests de adapters y workflow. Se ejecuta después del trabajo documental para que su diff no se mezcle con el de prosa, y porque su decisión ya está tomada. |
| 4 | `INC-20260912-r15-canonical-roots-dedup` | ACC-094 | #3 | Comparte superficie con #3: ambos reconcilian `Axiom.Spec/README.md`. Va después para que la estructura declarada se toque una sola vez, ya sin plantillas y sin raíces legacy. |
| 5 | `INC-20260912-r15-docs-closure-verification` | ACC-088 | #1, #2 | El gate llega cuando ya hay documentación veraz y completa que verificar. Si llegara antes, bloquearía el cierre de los propios incrementos que arreglan la documentación. |
| 6 | `INC-20260912-r15-manual-distribution` | ACC-095 | #2, #5 | Último por definición: distribuye el manual terminado y ya vigilado. Distribuir antes propagaría documentación incompleta a los proyectos. |

Los seis incrementos se ejecutaron en el orden definido, están archivados por
Core y permanecen enlazados a este plan.

## Fase 0 — decisiones que se cierran antes de implementar

Las decisiones se registran como artefactos con `axiom axiom-decision create`, no como prosa dentro de este plan. Solo `D-01` es transversal; el resto vive dentro del incremento que la necesita.

### D-01 — Estructura del manual único (bloquea #2, #5 y #6)

Se cierra en el incremento 1. Debe fijar: organización del manual, qué constituye una página de familia de comandos, esquema mínimo de cada página, convención de nombres, dónde vive el contenido histórico y cómo se marca. Y una definición operativa de «familia documentada», porque el incremento 2 mide cobertura contra ella y el gate del incremento 5 la reutiliza.

Criterio de cierre: un agente y una persona pueden, con la estructura declarada, saber qué debe contener una página nueva sin preguntar.

### D-02, D-03, D-04 — Contrato del gate documental (dentro de #5)

Alcance por tipo de cambio, forma auditable de declarar que un documento fue revisado, y condición de aviso frente a bloqueo con su progresión. No afectan a los incrementos anteriores, así que no justifican un incremento propio.

### D-05, D-06, D-07 — Contrato de distribución (dentro de #6)

Destino dentro del proyecto, política ante manual editado localmente, y momento de escritura frente a coste en comandos frecuentes. El precedente seguro es el modelo de bloque generado con zona propia del equipo que ya usa `AGENTS.md`.

### Decisión de formato de decisiones (dentro de #4)

Migrar los nueve ADR numerados a formato gestionado, o congelarlos como históricos. La migración da formato único real pero reescribe identificadores muy referenciados; la congelación conserva referencias pero mantiene dos formatos. Debe registrarse con su motivo antes de tocar nada.

## Estrategia E2E

- **#1**: barrido de enlaces relativos sin destinos inexistentes; barrido de vocabulario retirado sin coincidencias fuera de secciones históricas; correspondencia manual-fichero en `docs/configuration/files/`.
- **#2**: prueba de cobertura derivada del programa Commander construido, con demostración de que falla al añadir un comando sin documentar; muestreo de veracidad de opciones contra la ayuda compilada.
- **#3**: comparación exhaustiva de las 45 plantillas antes de retirar la copia canónica; test dorado ampliado con divergencia introducida a propósito; generación de los cuatro adapters con y sin override on-disk.
- **#4**: `axiom index validate` con recuento explicable; creación y archivado de un artefacto de prueba en entorno hermético; barrido de referencias a decisiones sin enlaces rotos.
- **#5**: cierre con documentación sin revisar detectado y nombrado; cierre limpio registrado; cambio sin impacto documental declarado; modos aviso y bloqueo; negativos de gobierno.
- **#6**: instalación, adopción y actualización en repositorios y homes herméticos; idempotencia por contenido; manual editado localmente según política; detección de manual atrasado.

Por incremento: `npm run build`, suites dirigidas del área tocada, `npm run doctor`, `npm run readiness:first-project` y `git diff --check`. Al cierre del lote: suite completa de Vitest y `axiom index validate`.

## Riesgos y dependencias

| Riesgo | Efecto | Mitigación |
| --- | --- | --- |
| El gate documental llega antes de que la documentación esté completa | Bloquea el cierre de los propios incrementos que la arreglan | Orden fijado: #5 después de #1 y #2, y progresión declarada de aviso a bloqueo |
| El incremento 2 es grande y se alarga | El lote se detiene detrás de él | La unidad de cobertura la fija `D-01`: si el volumen es excesivo, se documenta por familia y no por sub-comando, y la prueba de cobertura mide contra esa unidad |
| Retirar `Axiom.Spec/templates/` pierde contenido que solo existía allí | Pérdida silenciosa de material útil | Comparación archivo por archivo y registro de la elección antes de eliminar, como criterio de aceptación |
| Migrar identificadores de decisión rompe referencias | Enlaces muertos en documentación y artefactos | Barrido de referencias como criterio de aceptación; la alternativa de congelar como histórico queda disponible |
| Distribuir el manual pisa documentación propia de un equipo | Pérdida de contenido del cliente | Política de sobrescritura cerrada antes de implementar y precedente de bloque generado con zona propia |
| Documentar revela comportamientos indeseados | Tentación de arreglarlos dentro del incremento documental | Se registran como bug aparte; el incremento documental no cambia comportamiento |
| La prueba de cobertura se convierte en un segundo inventario de comandos | Otro artefacto que mantener a mano | La prueba deriva los comandos del programa Commander construido, no de una lista escrita |
| El gate se implementa como comando suelto | Aparece una segunda vía de cierre que lo esquiva | Enganche obligatorio en el ejecutor de transiciones gobernado, coordinado con `ACC-041`, `ACC-043` y `ACC-045` |

## Cambios en contexto técnico

Al cierre del lote deben reconciliarse las afirmaciones activas sobre: `Axiom/docs/**` como manual único de producto y su cobertura completa, fuente única de plantillas en el runtime y frontera resultante con el repositorio canónico, formato y raíz únicos de decisiones, verificación documental como parte del cierre, y manual de producto entre las superficies que recibe un proyecto adoptante. Los archivos propietarios se determinan al integrar cada incremento; el índice lo regenera Core, nunca a mano.

## Consolidación y archivado

El orquestador consolidó una sola vez el conocimiento estable en la spec
canónica y el contexto técnico después de cerrar los seis incrementos. El plan
integral registra las ocho acciones como `validado`; receipts, freezes y estados
lifecycle fueron gestionados por Core.

## Resultado final

R-15 completó sus seis incrementos y las ocho acciones `ACC-088..ACC-095`.
Todos los incrementos están archivados por Core, incluida
`INC-20260912-r15-manual-distribution`. La integración estable se realizó una
sola vez en `specs/00..08` y `context/**`.

El contrato final mantiene `Axiom/docs/**` como manual único del runtime y
materializa una copia en `docs/axiom/` durante `workspace setup`, `workspace
adopt` y `axiom upgrade`. El bundle se genera desde el árbol de docs antes del
build; el manifest conserva `sourceHash` y hashes SHA-256 por archivo; las
ediciones locales se preservan como `stale` y la versión nueva se separa bajo
`.stale/`; preview, `sync` y `configure` no escriben esa superficie.

Validación final: generador `1/1`, manual distribution `6/6`, setup/adopt/upgrade
y cobertura documental `41/41`, typecheck, build, doctor (0 fallos), readiness, index validate (39
artefactos, 0 fallos) y diff check PASS. La suite global clasificó `3742/3764`
tests verdes en `346/358` archivos; los 22 fallos quedaron fuera de R-15 y
corresponden a contratos/timeouts preexistentes de R-14/R-13 (incluida la
repetición de `workspace-step-reconciliation.test.ts`). No se observó una
regresión nueva del lote.

## Fuentes y supuestos

- Registro de acciones `ACC-088`..`ACC-095` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia inicial, ya resuelta: no había referencias a `Axiom/docs/**` en pruebas, chequeos de doctor ni scripts; `TC-010` y `TC-011` cubrían solo el material de catálogo.
- Evidencia de cobertura final: 50 familias de primer nivel derivadas de la ayuda compilada, prueba `docs-command-coverage` en `5/5`, 84 archivos en el bundle runtime y comparación automatizada contra `Axiom/docs/**`.
- Evidencia de duplicación de plantillas: 45 más 45 archivos con 7 divergentes, recuentos de vocabulario retirado por copia, copias bundleadas en `artifact-skeleton.ts` y `workspace-adapter-templates.ts`.
- Evidencia de raíces: `resolveSpecArtifactRelPath` devolviendo `specs`, `DEFAULT_SPEC_REL_PATH = 'axiom.spec'`, copia suelta de `INC-20260817-r10-acc038-lifecycle-docs` commiteada en `4661172`, y su artefacto real archivado con receipts.
- Evidencia de distribución final: `distributeManual` en `@axiom/document-bootstrap`, llamadas desde `runWorkspaceSetup`/adopt y `runUpgrade`, 84 archivos bundleados, tests stale/preview/manifest, y `defaultAxiomRepoSiblingPath` fijando `../<projectName>.axiom`.
- La verificación se hizo con repositorios y homes herméticos; no se requirió un entorno de producción.
