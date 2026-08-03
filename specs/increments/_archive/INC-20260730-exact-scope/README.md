# INC-20260730-exact-scope: Type Check Recovery de Entradas

## Metadata

- **ID**: INC-20260730-exact-scope
- **Status**: closed
- **Goal**: Limitar el surface area de los parsers del CLI y del orquestador extendiendo la validación de esquemas exactos (Zod) ya existente en el monorepo a las entradas críticas que todavía la eluden, con validación fail-closed: si el input falla, el proceso se interrumpe y reporta el error, sin intentar "recuperar" estados inválidos.
- **Scope**: convergencia de los 3 loaders ad-hoc de `axiom.config/profiles.yaml` en `apps/cli` (`src/commands/roles.ts`, `src/commands/init.ts`, `src/commands/topology.ts`) sobre el schema canónico `InstallProfilesYamlSchema` de `@axiom/config-validation`; nueva función `validateInstallProfilesYamlContent` (+ tipo `InstallProfilesYamlValidationResult`) en ese package; nuevo código `AXIOM_INVALID_CONFIG` en el catálogo de `@axiom/core`; dependencia real `@axiom/config-validation` agregada a `apps/cli`.
- **Non-goals**: no se migra ningún otro parser del repo en este incremento (la spec original ya lo declaraba: "solo las entradas críticas"). No se toca `axiom.yaml`/`integrations.yaml`/`policy-as-code.yaml`/`capabilities.yaml`/`providers.yaml` (ya validados desde antes por `@axiom/config-validation`, fuera de este alcance). No se migra el patrón de argumentos CLI (`IncrementSubcommandArgsSchema`/`.safeParse`, ya existente y fuera de este incremento). No se toca ningún throw site clasificado como "external YAML parsing" en `INC-20260730-typed-recovery` salvo los 3 de `profiles.yaml` explícitamente retomados acá (ver "Decisiones de implementación").

## Corrección del framing original (importante)

El README original de este incremento decía "instalar `zod`" como si el proyecto no lo tuviera. Auditoría previa a la implementación confirmó que **eso ya no es cierto**: `zod` ya es dependencia de `apps/cli` y de los 9 packages de adapters, y ya existe un package maduro de validación (`@axiom/config-validation`) con `validateWithSchema` genérico y schemas concretos para `axiom.yaml` (v1 y v2), `integrations.yaml`, `policy-as-code.yaml`, `capabilities.yaml` y `providers.yaml` — incluyendo ya `InstallProfilesYamlSchema`, que estaba definido pero **sin ninguna función `validate*YamlContent` que lo usara** y sin ningún consumidor real en `apps/cli`. El Scope/Goal de este archivo se reescribió para reflejar que el trabajo real es **extender** ese validador existente (agregar la función de validación de contenido que faltaba + converger los 3 loaders que lo evitaban), no instalar nada desde cero. Ver la sección de Decisiones para el detalle completo de por qué se reescribió el framing.

## Acceptance Criteria

- [x] La validación utiliza un esquema exacto (Zod): `InstallProfilesYamlSchema`, ya existente en `@axiom/config-validation`, ahora efectivamente usado por los 3 call sites de `profiles.yaml` en `apps/cli` a través de la nueva `validateInstallProfilesYamlContent`.
- [x] Fallas en el parseo resultan en un error claro y salida temprana (fail-closed): un `profiles.yaml` estructuralmente inválido o con YAML malformado hace que `loadProfilesYaml`/`tryLoadProfilesYaml`/`loadProfilesYamlForValidation` lancen `AxiomError(AXIOM_INVALID_CONFIG, <mensaje con path+regla de cada error de schema>)` — nunca devuelven un objeto parcialmente parseado.

## Implementation Plan

1. Auditar `@axiom/config-validation` (schemas, validator, tests) y los 3 loaders de `profiles.yaml` en `apps/cli` para confirmar que `ProfilesYaml` (`@axiom/install-profiles`) es estructuralmente compatible con `InstallProfilesYamlSchema` (campo por campo) antes de converger.
2. Agregar `validateInstallProfilesYamlContent(rawYaml: string)` + tipo `InstallProfilesYamlValidationResult` en `packages/config-validation/src/validator.ts` (mismo patrón que `validateIntegrationsYamlContent`/`validatePolicyYamlContent`, con el agregado de exponer `data` tipado en el caso válido, análogo a `validateAxiomYamlContent`). Exportar ambos desde el barrel `src/index.ts`.
3. Agregar `AXIOM_INVALID_CONFIG` al catálogo `AXIOM_ERROR_CODES` de `packages/core/src/error.ts`, documentado y atado a los 3 throw sites reales que se migran en el mismo cambio (convención ya establecida por `INC-20260730-typed-recovery`).
4. Converger `loadProfilesYaml` (`apps/cli/src/commands/roles.ts`), `tryLoadProfilesYaml` (`init.ts`) y `loadProfilesYamlForValidation` (`topology.ts`) sobre `validateInstallProfilesYamlContent`, preservando la postura exacta de cada uno frente a la ausencia del archivo (ver Decisiones).
5. Agregar `@axiom/config-validation` como dependencia real de `apps/cli` en `package.json` (el `tsconfig.json` ya tenía el path/reference declarado, sin consumidor real).
6. Agregar tests: unitarios en `packages/config-validation` (válido, inválido por campo faltante, forma incorrecta, YAML malformado) y de integración en `apps/cli` (`roles.test.ts`, `topology.test.ts`, `init.test.ts`) cubriendo válido / inválido fail-closed / ausente-sigue-siendo-legítimo por cada call site.
7. Validar build + tests targeted + suite completa.

## Decisiones de implementación

### Por qué se reescribió el framing "instalar zod"

La spec original fue redactada antes de auditar el estado real del código. Al implementar se confirmó (con `grep`/lectura directa) que: (a) `zod` ya es `dependency` de `apps/cli/package.json` y de los 9 `packages/adapters/*`; (b) `@axiom/config-validation` ya exporta `AxiomYamlSchema`, `AxiomYamlSchemaV2`, `IntegrationsYamlSchema`, `PolicyYamlSchema`, `CapabilityModelYamlSchema`, `ProviderRegistryYamlSchema` y `InstallProfilesYamlSchema`, todos ya usados por `validate*YamlContent` — EXCEPTO `InstallProfilesYamlSchema`, que existía en `schemas.ts` pero no tenía ninguna función de validación de contenido asociada ni ningún consumidor. Instalar `zod` de nuevo o crear un segundo sistema de validación hubiera sido exactamente la arquitectura especulativa/duplicada que `AGENTS.md` prohíbe. Se corrigió el Goal/Scope de este archivo para decir explícitamente "extender lo existente", y se documenta acá el porqué del cambio de wording para que el registro sea honesto sobre by qué la redacción original ya no aplica.

### El gap de mayor valor: 3 loaders de `profiles.yaml` sin converger

`apps/cli` no importaba `@axiom/config-validation` en ningún lado (sólo aparecía en un COMENTARIO de `init.ts:301`, sobre `AxiomYamlSchemaV2`, sin relación con `profiles.yaml`). Mientras tanto había 3 loaders de `axiom.config/profiles.yaml` con lógica duplicada (`fs.readFileSync` + `yaml.load` + shape-check manual `typeof parsed !== 'object'`) y fallando de forma inconsistente entre sí:

| Call site | Firma previa | Postura ante ausencia |
|---|---|---|
| `roles.ts#loadProfilesYaml` | `(projectRoot) => ProfilesYaml \| null` | `null` (legítimo) |
| `init.ts#tryLoadProfilesYaml` | `(yamlPath) => ProfilesYaml \| null` | `null` (legítimo) |
| `topology.ts#loadProfilesYamlForValidation` | `(projectRoot) => ProfilesYaml` | cae a `DEFAULT_PROFILES` (nunca `null`) |

Antes de converger se verificó que `ProfilesYaml` (`packages/install-profiles/src/types.ts`) es estructuralmente idéntico, campo por campo (incluyendo cuáles son opcionales), a `InstallProfilesYamlSchema` (`packages/config-validation/src/schemas.ts`) — confirmado además por el test preexistente `packages/install-profiles/tests/default-profiles.test.ts` ("`DEFAULT_PROFILES` valida contra `InstallProfilesYamlSchema`") y por inspección del `axiom.config/profiles.yaml` real del propio repo Axiom. No hubo necesidad de adaptar el schema: es un fit exacto, no una migración forzada.

### Postura fail-closed vs. ausencia legítima, decidida por call site

Los 3 call sites lanzaban `Error` crudo o devolvían `null`/fallback de forma ad-hoc; se preservó EXACTAMENTE esa postura, sólo reemplazando el cuerpo interno (parseo + shape-check manual) por una sola llamada a `validateInstallProfilesYamlContent` + un `throw new AxiomError(AXIOM_INVALID_CONFIG, ...)` en vez de `throw new Error(...)`:

- **`roles.ts#loadProfilesYaml`**: sigue devolviendo `null` si el archivo NO existe (ausencia legítima — `runRolesList` degrada a lista vacía con mensaje informativo; `runRolesAdd` arranca desde un `ProfilesYaml` vacío). Sigue lanzando (ahora `AxiomError`) si el archivo existe pero no es YAML parseable o no cumple el schema — ambos callers ya envuelven la llamada en `try/catch` y mapean a `{message, exitCode: 1}`, así que el cambio de tipo de excepción (`Error` → `AxiomError`, subclase de `Error`) es 100% compatible sin tocar los callers.
- **`init.ts#tryLoadProfilesYaml`**: mismo criterio — `null` ante ausencia (el caller itera varios paths candidatos y trata cada ausencia como "seguir probando el siguiente candidato"), `AxiomError` fail-closed ante forma inválida. El caller YA distinguía: si el profile pedido es canónico (`product-owner`/`builder`), un candidato corrupto se ignora (sigue probando/cae a `DEFAULT_PROFILES`); si el profile es un alias (`analista`/`arquitecto`, necesita el `roleAliases` del archivo real), el error se RE-PROPAGA. Ese comportamiento no se tocó — sigue siendo el caller quien decide, el loader sólo fail-closed reporta.
- **`topology.ts#loadProfilesYamlForValidation`**: sigue devolviendo `DEFAULT_PROFILES` ante ausencia (invariante de `INC-20260710-dynamic-team-roles`: `validRoleIds` nunca debe quedar vacío sólo porque el proyecto no tiene `profiles.yaml` propio). Sigue lanzando `AxiomError` fail-closed ante forma inválida — el caller (`runTopologyValidate`) ya capturaba cualquier `Error` y mapeaba a `exitCode: 1`.

"Fail-closed" en este incremento significa exactamente eso: un `profiles.yaml` PRESENTE pero inválido bloquea con error claro; un `profiles.yaml` AUSENTE donde la ausencia ya era un estado legítimo documentado NO se volvió fatal — no se introdujo una regresión de "archivo opcional ahora obligatorio".

### `validateInstallProfilesYamlContent`: por qué expone `data` (a diferencia de `validateWithSchema`)

Las otras 4 funciones `validate*YamlContent` (integrations/policy/capability-model/provider-registry) devuelven sólo `{valid, errors}` porque sus callers actuales no necesitan el objeto parseado. Los 3 call sites de `profiles.yaml`, en cambio, necesitan el `ProfilesYaml` ya parseado como valor de retorno (no sólo un booleano). Se diseñó `InstallProfilesYamlValidationResult` como una unión discriminada `{valid: true; data} | {valid: false; errors}`, igual al patrón ya establecido por `AxiomYamlValidationResult` (para `axiom.yaml`), en vez de forzar a cada call site a volver a parsear el YAML después de validarlo.

### Nuevo código `AXIOM_INVALID_CONFIG`

Se agregó al catálogo cerrado de `packages/core/src/error.ts`, siguiendo la convención documentada por `INC-20260730-typed-recovery` ("agregar un código nuevo va de la mano de migrar el/los throw site(s) correspondientes en el mismo cambio"): grounded en los 3 throw sites migrados en este incremento. Se decidió NO reutilizar `AXIOM_INVALID_OPTION` (ya existente, para input de CLI: flags/enums/argumentos) porque `profiles.yaml` es configuración persistida en disco, no un argumento de invocación — la distinción le permite a un subagente ramificar entre "el operador escribió mal un flag" y "hay un archivo de config roto en el proyecto", que ameritan remediaciones distintas.

### Qué se dejó deliberadamente sin migrar (blast radius acotado)

Consistente con el non-goal explícito ("sólo las entradas críticas"):

- **`axiom.yaml`/`integrations.yaml`/`policy-as-code.yaml`/`capabilities.yaml`/`providers.yaml`**: ya validados desde antes de este incremento; no se tocaron.
- **`IncrementSubcommandArgsSchema`/argumentos de CLI**: el patrón `.safeParse` ya existe (`axiom-increment.ts`, `axiom-bug.ts`) y ya es fail-closed; no requería cambios.
- **Los demás throw sites de parseo YAML externo documentados como non-goal explícito por `INC-20260730-typed-recovery`** (`sync.ts`, `gateway.ts`, `audit.ts`, `configure.ts`, y cualquier otro fuera de los 3 loaders de `profiles.yaml`): quedan fuera de este incremento también. Sólo se retomaron los 3 loaders de `profiles.yaml` porque son exactamente el gap que este incremento pidió cerrar (config YAML crítica sin schema exacto); el resto sigue fuera de scope.
- **`writeProfilesYaml`/escritura de `profiles.yaml`** (en los 3 archivos): no se valida el objeto ANTES de escribirlo (sólo se valida al LEER). Fuera de alcance: los objetos que se escriben se construyen en memoria a partir de datos ya validados (el `ProfilesYaml` cargado + una mutación puntual tipada por TypeScript), no de input externo no confiable.

## Validación y review

- `npm run build` (`tsc -b`, monorepo completo): **pasa**, sin errores.
- `npx vitest run apps/cli packages/config-validation packages/core packages/install-profiles`: **139 archivos, 1356 passed / 6 failed (de 1362)**. Los 6 fallos son exactamente los documentados como pre-existentes y fuera de alcance:
  - `packages/install-profiles/tests/composer.test.ts` (5 fallos deterministas, matrices de capabilities `builder`/`product-owner` y `opencode` — confirmado que NO están relacionados con `profiles.yaml`/schema de este incremento; el archivo de test no fue tocado).
  - `apps/cli/tests/launcher-panels.test.ts` (1 timeout) — re-ejecutado EN AISLAMIENTO (`npx vitest run apps/cli/tests/launcher-panels.test.ts`): **14/14 pasa**, confirmando el flake de contención documentado.
- Suite dirigida a los archivos tocados (`apps/cli/tests/roles.test.ts`, `apps/cli/tests/init.test.ts`, `apps/cli/tests/topology.test.ts`, `packages/config-validation/tests/validator.test.ts`, `packages/core/tests/error.test.ts`): **78/78 pasa** (incluye los 11 tests nuevos: 4 en `validator.test.ts`, 3 en `roles.test.ts`, 2 en `topology.test.ts`, 2 en `init.test.ts`).
- Suite global completa (`npx vitest run`, sin filtro): **336 archivos, 3489 tests, 3484 passed, 5 failed**. Los 5 fallos son exactamente los mismos 5 de `composer.test.ts` (pre-existentes); en esta corrida `launcher-panels.test.ts` y `packages/memory/tests/engram-backend.test.ts` NO flakearon (0 de los 2 flakes documentados se manifestaron), consistente con su naturaleza de contención intermitente. El total de tests subió de la referencia post-batch (3478) a 3489 (+11), exactamente los tests nuevos agregados por este incremento.
- No se detectó ninguna regresión: ningún test que pasaba antes de este cambio falla ahora; los 3 call sites migrados mantienen su contrato público (`null`/`DEFAULT_PROFILES`/throw) sin cambios observables para sus callers salvo el mensaje de error (ahora incluye el detalle de cada campo inválido) y el tipo de excepción (`AxiomError` en vez de `Error` crudo, subclase compatible).

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

Lo aportado por ESTE incremento quedó en: RF-AXM-061 con el matiz ausente-vs-inválido (`01`), `AXIOM_INVALID_CONFIG` (`03`), `@axiom/config-validation` como validador canónico y la regla "un schema sin consumidores es deuda" (`06`), término `Fail-closed sobre inválido, no sobre ausente` (`08`), resumen de tanda (`00`).

### Contexto técnico (`Axiom.Spec/context/**`)

**Sí aplicó.** Documentos actualizados por este incremento: `references/01-inventario-de-packages.md` (filas `@axiom/config-validation` y su nuevo consumo desde `apps/cli`), `references/02-historial-de-incrementos.md`.

El pase de contexto no fue solo aditivo: se corrigió el punto 5 de `context/TECHNICAL_CONTEXT.md`, que declaraba `TC-011` como bloqueo vigente y citaba "3425/3427 tests" — `npm run doctor` da hoy `PASS` (46/61 OK, 0 fallos) y la suite vigente es 3489 tests / 3483 verdes / 6 rojos preexistentes.

## Estado de cierre

El incremento cumple las 2 acceptance criteria: la validación de `profiles.yaml` en los 3 call sites de `apps/cli` usa ahora el esquema exacto `InstallProfilesYamlSchema` (Zod, `@axiom/config-validation`) a través de la nueva `validateInstallProfilesYamlContent`, y las fallas de parseo/schema resultan en `AxiomError(AXIOM_INVALID_CONFIG)` con mensaje claro y salida temprana (fail-closed), preservando la ausencia legítima del archivo donde ya lo era. El framing original "instalar zod" se corrigió explícitamente (ver sección dedicada) porque zod y `@axiom/config-validation` ya existían; el trabajo real fue de extensión y convergencia, no de instalación. `npm run build` pasa sin errores; la suite dirigida a los archivos tocados pasa 78/78; la suite global pasa 3484/3489 con los únicos 5 fallos siendo los pre-existentes y documentados de `composer.test.ts` (fuera de alcance, no tocados por este incremento), y los 2 flakes de contención conocidos no se manifestaron en esta corrida (confirmados no relacionados al re-ejecutar `launcher-panels.test.ts` en aislamiento). Se marca `Status: closed`.
