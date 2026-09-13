# r15 docs truth baseline

> **Código**: INC-20260912-r15-docs-truth-baseline
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-12
> **Tipo de cambio**: corrección de veracidad documental
> **Acciones de origen**: `ACC-091`, `ACC-092`, `ACC-093`
> **Plan**: `PLAN-INC-20260912-r15-documentation-single-source` (posición 1 de 6)

## Resumen

Dejar de afirmar cosas falsas en la documentación del runtime y fijar la estructura del manual único antes de escribir contenido nuevo. Cubre tres frentes: el `README.md` del runtime, los manuales que describen ficheros de configuración inexistentes y los índices que siguen anunciando superficies retiradas. Además registra como artefacto de decisión la estructura del manual que los incrementos siguientes deben respetar.

No añade cobertura de comandos: eso es el incremento 2. Aquí solo se corrige y se declara la forma.

## Contexto y motivación

La auditoría R-15 (sesión 2026-09-12) verificó que ninguna prueba, chequeo de doctor ni script valida la documentación del runtime, y que por eso conviven afirmaciones superadas con las vigentes:

- `Axiom/README.md` enlaza a `../openspec/changes/archive/2026-06-29-0019-plan/0019-archive-summary.md`, y no existe `openspec/` ni en el repositorio ni en el workspace. Hay tres menciones más en prosa (`docs/README.md`, `docs/cli/components.md`, `packages/installer/README.md`).
- El mismo README presenta los adapters como operativos con `dist/` materializado, cuando `ACC-084` (R-14) desversionó esos artefactos y ajustó `TC-009` para avisar ante ausencia de build; y enumera ocho comandos y «29+ sub-comandos» frente a las 47 familias que registra `apps/cli/src/index.ts`.
- Cuatro manuales de `docs/configuration/files/` describen ficheros que no existen (`onboarding.yaml`, `scaffolding-contract.yaml`, `command-protocol.yaml`, `local-overlay-policy.yaml`). Cada uno lo declara en su primera línea, pero el índice los presenta bajo el mismo epígrafe que los reales.
- `docs/README.md` no menciona `docs/cli/tui.md`, que existe y está correctamente marcado como histórico; `docs/configuration/profiles-overlays-targets.md` conserva en el nombre un eje retirado en R-04 aunque su contenido ya está actualizado; y `Axiom.Spec/specs/README.md` no lista `decisions/`, `adr/`, `plans/` ni `archive/`, que sí existen.

Escribir manuales nuevos sobre esta base multiplicaría el problema: el lector no puede distinguir qué es vigente y qué es historia, y un agente que lea estos textos aprenderá contratos que ya no existen.

## Alcance

### Incluido

- `ACC-091`: corregir `Axiom/README.md`. Separar de forma inequívoca la foto histórica del estado vigente, eliminar el enlace muerto y las tres menciones en prosa a `openspec/`, corregir la afirmación sobre `dist/` de adapters y alinear el inventario de comandos y paquetes con el runtime real.
- `ACC-092`: resolver los cuatro manuales de ficheros inexistentes, retirándolos o marcándolos como contrato histórico no materializado también en `docs/configuration/files/README.md`, y documentar los cinco YAML vigentes sin manual (`agents-catalog`, `skills-catalog`, `toolchain-catalog`, `mcp-manifest`, `workflows`).
- `ACC-093`: corregir los índices. `docs/README.md` debe reflejar lo que existe, incluido lo marcado como histórico; renombrar `docs/configuration/profiles-overlays-targets.md` al vocabulario vigente actualizando sus referencias; y corregir el índice estructural `Axiom.Spec/specs/README.md`.
- Registrar `D-01` (estructura del manual único) como artefacto de decisión mediante `axiom axiom-decision create`, no como prosa dentro de este README.

### Excluido

- Escribir la documentación de las familias de comandos no cubiertas: es el incremento 2.
- Tocar el contenido de `Axiom.Spec/specs/manuales/**`. Decisión del usuario del 2026-09-12: ese conjunto es material interno de esta instalación, no se fusiona ni se conserva como fuente y expira con `Axiom.Spec`. En este incremento solo se corrige el índice estructural `specs/README.md`, no los manuales internos.
- Crear el gate de verificación documental: es el incremento 5.
- Cambiar comportamiento de producto. Este incremento no toca código de runtime salvo comentarios o rutas de documentación referenciadas desde el código.

## Documentos del incremento

- `01_Requisitos.md`: qué debe cumplir cada corrección.
- `02_Cambios_Modelo.md`: qué archivos se corrigen, renombran o retiran.
- `03_Criterios_Aceptacion.md`: criterios verificables por acción.
- `04_Interacciones_UI.md`: efecto en la navegación de la documentación.

## Dudas abiertas

- `D-01` se cierra dentro de este incremento: qué estructura tiene el manual único, qué se considera «una familia documentada», cómo se nombran las páginas y dónde viven las históricas. Sin esa decisión, el incremento 2 elegiría formato en caliente y el 6 distribuiría algo inestable.
- Si los cuatro manuales de ficheros inexistentes se retiran o se conservan como histórico: la decisión debe quedar registrada en el propio incremento con su motivo, y ser la misma para los cuatro salvo justificación explícita por archivo.

## Decisiones funcionales cerradas

- El destino del manual único es `Axiom/docs/**`, dentro del repositorio del runtime. No se reabre la frontera de ADR-0032.
- La documentación histórica no se borra por defecto: se marca con un encabezado explícito de histórico, como ya hace `docs/cli/tui.md` y como hacen `specs/00..08`.

## Consolidación en la spec general

Al cerrarse, este incremento no aporta conocimiento de producto nuevo: corrige documentación. La única integración esperada es la decisión `D-01` como artefacto y, si procede, una línea en el contexto técnico que declare `Axiom/docs/**` como manual único de producto. Los archivos propietarios se determinan al integrar; el índice lo regenera Core.

## Estrategia E2E

- Barrido de enlaces relativos en `Axiom/README.md` y `Axiom/docs/**`: ningún destino inexistente.
- Barrido de vocabulario retirado (`overlay` como eje, `gateway`, `enterprise`, `product-owner`, `generated-snapshots`) en documentación activa: cero coincidencias fuera de secciones marcadas como históricas.
- Comprobación de que cada manual de `docs/configuration/files/` corresponde a un fichero existente en `Axiom/axiom.config/` o está marcado como histórico.
- `npm run build`, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` en verde.

## Trazabilidad y fuentes

- Acciones `ACC-091`, `ACC-092`, `ACC-093` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia verificada: `Axiom/README.md`, `Axiom/docs/README.md`, `Axiom/docs/configuration/files/{README,onboarding,scaffolding-contract,command-protocol,local-overlay-policy}.md`, `Axiom/docs/configuration/profiles-overlays-targets.md`, `Axiom/axiom.config/` (12 YAML), `Axiom/apps/cli/src/index.ts` (47 familias registradas), `Axiom.Spec/specs/README.md`.

## Notas de implementación

- D-01 fue creado por Axiom como `DEC-20260912-233336-b2pyyy` y enlazado a
	este incremento y al plan `PLAN-INC-20260912-r15-documentation-single-source`.
- Los cuatro manuales de contratos no materializados se conservan con marca
	histórica homogénea; los cinco YAML reales reciben manual propio.
- La página renombrada usa el nombre vigente `configuration-policy-targets.md`.

## Integración canónica pendiente

La consolidación de `specs/00..08` y `context/**`, así como la transición
`verify`/archive, queda pendiente para el orquestador. El hecho estable a
integrar es que `Axiom/docs/**` es el manual único del runtime y que su
contenido histórico debe estar marcado explícitamente.

## Revisión independiente

La review independiente inicial encontró dos blockers documentales: el claim
de `dist/` materializado en `packages/adapters/README.md` y omisiones en
`docs/README.md`. Ambos se corrigieron y la re-review N=1 los marcó como
`verified`, junto con D-01; el ledger queda en `review-ledger.md`.

## Estado de validación humana

Validado para archive por review independiente y pendiente únicamente de la
transición lifecycle gobernada por el orquestador. Validaciones ejecutadas:

- Barrido de enlaces relativos de `Axiom/README.md` y `Axiom/docs/**`: PASS.
- Barrido de vocabulario retirado y `openspec/` en documentación activa: PASS.
- Correspondencia manual-fichero y marcas históricas: PASS.
- `npm run build`: PASS.
- `npm run doctor`: PASS, 0 fallos; 2 advertencias y 11 checks omitidos por
	condiciones preexistentes del entorno.
- `npm run readiness:first-project`: PASS.
- Suites dirigidas de agents, skills y checks de configuración: PASS, 148
	tests.
- `git diff --check`: sin errores; solo emitió avisos CRLF sobre archivos
	preexistentes del worktree.
- Receipt de fase `verify`: íntegro, hash
  `37447a2b0e42922e6af32ccadae40ef36f03559a31bc92cd06bf1ffdd42018fd`.
- Receipts del incremento: 7/7 hashes SHA-256 verificados.

El contenido narrativo de D-01 fue completado en el README del artefacto
creado por Core. La metadata y los enlaces siguen gobernados por Axiom; no se
editaron `metadata.yml`, receipts ni `candidate-freeze.json` manualmente.
