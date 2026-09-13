# r15 template single source

> **Código**: INC-20260912-r15-template-single-source
> **Estado**: Pendiente de revisión del orquestador
> **Fecha de creación**: 2026-09-12
> **Tipo de cambio**: eliminación de duplicación estructural
> **Acción de origen**: `ACC-090`
> **Plan**: `PLAN-INC-20260912-r15-documentation-single-source` (posición 3 de 6)

## Resumen

Dejar una sola fuente de plantillas. La copia del runtime, `Axiom/axiom.spec/templates/`, pasa a ser la fuente única declarada; la copia de `Axiom.Spec/templates/` se retira después de rescatar lo que aporte, y el contrato deja de presentarla como canónica. El test dorado pasa a comparar la fuente completa en lugar de unos pocos títulos de sección.

## Contexto y motivación

Verificado el 2026-09-12: hay 45 plantillas en `Axiom.Spec/templates/` y 45 con el mismo nombre en `Axiom/axiom.spec/templates/`. Siete divergen en contenido real, no en espacios; en `discovery-provider-overview-template.md` difieren 45 de unas 50 líneas.

La deriva va en la dirección contraria a la declarada. La copia canónica conserva vocabulario retirado en R-04: 13 menciones de overlay, 4 de `product-owner`, 2 de `axiom-gateway`, 2 de `generated-snapshots` y 1 de `enterprise`. La copia del runtime ya describe la configuración `builder` con política local-only y el proveedor `cmm`.

Existe además una tercera copia bundleada en código, en `packages/workflow/src/artifact-skeleton.ts` y `packages/cli-commands/src/commands/workspace-adapter-templates.ts`, cuyo test dorado solo compara títulos de sección de unos pocos archivos, por lo que la divergencia pudo crecer sin que nada fallara. Los generadores de `opencode`, `claude-code`, `codex` y `antigravity` leen `axiom.spec/templates/agents-md-template.md` con precedencia present-file-wins sobre la copia bundleada.

Decisión explícita del usuario del 2026-09-12: gana la copia del runtime por ser la más reciente y la única con vocabulario vigente, y el contrato debe declararlo así.

## Alcance

### Incluido

- Declarar `Axiom/axiom.spec/templates/` como fuente única de plantillas en el contrato, en los comentarios de código que hoy apuntan a la copia canónica y en la documentación activa.
- Comparar plantilla por plantilla las dos copias y rescatar en la fuente elegida el contenido útil que solo exista en la canónica, corrigiendo en el trayecto cualquier vocabulario retirado que quede.
- Retirar `Axiom.Spec/templates/` una vez completado el rescate, y reconciliar la estructura declarada en `Axiom.Spec/README.md`.
- Actualizar el test dorado para que compare la fuente elegida en su totalidad, no solo títulos de sección de un subconjunto, de modo que cualquier divergencia futura entre la copia bundleada y la fuente falle.
- Mantener funcionando el override present-file-wins de los adapters, alineado con la fuente elegida.

### Excluido

- Retirar plantillas por sospecha de desuso. Las seis sin consumidor conocido (`getting-started-template.md`, `troubleshooting-template.md`, `onboarding-member-template.md`, `discovery-provider-overview-template.md`, `artifact-baseline-matrix.md`, `agent-log-template.md`) se conservan; su revisión se aplaza al momento de instalar Axiom sobre Axiom, cuando se pueda observar qué se usa de verdad.
- Renombrar la carpeta `axiom.spec/` ni resolver la coherencia de nomenclatura entre `axiom.spec/`, `<proyecto>.axiom` y `<proyecto>.spec`. Es una duda registrada en R-15 sin acción y encaja en la instalación de Axiom sobre Axiom.
- Cambiar el mecanismo de materialización de skills y agentes.

## Documentos del incremento

- `01_Requisitos.md`: qué exige la fuente única.
- `02_Cambios_Modelo.md`: qué se mueve, se retira y se toca en código.
- `03_Criterios_Aceptacion.md`: criterios verificables.
- `04_Interacciones_UI.md`: efecto observable en generación de superficies.

## Dudas abiertas

- Si alguna de las siete plantillas divergentes contiene en la copia canónica contenido que deba prevalecer sobre el del runtime. Se resuelve en implementación con comparación archivo por archivo y queda registrado el motivo de cada elección.
- Si retirar `Axiom.Spec/templates/` obliga a algún ajuste en el contexto técnico que describe la frontera de repositorios. Se comprueba al integrar.

## Decisiones funcionales cerradas

- La fuente única es la copia del runtime.
- El repositorio canónico deja de alojar plantillas.
- No se retira ninguna plantilla por sospecha de desuso en este incremento.

## Registro de comparación de plantillas

La comparación previa a la retirada cubrió 45/45 archivos por nombre y contenido.
En los 38 archivos no divergentes, la fuente runtime se conserva sin cambios porque
ambas copias eran equivalentes. En las siete divergencias, gana siempre la fuente
runtime: contiene la revisión vigente y la copia Spec conserva rutas, perfiles o
providers retirados. No se retiró ninguna de las seis plantillas sin consumidor
conocido.

| Archivo | Resultado | Motivo |
|---------|-----------|--------|
| `agent-log-template.md` | runtime | Idéntico; conservar runtime |
| `agents-md-template.md` | runtime | `.axiom-state` y manifests actuales; Spec usaba `.sdd` |
| `artifact-baseline-matrix.md` | runtime | Idéntico; conservar runtime |
| `bootstrap-template.md` | runtime | Idéntico; conservar runtime |
| `bug-acceptance-template.md` | runtime | Idéntico; conservar runtime |
| `bug-metadata-template.yaml` | runtime | Idéntico; conservar runtime |
| `bug-model-changes-template.md` | runtime | Idéntico; conservar runtime |
| `bug-requirements-template.md` | runtime | Idéntico; conservar runtime |
| `bug-template.md` | runtime | Idéntico; conservar runtime |
| `bug-ui-interactions-template.md` | runtime | Idéntico; conservar runtime |
| `business-flows-template.md` | runtime | Idéntico; conservar runtime |
| `context-readme-template.md` | runtime | Idéntico; conservar runtime |
| `copilot-instructions.template.md` | runtime | Idéntico; conservar runtime |
| `decision-template.md` | runtime | Idéntico; conservar runtime |
| `discovery-provider-overview-template.md` | runtime | `builder`, `local-only` y `cmm` vigentes; Spec tenía vocabulario retirado |
| `domain-model-template.md` | runtime | Idéntico; conservar runtime |
| `executive-summary-template.md` | runtime | Idéntico; conservar runtime |
| `functional-requirements-template.md` | runtime | Idéntico; conservar runtime |
| `getting-started-template.md` | runtime | Configuración `axiom.config`, `builder` y `local-only` actuales |
| `glossary-template.md` | runtime | Idéntico; conservar runtime |
| `increment-acceptance-template.md` | runtime | Idéntico; conservar runtime |
| `increment-metadata-template.yaml` | runtime | `local-only` y `cmm` vigentes; Spec tenía perfiles y providers retirados |
| `increment-model-changes-template.md` | runtime | Idéntico; conservar runtime |
| `increment-requirements-template.md` | runtime | Idéntico; conservar runtime |
| `increment-template.md` | runtime | Idéntico; conservar runtime |
| `increment-ui-interactions-template.md` | runtime | Idéntico; conservar runtime |
| `integrations-template.md` | runtime | Idéntico; conservar runtime |
| `memory-candidate-template.md` | runtime | Idéntico; conservar runtime |
| `migration-template.md` | runtime | Idéntico; conservar runtime |
| `non-functional-requirements-template.md` | runtime | Idéntico; conservar runtime |
| `onboarding-member-template.md` | runtime | Estado local `.axiom-state`, `builder` y `local-only` actuales |
| `plan-metadata-template.yaml` | runtime | Idéntico; conservar runtime |
| `plan-template.md` | runtime | Idéntico; conservar runtime |
| `product-skill-template.md` | runtime | `builder` y capabilities actuales; Spec tenía `product-owner` y contrato antiguo |
| `role-plan-template.md` | runtime | Idéntico; conservar runtime |
| `security-template.md` | runtime | Idéntico; conservar runtime |
| `skills-lock-template.yaml` | runtime | Idéntico; conservar runtime |
| `spec-template.md` | runtime | Idéntico; conservar runtime |
| `specs-readme-template.md` | runtime | Idéntico; conservar runtime |
| `technical-context-template.md` | runtime | Idéntico; conservar runtime |
| `token-metrics-template.md` | runtime | Idéntico; conservar runtime |
| `troubleshooting-template.md` | runtime | Estado local `.axiom-state`, `builder` y `local-only` actuales |
| `user-interfaces-template.md` | runtime | Idéntico; conservar runtime |
| `verify-multirepo-evidence-manifest-template.yaml` | runtime | Idéntico; conservar runtime |
| `verify-role-global-state-mapping.yaml` | runtime | Idéntico; conservar runtime |

Las seis plantillas conservadas sin consumidor conocido son `agent-log-template.md`,
`artifact-baseline-matrix.md`, `discovery-provider-overview-template.md`,
`getting-started-template.md`, `onboarding-member-template.md` y
`troubleshooting-template.md`.

## Consolidación en la spec general

Conocimiento estable a integrar: existe una sola fuente de plantillas, vive en el runtime, y el override de adapters se resuelve contra ella. Afecta al contexto técnico de arquitectura y datos y a la descripción de la frontera fijada por ADR-0032, que debe reflejar que el repositorio canónico ya no aloja plantillas.

## Estrategia E2E

- Comparación exhaustiva de las 45 plantillas antes de retirar la copia canónica, con registro del resultado por archivo.
- Test dorado ampliado: cualquier diferencia entre copia bundleada y fuente única falla, incluida una diferencia introducida a propósito para demostrar la eficacia.
- Pruebas de generación de los cuatro adapters que leen la plantilla en disco, con y sin override presente.
- Barrido de vocabulario retirado en la fuente única: cero coincidencias fuera de notas históricas explícitas.
- `npm run build`, suites de adapters y workflow, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` en verde.

## Trazabilidad y fuentes

- Acción `ACC-090` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia verificada: 45 plantillas por copia con 7 divergentes; recuentos de vocabulario retirado por copia; `packages/workflow/src/artifact-skeleton.ts`; `packages/cli-commands/src/commands/workspace-adapter-templates.ts`; `packages/adapters/{opencode,claude-code,codex,antigravity}/src/generator.ts` con `DEFAULT_TEMPLATE_PATH = 'axiom.spec/templates/agents-md-template.md'`.

## Estado de validación humana

OK. La revisión independiente final no encontró blockers. La verificación
independiente corrigió referencias de procedencia stale en tres fuentes
bundleadas y retiró cuatro salidas `apps/cli/dist` ignoradas que ya no tenían
fuente actual.

## Resultado de ejecución

- Implementado el test dorado de comparación completa y sincronizadas las copias bundleadas del workflow.
- Retirada físicamente `Axiom.Spec/templates/`; la fuente única queda en `Axiom/axiom.spec/templates/`.
- Actualizados los comentarios de procedencia y la documentación activa de frontera.
- Actualizado el resto de referencias activas que presentaban `Axiom.Spec/templates` como procedencia, incluyendo `workspace-spec-base`, `workspace-skills` y el reporte de migración legacy.
- Reconciliados dos claims de procedencia en la spec canónica activa (`specs/03` y `specs/04`); la ruta retirada queda solo en historia explícita cuando corresponde.
- Limpiadas las salidas generadas stale `apps/cli/dist/commands/workspace-adapter-templates.*`.
- Comparación independiente: 45/45 nombres históricos presentes y exactamente 7 divergencias, coincidentes con el registro de rescate.
- Validación focal independiente: 5 archivos y 97/97 tests en verde.
- `npm run typecheck`: PASS.
- `npm run build`: PASS.
- `npm run doctor`: PASS, 0 fallos; 2 advertencias y 11 checks omitidos son condiciones del entorno.
- `npm run readiness:first-project`: PASS.
- `git diff --check`: PASS.
- Review independiente final N=1: OK; AC-090-01..07 y AC-GEN-01 cumplidos.
- La suite global `npm test` no concluyó porque quedó repitiendo
	`apps/cli/tests/workspace-step-reconciliation.test.ts`; se detuvo sin
	transición parcial y no sustituye la evidencia focal.
- Pendiente: receipt, freeze final y archive del orquestador.
