# Increment: superficie MCP-only para `axiom.*`

Status: closed
Date: 2026-08-05

## Goal

Definir `axiom.topologyRead`, `axiom.migrationManifestRead` y
`axiom.adoptionStateRead` como capabilities de la superficie MCP unificada,
separadas del modelo genérico de capabilities y providers. El broker MCP debe
seguir exponiendo los tres ids, pero la ausencia de un provider tradicional no
debe interpretarse como una configuración incompleta del provider model.

## Context

El broker `axiom` y el registro de `@axiom/mcp-tools` ya existen y atienden
los tres ids. Antes de este incremento, el capability model mezclaba esos ids
con el conjunto provider-routed: sus constantes enumeraban 19 ids, el schema
aceptaba el quinto dominio `axiom`, algunos tipos todavía decían que solo
había cuatro dominios y los perfiles incluían ids que `providers.yaml` no
declaraba.

La separación debe corregir esa mezcla sin retirar el broker, sus handlers ni
la compatibilidad de los kinds `sdd`, `spec` y `memory`.

## Scope

- `Axiom/packages/capability-model/src/types.ts` y `src/index.ts`: distinguir
  el contrato provider-routed del catálogo MCP-only sin casts que oculten el
  dominio real.
- `Axiom/packages/capability-model/src/constants.ts` y `src/schemas.ts`:
  mantener una fuente explícita para los ids MCP-only y evitar que se exijan
  providers tradicionales para ellos.
- `Axiom/packages/capability-model/src/loader.ts` y pruebas del paquete:
  cargar/representar correctamente los dos grupos y hacer explícita la
  superficie de ejecución.
- `Axiom/packages/install-profiles/src/default-profiles.ts`,
  `Axiom/axiom.config/profiles.yaml` y las fixtures directamente afectadas:
  no tratar `axiom.*` como capabilities provider-routed del perfil.
- `Axiom/packages/mcp-tools` y `packages/mcp-server` solo en las pruebas o
  tipos estrictamente necesarios para demostrar que los tres ids siguen
  registrados y visibles en el broker.

## Non-goals

- No retirar ni renombrar el broker MCP `axiom`.
- No retirar los tres handlers ni cambiar su aislamiento project-scoped.
- No añadir un provider nuevo ni mapear automáticamente `axiom.*` a
  `axiom-gateway`.
- No corregir `CC-004` en este incremento; el bug y la política de Doctor se
  tratan en artefactos separados y posteriores.
- No integrar todavía la spec general ni `context/**`; lo hará el autopilot
  una sola vez al final.

## Acceptance criteria

- [x] El modelo provider-routed y la superficie MCP-only tienen contratos
      explícitos y no dependen de casts incompatibles.
- [x] Los tres ids `axiom.*` están identificados en una lista/metadata
      MCP-only estable y reutilizable por consumidores posteriores.
- [x] El mapa provider-routed rechaza definiciones con dominio `axiom` y el
  mapa MCP-only rechaza ids `axiom.*` desconocidos.
- [x] `config-validation` conserva y valida la forma `mcpOnlyCapabilities`
  sin eliminarla al parsear el YAML.
- [x] `axiom.*` no requiere una entrada de provider tradicional para que el
      modelo provider-routed sea válido.
- [x] La composición de perfiles no incluye `axiom.*` como capabilities que
      deban resolverse por provider.
- [x] El registro MCP conserva los tres handlers y el broker `axiom` conserva
      exactamente su conjunto de herramientas actual.
- [x] Los consumidores tipados de `CapabilityModel` y `RouteToolContext`
  conocen el mapa MCP-only.
- [x] Las pruebas demuestran la separación y una prueba de routing genérico
      no confunde MCP-only con provider-routed.
- [x] El build y las suites focalizadas de capability-model, install-profiles,
      mcp-tools y mcp-server pasan.

## Open questions

Ninguna bloqueante. Se elige una separación explícita y conservadora: MCP
mantiene sus ids y handlers; el modelo genérico de providers no los usa como
obligación de cobertura.

## Assumptions

- El registro MCP (`MCP_TOOL_CAPABILITY_IDS` y `MCP_TOOL_HANDLERS`) sigue siendo
  la fuente operativa de los ids `axiom.*`.
- `axiom-gateway` puede seguir siendo el transporte del broker MCP, pero eso
  no convierte los tres ids en capabilities provider-routed del modelo
  genérico.
- Las capabilities provider-routed canónicas restantes conservan sus ids y
  estados actuales.

## Implementation notes

Se separaron los contratos `ProviderRoutedCapabilityDefinition` y
`McpOnlyCapabilityDefinition`; `CANONICAL_CAPABILITY_IDS` representa solo la
superficie provider-routed y `MCP_ONLY_CAPABILITY_IDS` conserva los tres ids
`axiom.*`. `loadCapabilityModel` devuelve ambos mapas y solo construye cadenas
de fallback para el mapa provider-routed. Los schemas rechazan dominios
MCP-only en el mapa provider-routed y rechazan ids `axiom.*` desconocidos. Los
perfiles bundled y dogfooded ya no habilitan `axiom.*`; los ejemplos MCP
tampoco declaran esos ids bajo `axiom-gateway`.

## Validation

- `npx vitest run packages/capability-model/tests packages/config-validation/tests packages/install-profiles/tests packages/mcp-tools/tests packages/mcp-server/tests`: 11 archivos y 145 pruebas pasaron en la última batería dirigida.
- `npx vitest run packages/mcp-tools/tests/registry.test.ts packages/mcp-server/tests/server.test.ts packages/mcp-server/tests/axiom-broker.e2e.test.ts`: 3 archivos y 44 pruebas pasaron.
- `npm run build`: paso (`tsc -b`).
- `get_errors` sobre los archivos fuente modificados: sin errores.
- `checkCandidateFreeze(INC-20260805-mcp-only-axiom-capabilities)` con `AXIOM_TEST_FORCE_JSON=1`: `ok: true` antes de documentar este resultado.
- Receipt verificable final: `receipts/2026-08-05T15-53-49.873Z-verify-success.json`, hash `aa11722b5ccc4b026b3fe0a0f35cac0e83432c955f39a94f1e4fb6af4ee5eddc`.

## Result

Implementación completada. Los tres ids `axiom.*` siguen registrados en
`MCP_TOOL_HANDLERS` y visibles en el broker `axiom`, pero no forman parte del
modelo que el routing genérico resuelve mediante providers. Los contratos
provider-routed y MCP-only son estrictos y los consumidores tipados exponen
ambos mapas. No se introdujeron fallos en las validaciones ejecutadas.

La integración estable en `Axiom.Spec/specs/00..08` y `context/**` quedó
consolidada durante la pasada única de R-04.

## General spec integration

Integrado en la pasada única de R-04:

- `specs/00_Resumen_Ejecutivo.md`, `01_Requisitos_Funcionales.md` y
  `02_Requisitos_No_Funcionales.md`: separación de superficies y contrato de
  cobertura.
- `specs/03_Modelo_Operativo_y_Datos.md`: mapas y dominios del capability
  model.
- `specs/05_Interfaces_Operativas.md`, `06_Integraciones_y_Capacidades.md`,
  `07_Gobierno_y_Seguridad.md` y `08_Glosario.md`: broker MCP, MCP-only,
  providers y nomenclatura vigente.
- `context/architecture/02-modelo-de-datos-y-configuracion.md` y
  `context/integrations/01-capabilities-providers-y-toolchain.md`: forma real
  de los mapas y separación de ejecución.

No se crearon ni eliminaron documentos de `context/**`; se actualizaron los
documentos propietarios existentes.

El artefacto queda archivado por el autopilot.
