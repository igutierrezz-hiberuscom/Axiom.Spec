# INC-20260730-autopilot-integration: Autopilot Integration

## Metadata

- **ID**: INC-20260730-autopilot-integration
- **Status**: closed
- **Goal**: Integrar las piezas de gobierno verificable (candidate freeze y phase receipts) en el orquestador `axiom-autopilot`, para que las directivas de seguridad se apliquen también en los flujos desatendidos de la propia herramienta, y no solo en el ciclo manual de un incremento individual.
- **Scope**: Las siete superficies distributables de `axiom-autopilot` (skills, agent y prompt) que definen su playbook:
  - `.agents/skills/axiom-autopilot/SKILL.md` (raíz de workspace)
  - `Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md`
  - `Axiom.SDD/.github/skills/axiom-autopilot/SKILL.md`
  - `Axiom.SDD/.github/agents/axiom-autopilot.agent.md`
  - `Axiom.SDD/.github/prompts/axiom-autopilot.prompt.md`
  - `.claude/skills/axiom-autopilot.md`
  - `.claude/commands/axiom-autopilot.md`
- **Non-goals**: Reescribir la lógica base de descomposición del orquestador; modificar el comportamiento de los subagentes internos (`axiom-increment`, `axiom-review`); implementar los comandos `axiom freeze` / `axiom phase receipt` en sí (son objeto de los incrementos hermanos `INC-20260730-candidate-freeze` e `INC-20260730-phase-receipts`, que tocan `Axiom/` y están fuera del alcance de este incremento); tocar cualquier archivo bajo `Axiom/`.

## Acceptance Criteria

- [x] El skill de `axiom-autopilot` incluye instrucciones explícitas de:
  - [x] delegar a subagentes usando un scope tipado y determinista (nunca descripciones vagas);
  - [x] verificar el freeze del candidate (`axiom freeze --increment <id>`) antes de enviarlo a un subagente para `apply`;
  - [x] capturar y verificar los recibos de fase (`axiom phase receipt`) antes de integrar el conocimiento del incremento.
- [x] El ciclo de validación de candidate freeze y recibos es un requerimiento **formal** (no opcional) del workflow de autopilot, expresado con lenguaje imperativo consistente con la sección de guardrails de cada archivo cuando esta existe.
- [x] Las siete superficies de `axiom-autopilot` (no solo una) llevan las tres directivas, para que se propaguen también al instalar `Axiom.SDD` en proyectos consumidores.

## Implementation Plan

### Fase 1: Auditoría de superficies existentes

1. Localizar las siete superficies que definen `axiom-autopilot` (skills, agent, prompt, command) en el workspace y en `Axiom.SDD`.
2. Confirmar con `grep` cuáles ya mencionan `axiom freeze` / `axiom phase receipt` y cuáles no.
3. Leer en detalle la única superficie ya conforme (`.agents/skills/axiom-autopilot/SKILL.md`, en la raíz del workspace) para usarla como referencia de ubicación (paso 4 = freeze antes de delegar `apply`; paso 6 = receipts antes de dar por bueno un incremento).

### Fase 2: Propagación adaptada a cada formato

4. Para los dos variantes largas y ricas en prosa (`Axiom.SDD/.github/skills/axiom-autopilot/SKILL.md` en español, `.claude/skills/axiom-autopilot.md` en inglés), integrar las tres directivas dentro de la prosa existente de los pasos 4 y 6 del playbook, y añadir un guardrail explícito que las declare formales.
5. Para las superficies cortas (`Axiom.SDD/.github/agents/axiom-autopilot.agent.md`, `Axiom.SDD/.github/prompts/axiom-autopilot.prompt.md`, `.claude/commands/axiom-autopilot.md`), añadir una versión concisa que nombre literalmente `axiom freeze --increment <id>` y `axiom phase receipt` sin expandir la estructura del archivo.
6. Para el espejo corto ya existente (`Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md`), replicar exactamente el patrón de la referencia (bullets bajo los pasos 4 y 6), traducido a inglés para mantener la coherencia de idioma del archivo.
7. Mantener intacto el idioma original de cada archivo (español/inglés) y no tocar la superficie ya conforme ni ningún archivo bajo `Axiom/`.

### Fase 3: Verificación

8. Releer cada archivo editado completo para confirmar coherencia de redacción y formato.
9. Ejecutar `grep` de `axiom freeze` y `axiom phase receipt` sobre las siete superficies y confirmar que las siete aparecen en ambas búsquedas.

## Decisiones de implementación

- **Redacción exacta de las tres directivas** (adaptada por archivo, mismo contenido semántico en las siete superficies):
  1. Scope tipado y determinista: la delegación a `axiom-increment` debe usar un scope tipado y determinista (archivos/paquetes/repositorios exactos), nunca una descripción vaga que deje el alcance a criterio del subagente.
  2. Freeze previo: antes de delegar el `apply` de un incremento a un subagente, el orquestador debe verificar `axiom freeze --increment <id>` y confirmar que el snapshot es válido y vigente; no debe delegar el `apply` de un candidate sin freeze o con freeze desactualizado.
  3. Receipts antes de integrar: al verificar cada incremento devuelto por un subagente, el orquestador debe capturar y verificar sus recibos criptográficos (`axiom phase receipt`) como certificación de gobierno verificable, y no debe dar el incremento por verificado ni integrar su conocimiento (paso 7 del playbook) si los recibos faltan o no validan.
- **Ubicación por archivo**: en los dos formatos largos (`.github/skills/.../SKILL.md`, `.claude/skills/axiom-autopilot.md`) las directivas se integraron en la prosa existente de los pasos "Delegar cada incremento" / "Spawn a self-contained subagent" (freeze) y "Verificar cada incremento" / "Independently verify" (receipts), y además se añadió un guardrail explícito elevándolas a requisito formal. En los formatos cortos (`agent.md`, `prompt.md`, `command.md`, espejo `.agents` corto) se insertaron como una o dos frases concisas dentro de las reglas o pasos ya existentes, sin ampliar la estructura del archivo.
- **Idioma preservado por archivo**: los archivos en español (`.github/skills/.../SKILL.md`, `.github/agents/axiom-autopilot.agent.md`, `.github/prompts/axiom-autopilot.prompt.md`) recibieron las directivas en español; los archivos en inglés (`.agents/skills/axiom-autopilot/SKILL.md` de `Axiom.SDD`, `.claude/skills/axiom-autopilot.md`, `.claude/commands/axiom-autopilot.md`) las recibieron en inglés. No se tradujo el archivo de referencia (que mezcla bullets en español dentro de un skill en inglés); ese archivo queda fuera del scope de este incremento y no fue tocado.
- **Comandos usados literalmente**: se usó `axiom freeze --increment <id>` y `axiom phase receipt`, tal como los define el propio scope de este incremento y los incrementos hermanos `INC-20260730-candidate-freeze` (`axiom freeze --increment <id>` genera `.frozen.json`) e `INC-20260730-phase-receipts` (receipts JSON en `specs/increments/<id>/receipts/`). Esos dos incrementos, y el comando `axiom freeze` en sí (`Axiom/apps/cli/src/commands/freeze.ts`), pertenecen a otro incremento concurrente y no se tocaron.
- **Archivo de referencia no modificado**: `.agents/skills/axiom-autopilot/SKILL.md` (raíz del workspace) ya cumplía las tres directivas antes de este incremento; se dejó intacto y se usó únicamente como plantilla de ubicación para el resto de superficies, tal como indicaba el encargo.
- **Formalidad del requisito**: en los dos archivos que ya tenían una sección `## Guardrails` explícita, se añadió un guardrail dedicado que declara ambos checks como "formal, mandatory gates" / "requisitos formales del ciclo; nunca los omitas". En los archivos sin sección de guardrails propia, la formalidad se expresó con lenguaje imperativo en línea ("es un requisito formal, no opcional", "this is a formal gate, not optional").

## Validación y review

No existe un comando de validación para contenido Markdown en este repositorio (no hay build/test/lint aplicable a `.md`). Se declara explícitamente:

`No validation command was found. Performed best-effort validation by inspecting changed files and checking consistency against the requested behavior.`

Validación best-effort realizada:

- `grep "axiom freeze"` sobre el workspace: aparece en las siete superficies de `axiom-autopilot` (conteos: `.agents/skills/axiom-autopilot/SKILL.md`:1, `Axiom.SDD/.agents/skills/axiom-autopilot/SKILL.md`:1, `Axiom.SDD/.github/skills/axiom-autopilot/SKILL.md`:2, `Axiom.SDD/.github/agents/axiom-autopilot.agent.md`:1, `Axiom.SDD/.github/prompts/axiom-autopilot.prompt.md`:1, `.claude/skills/axiom-autopilot.md`:2, `.claude/commands/axiom-autopilot.md`:2), más 3 apariciones fuera de alcance en `Axiom.Spec/specs/increments/INC-20260730-candidate-freeze/README.md`, 1 en este propio README y 2 en `Axiom/apps/cli/src/commands/freeze.ts` (código de otro incremento, no tocado).
- `grep "axiom phase receipt"` sobre el workspace: aparece en las mismas siete superficies (conteos: `.agents/skills/...`:1, `Axiom.SDD/.agents/skills/...`:1, `Axiom.SDD/.github/skills/...`:2, `Axiom.SDD/.github/agents/...`:1, `Axiom.SDD/.github/prompts/...`:1, `.claude/skills/axiom-autopilot.md`:2, `.claude/commands/axiom-autopilot.md`:2), más 1 aparición en este propio README.
- Lectura completa de cada uno de los seis archivos editados tras el cambio, confirmando que la redacción encaja con el resto del documento, que no se rompió ningún encabezado/lista Markdown, y que ningún archivo bajo `Axiom/` fue tocado.
- Confirmado que las tres directivas cubren exactamente los tres puntos del `Scope` original: scope tipado, verificación de freeze antes de `apply`, y captura/verificación de receipts antes de integrar conocimiento.

## General spec integration

**Realizada** en la pasada única de integración a nivel de lote (2026-08-02), junto con los otros cinco incrementos `INC-20260730-*`. Se tocaron los nueve ficheros canónicos:

- `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
- `Axiom.Spec/specs/01_Requisitos_Funcionales.md`
- `Axiom.Spec/specs/02_Requisitos_No_Funcionales.md`
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
- `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`
- `Axiom.Spec/specs/05_Interfaces_Operativas.md`
- `Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`
- `Axiom.Spec/specs/07_Gobierno_y_Seguridad.md`
- `Axiom.Spec/specs/08_Glosario.md`

Lo aportado por ESTE incremento quedó en: Propagación normativa a las 7 superficies y la regla "una directiva solo en la copia local no está adoptada" (`07`), resumen de tanda (`00`).

### Contexto técnico (`Axiom.Spec/context/**`)

**Sí aplicó.** Documentos actualizados por este incremento: `references/03-riesgos-y-brechas-conocidas.md` (Brecha 6: 7 copias sin generador único, nada lo verifica), `TECHNICAL_CONTEXT.md` (punto 6), `references/02-historial-de-incrementos.md`.

El pase de contexto no fue solo aditivo: se corrigió el punto 5 de `context/TECHNICAL_CONTEXT.md`, que declaraba `TC-011` como bloqueo vigente y citaba "3425/3427 tests" — `npm run doctor` da hoy `PASS` (46/61 OK, 0 fallos) y la suite vigente es 3489 tests / 3483 verdes / 6 rojos preexistentes.

## Estado de cierre

El incremento se marca `closed`: el objetivo es claro, los criterios de aceptación existen y se cumplieron, los cambios se implementaron en las seis superficies pendientes (la séptima ya cumplía y no requería cambios), se ejecutó la validación disponible (best-effort, sin comando de test para Markdown, más verificación cruzada por `grep`) y el resultado se revisó contra cada criterio de aceptación. La integración en `general-spec`/`specs/00..08`/`context/**` queda explícitamente diferida al paso de consolidación de lote del propio orquestador `axiom-autopilot`, tal como exige el encargo recibido; no bloquea el cierre porque es un diferimiento intencional del proceso batch, no un vacío de trabajo.
