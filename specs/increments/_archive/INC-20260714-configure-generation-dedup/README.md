# INC-20260714 — `configure.ts` adapter-generation dedup (FIX-E parity)

> **Código**: INC-20260714-configure-generation-dedup
> **Estado**: archived
> **Fecha**: 2026-07-14
> **Batch**: cierre de brechas post-KVP25 (gap 5/5 en el enunciado; orden de ejecución 1→2→3→**5**→4)
> **Groundeo**: catálogo `INC-20260713-e2e-corner-case-catalog` (CC-ADP-04); `INC-20260713-fix-adopt-upgrade` (FIX-E, `resolveAgentsMdTemplateContent`).

## Goal

`sync.ts` resuelve el contenido del template AGENTS.md con **precedencia on-disk → fallback
bundled** (`resolveAgentsMdTemplateContent`, FIX-E) y lo pasa como `templateContent` a
`generateOpencodeConfig`/`generateClaudeCodeConfig`, de modo que un proyecto adoptado cuyo spec
repo carece de `axiom.spec/templates/agents-md-template.md` NO falla con `template-missing`.
`configure.ts` llama a `generateOpencodeConfig` **sin** `templateContent` (`configure.ts` ~L497),
así que reproduce el defecto exacto que FIX-E corrigió en `sync`: dos caminos de generación
divergentes. Brecha **CC-ADP-04**.

## Owner decision (el diseño)

- **Una sola fuente de verdad para el resolver de template.** Mover `resolveAgentsMdTemplateContent`
  (hoy local en `sync.ts`) al módulo compartido `workspace-adapter-templates.ts` (owner
  `@axiom/cli-commands`, ya exporta `AGENTS_MD_TEMPLATE` y ya lo consumen `sync.ts` y
  `workspace-adapters.ts`). Exportarlo desde ahí y reimportarlo en `sync.ts` (sin cambiar su
  comportamiento) y en `configure.ts`.
- **`configure.ts` usa el resolver compartido**: pasar `templateContent:
  resolveAgentsMdTemplateContent(resolution.rootPath)` a su llamada `generateOpencodeConfig`
  (~L497). Con eso `configure` aplica la MISMA precedencia on-disk → bundled que `sync`.
- **Sin regresión.** En el repo self-hosted de Axiom (y cualquier proyecto con el template
  on-disk), `resolveAgentsMdTemplateContent` devuelve el contenido on-disk (byte-idéntico al
  bundled), así que `configure` no cambia su salida. El beneficio es sólo para proyectos
  adoptados sin el template on-disk (dejan de arriesgar `template-missing`).

## Scope

- `apps/cli/src/commands/workspace-adapter-templates.ts` — añadir/exportar
  `resolveAgentsMdTemplateContent(rootPath): string` (on-disk `<root>/axiom.spec/templates/
  agents-md-template.md` si existe y es legible, si no `AGENTS_MD_TEMPLATE`). (Es el mover del
  helper que hoy vive en `sync.ts`.)
- `apps/cli/src/commands/sync.ts` — importar el helper compartido en vez de definirlo local
  (comportamiento idéntico).
- `apps/cli/src/commands/configure.ts` — importar el helper y pasar `templateContent` a
  `generateOpencodeConfig`.
- **Tests** — un test que compruebe que `configure` aplica el fallback bundled cuando el
  template on-disk NO está (no arroja `template-missing`, escribe el output opencode), y que
  cuando el on-disk SÍ está, éste tiene precedencia. Reusar el patrón de los escenarios FIX-E de
  `sync.test.ts` (Scenario 7/7b). No debilitar tests existentes de `configure`.

## Non-goals

- No cambiar el comportamiento de `sync` (sólo se factoriza el helper; salida idéntica).
- No añadir generación claude-code a `configure` si hoy no la hace (sólo alinear la llamada
  opencode existente con el resolver compartido).
- No tocar los generadores de adapters (`@axiom/adapters-*`).

## Acceptance

- `configure` resuelve el template con el MISMO resolvedor que `sync` (on-disk override →
  bundled fallback); un proyecto sin el template on-disk ya no arriesga `template-missing`.
- El helper es una única fuente de verdad compartida por `sync` y `configure`.
- Sin regresión (repo self-hosted byte-idéntico).
- Gate verde en `Axiom/`: `npm run build` → `npm test` → `npm run typecheck` →
  `node apps/cli/dist/index.js doctor` (módulo el flake 5000ms).

## Result

**Implementado 2026-07-14.** Cambios en `Axiom/`:

- `apps/cli/src/commands/workspace-adapter-templates.ts` — se movió
  `resolveAgentsMdTemplateContent(rootPath): string` (antes local en
  `sync.ts`) y su const `AGENTS_MD_TEMPLATE_ONDISK_RELATIVE`, ubicados justo
  después de `AGENTS_MD_TEMPLATE`. Se agregaron los imports `fs`/`path`
  (nuevos en este módulo). Función `export`ada; comportamiento
  byte-idéntico (mismo cuerpo, misma precedencia on-disk → bundled).
- `apps/cli/src/commands/sync.ts` — se eliminó la definición local de
  `resolveAgentsMdTemplateContent`/`AGENTS_MD_TEMPLATE_ONDISK_RELATIVE`; el
  import `{ AGENTS_MD_TEMPLATE } from './workspace-adapter-templates'` se
  reemplazó por `{ resolveAgentsMdTemplateContent } from
  './workspace-adapter-templates'` (`AGENTS_MD_TEMPLATE` ya no se usaba
  directamente en este archivo). Las dos llamadas a
  `resolveAgentsMdTemplateContent(rootPath)` en `materializeAdapterOutputs`
  quedan intactas. Sin cambio de comportamiento.
- `apps/cli/src/commands/configure.ts` — antes: `generateOpencodeConfig({
  projectRoot, projectName, resolvedProfile, skills: [] })` (~L497, SIN
  `templateContent`, defecto CC-ADP-04). Después: se agregó el import
  `resolveAgentsMdTemplateContent` desde `./workspace-adapter-templates` y
  se agregó `templateContent: resolveAgentsMdTemplateContent(resolution.rootPath)`
  a esa misma llamada. También se actualizó el comentario de cabecera del
  módulo (bloque de decisiones de diseño) documentando D-001.
- **Tests** — nuevo archivo `apps/cli/tests/configure-template-resolver-dedup.test.ts`
  (2 tests, aislado de `configure.test.ts`/`sync.test.ts`). No se tocó
  ningún test existente.

**Nota sobre alcance real del fix**: `configure.ts` sólo invoca
`generateOpencodeConfig` cuando `installResult.value.generatedFiles` (del
registry estático `GENERATED_FILES_BY_TARGET` de `@axiom/installer`) NO
cubre `.opencode/AGENTS.md` (branch AD-CONF-1/B1). Con el registry actual
eso NUNCA ocurre para el target `opencode` (siempre lo cubre — confirmado
por `configure.test.ts` Scenario 4, que documenta el generator como
SKIPEADO en el camino normal). Es decir: hoy este branch es código
defensivo para un target futuro que A4 aún no cubra; con el registry
actual es inalcanzable en producción para `opencode`. El fix (D-001) sigue
siendo correcto y necesario: cierra la divergencia de código (misma fuente
de verdad que `sync`) para cuando ese branch SÍ se alcance (registry
futuro, o cualquier llamada directa al mismo código). Para poder ejercitar
el branch real en un test (sin tocar el registry productivo), el nuevo test
mockea `@axiom/installer`'s `installProfile` para que su VALOR DEVUELTO
(no el `install-profile.json` real persistido en disco) omita
`.opencode/AGENTS.md` de `generatedFiles`, forzando `opencodePathsCovered =
false`. Con eso se confirma que la llamada real a
`generateOpencodeConfig` dentro de `configure.ts` ya usa el resolver
compartido: Scenario A (sin template on-disk) escribe `.opencode/AGENTS.md`
+ `skills-lock.yaml` con el contenido BUNDLEADO (sin `template-missing`);
Scenario B (con template on-disk) confirma la precedencia on-disk sobre el
bundled — mismo patrón que `sync.test.ts` Scenario 7/7b.

## Validation

Gate ejecutado en `Axiom/` (2026-07-14):

- `npm run build` (`tsc -b`) — limpio, sin errores.
- `npm test` (`vitest run`) — 279/280 archivos de test, 2811/2812 tests
  passed. El único fallo (`apps/cli/tests/context.test.ts`, `Test timed
  out in 5000ms`) es el flake conocido de I/O pesado bajo carga paralela
  (documentado en el brief de ejecución). Confirmado con
  `npx vitest run apps/cli/tests/context.test.ts --no-file-parallelism`:
  12/12 tests passed en 923ms. No se debilitó ningún test ni se subió el
  timeout configurado.
- `npm run typecheck` (`tsc -b`) — limpio, sin errores.
- `node apps/cli/dist/index.js doctor` — `Resultado: PASS` (45/57 OK · 0
  FALLO · 3 ADVERTENCIA · 9 OMITIDO; las 3 advertencias y 9 omitidos son
  preexistentes, no relacionados con este cambio).

## Acceptance review

- `configure` resuelve el template con el MISMO resolver que `sync`
  (on-disk override → bundled fallback): **cumplido** — ambos importan
  `resolveAgentsMdTemplateContent` desde `workspace-adapter-templates.ts`.
- El helper es una única fuente de verdad compartida: **cumplido** — vive
  una sola vez, en `workspace-adapter-templates.ts`, exportado.
- Sin regresión (repo self-hosted byte-idéntico): **cumplido** — el propio
  repo `Axiom/` tiene su template on-disk, así que `resolveAgentsMdTemplateContent`
  sigue devolviendo el contenido on-disk sin cambios; `doctor` PASS
  confirma que `AGENTS.md` del repo sigue siendo válido.
- Gate verde en `Axiom/`: **cumplido** (ver `## Validation`).

## General spec integration

No se requirió integración en `Axiom.Spec/general-spec.md`: este
incremento es una corrección de dedup interna a `Axiom/` (mover un helper
existente a su módulo compartido y reimportarlo), sin introducir
conocimiento de producto ni comportamiento nuevo. La única fuente de
verdad relevante (precedencia on-disk → bundled del template AGENTS.md,
FIX-E) ya estaba documentada en el increment
`INC-20260713-fix-adopt-upgrade`; este incremento sólo extiende su
aplicación a `configure.ts`.

**Status: closed** — goal claro, acceptance criteria cumplidos, cambios
implementados, gate ejecutado y verde (con el flake conocido confirmado y
aislado), review de acceptance completado, sin integración pendiente a
`general-spec.md`.
