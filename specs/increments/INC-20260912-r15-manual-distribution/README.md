# r15 manual distribution

> **Código**: INC-20260912-r15-manual-distribution
> **Estado**: Pendiente
> **Fecha de creación**: 2026-09-12
> **Tipo de cambio**: nueva superficie de distribución
> **Acción de origen**: `ACC-095`
> **Plan**: `PLAN-INC-20260912-r15-documentation-single-source` (posición 6 de 6)

## Resumen

Hacer que el manual de producto llegue a los proyectos que instalan, adoptan o actualizan Axiom. Hoy no viaja ningún manual: el equipo que adopta Axiom recibe el esqueleto de la especificación y la configuración, pero no puede leer en su propio repositorio qué es cada comando ni cómo se usa.

## Contexto y motivación

Verificado el 2026-09-12 sobre el inventario de `apps/cli/src/commands/workspace-setup.ts`: el repositorio autoral de un proyecto recibe `specs/README.md`, los nueve documentos `00..08`, `context/TECHNICAL_CONTEXT.md`, `context/README.md`, las carpetas `specs/{increments,bugs,archive}` y `context/{architecture,integrations,operations,references}`, los YAML de `axiom.config`, `axiom.skills.lock` y `.axiom/mcp.yml`. No recibe `docs/**` ni `templates/`. `docs/generated-files.md` confirma que lo que llega por target son ficheros de instrucciones y de estado, no manuales.

Petición explícita del usuario del 2026-09-12: el manual único debe materializarse en el proyecto adoptante, para que el equipo entienda el producto sin salir de su repositorio y para que un agente que lea ese repositorio entienda Axiom. La convención del repositorio autoral es `../<projectName>.axiom`, fijada por `defaultAxiomRepoSiblingPath` en `workspace-adopt.ts`.

Existe ya un mecanismo probado de materialización de contenido de producto en un proyecto: los catálogos de skills y agentes, con fuente declarada y huella verificada por `TC-010` y `TC-011`. La distribución del manual debe apoyarse en ese patrón en lugar de inventar un segundo camino de escritura.

## Alcance

### Incluido

- Destino del manual dentro del proyecto, declarado y documentado.
- Momento de escritura: instalación, adopción y actualización, con el mismo contrato en las tres.
- Idempotencia y política de sobrescritura frente a ediciones locales, coherente con el resto de superficies generadas del producto.
- Separación entre manual genérico de producto, que se distribuye, y material específico de una instalación concreta, que no.
- Reutilización del mecanismo de materialización por catálogo con fuente y huella, en lugar de una copia de carpeta sin control.
- Verificación de que el manual distribuido corresponde a la fuente y no ha quedado atrás tras una actualización.

### Excluido

- Escribir o completar el manual: lo entrega el incremento 2, que es requisito previo.
- Distribuir `Axiom.Spec/specs/manuales/**`, material interno de esta instalación que expira con ese repositorio.
- Convertir el manual en contenido editable por el proyecto con garantía de preservación, salvo que la política de sobrescritura elegida lo contemple explícitamente.
- Publicar el manual fuera del proyecto (sitio web, paquete npm, portal).

## Documentos del incremento

- `01_Requisitos.md`: qué exige la distribución.
- `02_Cambios_Modelo.md`: qué se materializa y dónde se engancha.
- `03_Criterios_Aceptacion.md`: criterios verificables.
- `04_Interacciones_UI.md`: qué observa el equipo que instala o actualiza.

## Dudas abiertas

Se cierran dentro de este incremento. Las decisiones Core son
`DEC-20260913-150856-rem5pr` (D-05), `DEC-20260913-150856-jqjert` (D-06) y
`DEC-20260913-150857-yznm5g` (D-07), enlazadas a este incremento y al plan R-15.

- **D-05 destino**: dónde vive el manual dentro del proyecto y cómo se distingue de la documentación propia del equipo.
- **D-06 sobrescritura**: qué ocurre cuando el proyecto editó el manual distribuido. Preservar, sobrescribir avisando o mantener bloque generado con zona propia del equipo, como ya se hace con `AGENTS.md`.
- **D-07 momento y coste**: si la distribución ocurre en cada `configure`/`sync` o solo en instalación, adopción y actualización explícita.

## Decisiones de ejecución

- **D-05, destino:** el manual genérico vive en `docs/axiom/` dentro del
	repositorio autoral del proyecto. La carpeta es visible y separa el manual
	del producto de la documentación propia del equipo.
- **D-06, sobrescritura:** los archivos intactos se actualizan por contenido y
	los editados localmente se preservan, se marcan `stale` y reciben una salida
	actualizada separada o diagnóstico accionable. Nunca se pisa contenido propio
	fuera de `docs/axiom/` ni se pierde edición en silencio.
- **D-07, momento:** setup, adopción y `upgrade` explícito distribuyen el
	manual; `sync` y `configure` no lo hacen para mantener acotado el coste de
	comandos frecuentes. Las tres operaciones son idempotentes.

## Decisiones funcionales cerradas

- El manual distribuido es el del runtime, único destino de referencia y actualización.
- La distribución reutiliza el patrón de materialización por catálogo con fuente y huella.

## Supuestos de ejecución

- La fuente se resuelve desde `Axiom/docs/**` durante el build y se embebe en
	constantes TypeScript; `tsc` no se usa para copiar assets.
- El materializador escribe exclusivamente bajo `docs/axiom/` y mantiene un
	manifiesto de correspondencia en ese mismo destino. La edición local se
	detecta comparando el hash actual con la huella previamente almacenada.
- Un archivo editado se conserva byte a byte, se marca `stale` en el manifiesto
	y se genera una salida actualizada separada, sin modificar `docs/` ajeno.

## Nota de implementación

La implementación se limita al bundle/materializador compartido de
`packages/document-bootstrap`, sus invocaciones explícitas desde setup/adopt y
upgrade, documentación runtime y pruebas focales. No se modifican metadatos de
freeze, receipts, índices gobernados ni incrementos previos.

## Consolidación en la spec general

Conocimiento estable a integrar: un proyecto que adopta Axiom recibe el manual de producto en su repositorio autoral, con contrato de idempotencia declarado. Afecta a `specs/05_Interfaces_Operativas.md`, a `specs/03_Modelo_Operativo_y_Datos.md`, al contexto técnico de instalación y onboarding, y a `docs/generated-files.md`.

## Estrategia E2E

- Instalación en proyecto nuevo: el manual aparece en el destino declarado, completo.
- Adopción de proyecto existente: el manual aparece sin pisar documentación propia del equipo.
- Actualización: el manual se refresca según la política elegida, y una segunda ejecución no produce cambios.
- Proyecto con el manual editado a mano: el comportamiento coincide con la política declarada y nunca pierde contenido en silencio.
- Verificación de correspondencia entre manual distribuido y fuente, con caso negativo de manual atrasado.
- `npm run build`, suites de instalación y de workspace, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` en verde, con repositorios y homes herméticos.

## Trazabilidad y fuentes

- Acción `ACC-095` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia verificada: inventario de `apps/cli/src/commands/workspace-setup.ts`, `Axiom/docs/generated-files.md`, `defaultAxiomRepoSiblingPath` en `apps/cli/src/commands/workspace-adopt.ts`, catálogos `axiom.config/{skills,agents}-catalog.yaml` con fuente y huella verificadas por `TC-010` y `TC-011`.
- Requisito previo: incremento 2 del lote.

## Estado de validación humana

Pendiente. Las decisiones están cerradas y enlazadas; el incremento se prepara
para implementación después de `refine`, `specify`, `plan`, `plan-approve` y
freeze gobernados por Core.

## Implementation notes

- `@axiom/document-bootstrap` embebe el árbol completo de `Axiom/docs/**` como
	constantes TypeScript generadas y expone un único `distributeManual`.
- El destino es `docs/axiom/`; `manifest.json` registra hashes SHA-256
	estables. Un archivo editado se conserva y la versión nueva queda en
	`docs/axiom/.stale/`.
- `runWorkspaceSetup` es el punto compartido por setup y adopt; `runUpgrade`
	lo invoca solo en upgrade explícito y en preview no escribe.

## Validation

- `node --test scripts/generate-manual-bundle.test.mjs`: PASS (1 test; compara
	byte a byte el bundle generado con `Axiom/docs/**`).
- `packages/document-bootstrap/tests/manual-distribution.test.ts`: PASS (5
	tests; idempotencia, stale, preview y manifests inválidos).
- Suites de setup, adopt y upgrade focales: PASS (30 tests).
- `npm run typecheck`: PASS.
- `npm run build`: PASS; regenera `manual-bundle.generated.ts` antes de `tsc`.
- `npm run doctor`: PASS, 0 fallos, 2 advertencias y 11 checks omitidos por
	condiciones preexistentes del repositorio.
- `npm run readiness:first-project`: PASS.
- `git diff --check`: PASS; solo mostró avisos CRLF de archivos ya modificados
	en el worktree.

## Independent review

La revisión independiente N=1 confirmó AC-095-01..08 y no encontró blockers
funcionales. REVIEW-001 quedó resuelto al alinear el checkbox de AC-GEN-01 con
la evidencia final. REVIEW-002 quedó resuelto al añadir una comparación
automatizada entre `manual-bundle.generated.ts` y la salida actual de
`Axiom/docs/**`. El resultado permanece `pending` hasta completar receipt,
freeze y archive gobernados por Core.

## Result

Implementación realizada; estado `pending` hasta completar los gates globales y
la revisión del orquestador.

## General spec integration

No se integran todavía hechos en `specs/00..08` ni `context/**`; la
consolidación estable corresponde al orquestador tras su review.
