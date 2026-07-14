# INC-20260714 — Toolchain install honesty (close the false-green)

> **Código**: INC-20260714-toolchain-install-honesty
> **Estado**: archived
> **Fecha**: 2026-07-14
> **Batch**: cierre de brechas post-KVP25 (gap 4/5; orden de ejecución 1→2→3→5→**4**, se hace la última)
> **Groundeo**: catálogo `INC-20260713-e2e-corner-case-catalog` (CC-ISO-04, Área J); `INC-20260710-honesty-and-toolchain-states` (modelo de estados marker/installed-working ya existente); memoria `project-axiom-kvp25-integration-test`.

## Goal

`axiom toolchain add` escribe YAML y `toolchain repair` crea un **directorio-marcador vacío**;
no descarga/configura/registra nada. El modelo de honestidad ya distingue estados
(`absent`/`marker`/`declared`/`installed-working`, `detectAllToolsWithProbe`) y emite warnings
(`required-tool-not-verified`, `optional-tool-unverified`, `optional-tool-absent`). PERO queda un
**"verde falso"**: un tool DECLARADO-pero-ausente o sólo-marker con `mvp:false` produce sólo un
warning y `toolchain validate` sale 0 / lo presenta como satisfecho. El operador puede creer que
un tool está instalado cuando sólo hay un marker. Brecha **CC-ISO-04**.

## Owner decision (auto-resuelta): **opción (b) — honestidad, no instaladores especulativos**

Se elige la **opción (b)** del enunciado (renombrar/clarificar `install`→`declare`/`scaffold` en
la UX y hacer que `validate` NO reporte verde para un tool declarado-pero-ausente aunque sea
`mvp:false`), NO la opción (a) (instaladores reales por proveedor). Rationale:

- **Alineado con los límites de bootstrap** (`Axiom.SDD/AGENTS.md`): "no introducir scripts que
  aún no existen" ni arquitectura especulativa. Ejecutar `uv tool install serena-agent` de verdad
  (opción a) añade side-effects de red + superficie difícil de testear de forma determinista, y
  bordea las integraciones pesadas que el bootstrap desaconseja.
- **Cierra directamente el "verde falso"** (el mínimo exigido) y hace que `validate` refleje la
  realidad (marker vs installed-working), que es la acceptance.
- La opción (a) (instaladores reales por proveedor + registro de su MCP) queda documentada como
  **consideración futura** (no se implementa ahora).

### Qué cambia concretamente

1. **`validate` probe-backed y honesto**: `toolchain validate` (y el check de toolchain del
   `doctor`) SIEMPRE corren el probe real (`detectAllToolsWithProbe`) y clasifican cada tool
   declarado por su estado REAL. Un tool declarado cuyo estado real NO es `installed-working`
   (`marker`/`declared`/`absent`) **NUNCA** se presenta como "OK/verde" — se reporta en una
   sección/estado honesto distinto ("declarado, no verificado" / "sólo marker" / "ausente"),
   **independientemente de `mvp`**. El resumen cuenta esos tools como no-satisfechos.
2. **Semántica de exit-code preservada donde importa**: un tool `mvp:false` no-installed-working
   sigue SIN ser un error bloqueante (exit no cambia a 1 por su culpa — eso queda reservado para
   required/mvp), pero **deja de contarse como verde**: el output humano y el resumen lo muestran
   como no-instalado. (Es decir: se cierra el verde-falso en la PRESENTACIÓN/estado, sin convertir
   cada tool opcional ausente en un fallo duro que rompería `doctor`.)
3. **UX honesta de `add`/`repair`**: aclarar en la ayuda/mensajes del comando `toolchain` que
   `add` DECLARA (escribe YAML) y `repair` hace SCAFFOLD de un marker (NO instala nada); sólo un
   probe `installed-working` cuenta como instalado. Renombrado/aliasing mínimo y aditivo si aplica
   (sin romper los subcomandos existentes).

## Scope

- `packages/toolchain/src/{validate,probe,detect,repair,types}.ts` — asegurar que el estado real
  (probe) manda; que `validateToolchain` clasifique honesto marker/declared/absent vs
  installed-working para TODO tool declarado (no sólo required); ajustar mensajes/tipos si hace
  falta (aditivo). NO debilitar el modelo existente; extenderlo para cerrar el verde-falso.
- `apps/cli/src/commands/toolchain.ts` — `runToolchainValidate`: correr el probe real por defecto
  y renderizar los estados honestos (nunca "OK" para un tool no `installed-working`); resumen que
  cuente los no-satisfechos; ayuda/mensajes de `add`/`repair` clarificando declare/scaffold.
- `axiom.config/toolchain-catalog.yaml` — sólo si hace falta alinear metadata (p. ej. marcar
  claramente qué tools son declare-only/marker); cambios mínimos.
- **Tests** — un tool DECLARADO con `mvp:false` cuyo estado real es `marker` (o `absent`):
  `validate` lo reporta como NO verificado / no instalado (NO verde), con el probe real (o un
  `probeFn` stub que devuelve marker). Un tool `installed-working` (probe stub) sí cuenta como OK.
  Regresión: required/mvp tools mantienen su semántica de error. NO debilitar los tests de honestidad
  existentes (`toolchain.test.ts` Scenario 2, doctor toolchain checks); extenderlos.

## Non-goals

- **NO** implementar instaladores reales por proveedor (opción a) — documentado como futuro.
- **NO** convertir cada tool opcional ausente en un error bloqueante (rompería `doctor` PASS);
  el cierre del verde-falso es de PRESENTACIÓN/estado, preservando exit-code semantics de
  required/mvp.
- **NO** registrar MCPs de tools no instalados (no hay instalación real que registrar).

## Acceptance

- `toolchain validate` refleja la realidad: un tool declarado-pero-ausente o sólo-marker (incluso
  `mvp:false`) NO aparece como verde/OK; se reporta con su estado honesto (marker/declared/absent
  vs installed-working). Un tool `installed-working` sí aparece OK.
- La UX de `add`/`repair` deja claro que declaran/scaffoldean (no instalan).
- required/mvp mantienen su semántica de error (sin regresión); `doctor` sigue PASS.
- Gate verde en `Axiom/`: `npm run build` → `npm test` → `npm run typecheck` →
  `node apps/cli/dist/index.js doctor` (módulo el flake 5000ms).

## Result

**Estado: closed.**

### Verificación del gap previo (antes de tocar código)

El modelo de honestidad (`ToolState`, `detectAllToolsWithProbe`, warnings
`required-tool-not-verified`/`optional-tool-unverified`/`optional-tool-absent`)
YA estaba implementado (INC-20260710) y `runToolchainValidate` YA corría el
probe real por defecto. El residuo confirmado era puramente de
**presentación**: `formatToolchainValidate` (`apps/cli/src/commands/toolchain.ts`)
imprimía el encabezado `✓ Toolchain válido con N warning(s)` (checkmark verde +
la palabra "válido") incluso cuando esos warnings eran exactamente
`required-tool-not-verified` / `optional-tool-unverified` / `optional-tool-absent`
— es decir, tools declaradas cuyo estado real NUNCA fue confirmado como
`installed-working`. Confirmado con el test "Scenario 2" existente
(`apps/cli/tests/toolchain.test.ts`), que afirmaba literalmente
`/✓ Toolchain válido con 2 warning\(s\)/` para dos tools required marcadas
sólo como `marker`. Adicionalmente se verificó que **todas** las entries del
catálogo real (`axiom.config/toolchain-catalog.yaml`) son `mvp: false`, así
que el caso "declarado-pero-no-instalado con mvp:false" no era un edge case
sino el camino común para cualquier tool agregada vía `axiom toolchain add`.

### Qué cambió

- `apps/cli/src/commands/toolchain.ts`:
  - `formatToolchainValidate` ahora recibe `manifest` + `detections` (además
    del `ToolchainValidationResult`) y:
    - Nunca imprime `✓ Toolchain válido` cuando alguna tool declarada
      (independientemente de `mvp`) no está confirmada como
      `installed-working`. En su lugar imprime
      `⚠ Toolchain sin errores bloqueantes, pero N/M tool(s) declarada(s) NO
      están confirmadas como instaladas (installed-working):`.
    - Agrega un bloque nuevo "Estado real de las tools declaradas (X/Y
      confirmada(s) como instalada(s))" que clasifica CADA tool del manifest
      por su estado real, vía el nuevo helper `describeHonestState` (nunca
      dice "instalada" salvo `installed-working`; `marker`/`declared`/`absent`
      se etiquetan explícitamente como "NO instalada — …").
    - `exitCode`/`ok` NO cambiaron (D-002): el gate sigue reservado a
      required/mvp vía `required-tool-missing`.
  - UX de `add`/`repair` (D-003): el mensaje de éxito de `add`, la
    descripción del comando `toolchain` y las descripciones de `add`/`repair`
    ahora aclaran explícitamente que `add` DECLARA (escribe YAML) y `repair`
    SCAFFOLDEA un marker vacío — ninguno instala un binario real.
- `packages/toolchain/src/repair.ts`: mensajes de `repairTool` (instruction-only,
  marker-ya-presente, scaffold-creado) ampliados con la misma aclaración
  declare/scaffold-vs-install, preservando los substrings que ya
  testeaban `repair-add-gitignore.test.ts` (`instruction-only`, `ya está
  presente`).
- **NO se tocó** `packages/toolchain/src/{validate,detect,probe,types}.ts` ni
  `packages/doctor/src/checks.ts`: el modelo de detección/validación y el
  check TC-004/TC-005 de `doctor` (que no pasa `detections`, por diseño
  back-compat) quedan intactos — el cierre es 100% de presentación en el CLI,
  tal como especifica D-002 (`doctor` sigue PASS, sin nuevos errores duros
  para tools opcionales ausentes).
- **NO se tocó** `axiom.config/toolchain-catalog.yaml` (no hizo falta alinear
  metadata).

### Antes / después (evidencia)

Antes (comportamiento previo, tool `mvp:false` sólo-marker):
```
✓ Toolchain válido con 1 warning(s):
  ⚠ [optional-tool-unverified] Tool opcional "codegraph" está declarada (estado: marker) ...
```

Después (smoke manual contra un proyecto temporal, ver Validation):
```
⚠ Toolchain sin errores bloqueantes, pero 1/1 tool(s) declarada(s) NO están confirmadas como instaladas (installed-working):
  ⚠ [optional-tool-unverified] Tool opcional "codegraph" está declarada (estado: marker) ...

Estado real de las tools declaradas (0/1 confirmada(s) como instalada(s)):
  ⚠ codegraph                            NO instalada — sólo marker (scaffold vacío de `repair`, sin probe confirmado)
```
`exitCode` se mantuvo en `0` en ambos casos (mvp:false, sin regresión de
exit-code semantics).

### Tests

- `apps/cli/tests/toolchain.test.ts`:
  - **Scenario 2** (reescrito, no debilitado): antes afirmaba el "verde falso"
    (`/✓ Toolchain válido con 2 warning\(s\)/`) para dos tools required
    marker-only; ahora afirma explícitamente que ESE texto YA NO aparece
    (`expect(text).not.toMatch(/✓ Toolchain válido/)`) y que en su lugar
    aparece el encabezado honesto `⚠ Toolchain sin errores bloqueantes, pero
    2/2 tool(s) …` + el desglose `NO instalada — sólo marker`. Se documenta
    inline por qué se reescribió (cierra el verde falso residual).
  - **Scenario 2b** (nuevo): tool `mvp:false` marker-only y, en un segundo
    `it`, `mvp:false` absent — ambos casos: `exitCode: 0` (sin regresión de
    exit-code), `validation.ok: true`, pero el texto humano NUNCA dice
    "✓ Toolchain válido" y el desglose marca la tool como "NO instalada".
  - **Scenario 2c** (nuevo): `probeFn` que confirma la tool (`installed-working`)
    → cero warnings, encabezado `✓ Toolchain válido.` y la tool aparece con
    `✓ ... instalada y verificada (installed-working)` en el desglose.
  - **Scenario 2d** (nuevo, regresión): required/mvp tool ausente (sin
    `probeFn` override, usando el probe real por defecto — sin binarios que
    probar porque el manifest está vacío, cero riesgo de timeout) →
    `exitCode: 1` + `required-tool-missing`, sin cambios respecto al
    comportamiento previo.
- No se debilitó ningún test existente: `packages/toolchain/tests/toolchain.test.ts`
  (incl. el escenario 8b `optional-tool-absent`), `packages/toolchain/tests/p1-tools.test.ts`,
  `packages/toolchain/tests/repair-add-gitignore.test.ts`,
  `packages/doctor/tests/toolchain.test.ts` y
  `apps/cli/tests/toolchain-catalog-real.test.ts` quedaron intactos y siguen
  en verde (la única reescritura fue el Scenario 2, autorizada explícitamente
  porque afirmaba el verde falso que este incremento cierra).

### Gate (repo `Axiom/`)

- `npm run build` → OK (tsc -b, sin errores).
- `npm test` → **280 test files passed (280), 2816 tests passed (2816)**, 0
  fallos, sin timeouts ni flakes (no hizo falta re-run aislado de
  `engram-backend.test.ts` — pasó limpio en la corrida completa: 15/15 en
  14.7s).
- `npm run typecheck` → OK (tsc -b, sin errores).
- `node apps/cli/dist/index.js doctor` → `Resultado: PASS` (`Resumen: 45/57 OK
  · 0 FALLO · 3 ADVERTENCIA · 9 OMITIDO`); TC-004/TC-005/TC-006 (toolchain)
  todos `✓`, sin regresión.
- Smoke manual (`axiom toolchain add|repair|validate` contra un proyecto
  temporal en el scratchpad, fuera del repo) confirmó el antes/después de
  arriba y que `add`/`repair` imprimen la aclaración declare/scaffold.

### Revisión contra acceptance criteria

- "`toolchain validate` refleja la realidad... NO aparece como verde/OK...
  independientemente de `mvp`" → **cumplido**: ver antes/después y Scenario 2b.
- "Un tool `installed-working` sí aparece OK" → **cumplido**: Scenario 2c.
- "UX de `add`/`repair` deja claro que declaran/scaffoldean" → **cumplido**:
  mensajes + descripciones de comando actualizados (D-003).
- "required/mvp mantienen su semántica de error... `doctor` sigue PASS" →
  **cumplido**: Scenario 2d + gate de doctor sin cambios.
- "Gate verde en `Axiom/`" → **cumplido**, números arriba.

### Consideración futura documentada (opción a, NO implementada)

Instaladores reales por proveedor (p. ej. `uv tool install serena-agent`,
`npm install -g codegraph`, etc.) que ejecuten la instalación de verdad y
registren su MCP quedan fuera de este incremento (bootstrap limits: no
scripts especulativos, no side-effects de red no determinísticos en tests).
Si se decide abordarlos a futuro, el punto de enganche natural es
`packages/toolchain/src/repair.ts` (agregar un modo `--install` opcional que
invoque un instalador real por tool, gated detrás de una confirmación
explícita del operador) y `probe.ts` (ya provee el contrato de verificación
post-instalación vía `probeToolInstalled`).

## General spec integration

No se requiere ninguna integración nueva en `Axiom.Spec/general-spec.md`: el
modelo de honestidad del toolchain (estados `absent/marker/declared/installed-
working`, semántica de exit-code reservada a required/mvp) ya estaba
documentado como conocimiento estable desde INC-20260710. Este incremento
sólo cierra un residuo de presentación sobre ese mismo modelo — no introduce
un concepto nuevo que amerite consolidarse por separado.
