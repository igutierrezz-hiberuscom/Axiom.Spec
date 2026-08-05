# Bug: `config-validation` rechaza el kind real de `cmm`

Status: closed
Date: 2026-08-05

## Symptom

`Axiom/axiom.config/providers.yaml` declara `cmm` con
`kind: structural-code-intel`. El schema de `@axiom/capability-model` lo
acepta, pero `@axiom/config-validation` rechaza el mismo archivo.

## Current behavior

`validateProviderRegistryYamlContent` devuelve un error en
`providers[3].kind` porque su enum de `ProviderKind` no contiene
`structural-code-intel`. El loader del capability model y las comprobaciones
actuales de Doctor sí aceptan y procesan ese valor.

## Expected behavior

Los validadores públicos que describen el mismo contrato de `providers.yaml`
deben aceptar `structural-code-intel` para `cmm`. Una validación del archivo
real de Axiom debe ser válida y no debe depender de qué paquete la ejecute.

## Impact

La configuración vigente puede fallar en scripts, CI o comandos que utilicen
`@axiom/config-validation`, aunque las operaciones principales que cargan el
capability model sigan funcionando.

## Reproduction steps

1. Leer `Axiom/axiom.config/providers.yaml`.
2. Ejecutar `validateProviderRegistryYamlContent` sobre su contenido.
3. Observar el error `providers.3.kind` para `structural-code-intel`.
4. Ejecutar `ProviderRegistrySchema.safeParse` de `@axiom/capability-model`
   sobre el mismo YAML y observar que es válido.

## Suspected cause

El enum `ProviderKindEnum` de `packages/config-validation/src/schemas.ts`
quedó con la lista anterior a la incorporación de `cmm`, mientras que el
schema propietario de `capability-model` ya fue actualizado.

- [x] `config-validation` acepta `structural-code-intel`.
- [x] El YAML real de `Axiom/axiom.config/providers.yaml` pasa
      `validateProviderRegistryYamlContent`.
- [x] El schema de `capability-model` sigue aceptando el provider `cmm`.
- [x] Existe una prueba focalizada para el kind `structural-code-intel`.
- [x] La suite de `config-validation` y las pruebas de `capability-model`
      pasan sin regresiones.

## Fix notes

La corrección de este bug se limita a añadir `structural-code-intel` al enum
público de `@axiom/config-validation` y cubrir el valor con una prueba de
regresión. Los cambios MCP-only que también aparecen en el mismo archivo de
schema pertenecen a `INC-20260805-mcp-only-axiom-capabilities`; no forman
parte de este bug ni alteran su alcance. No se altera la selección de
providers ni se introduce ningún provider nuevo.

## Validation

- `npx vitest run packages/config-validation/tests/validator.test.ts`: 26
      pruebas pasaron.
- `npx vitest run packages/config-validation/tests/validator.test.ts
      packages/capability-model/tests/loader.test.ts
      packages/capability-model/tests/resolver.test.ts`: suites focalizadas
      verdes como parte de la verificación cruzada del contrato.
- `npm run build`: pasó (`tsc -b`).
- El `providers.yaml` real pasó `validateProviderRegistryYamlContent`.
- `git diff --check`: sin errores en el diff focalizado.
- Receipt `verify` final: `receipts/2026-08-05T16-05-39.520Z-verify-success.json`,
  hash `f0f056d44c2f75d82bb2e82cb609b7e6efd9d5f7125457a009630d271e571e4b`.

## Result

El bug está corregido a nivel de código y pruebas. La integración estable se
consolidó en `specs/00..08` y `context/**` durante este lote.

## General spec integration

Integrado en la pasada única de R-04:

- `specs/03_Modelo_Operativo_y_Datos.md`: el contrato de providers queda
      alineado con la configuración real.
- `specs/06_Integraciones_y_Capacidades.md`: se conserva `cmm` como provider
      estructural vigente.
- `context/integrations/01-capabilities-providers-y-toolchain.md`: se mantiene
      la referencia a `structural-code-intel` como kind actual.
- `context/**`: sin otros cambios requeridos por este bug.

El artefacto queda archivado por el autopilot.
