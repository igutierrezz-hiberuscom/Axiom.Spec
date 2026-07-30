# INC-20260730-capability-enablement-kill-switches

Status: closed
Date: 2026-07-30
Plan: F1.2

## Goal

Extender el modelo de capabilities y sus consumidores para representar los estados `enabled`, `disabled`, `experimental` y `unavailable`, aplicando un bloqueo fail-closed antes de cualquier seleccion o ejecucion de provider.

## Context

`axiom.config/capabilities.yaml` usa actualmente listas de compliance y el composer ya materializa `enabledCapabilities` y `disabledCapabilities`. El resolver y el dispatcher necesitan interpretar el estado de la capability sin crear un registry paralelo ni fabricar una aprobacion cuando una capability esta bloqueada.

## Scope

- Anadir `CapabilityState` al modelo, sus schemas y la carga desde `capabilityStates` en el mismo `capabilities.yaml`.
- Mantener compatibilidad con las listas `required`, `optional` y `postMvpOptional`; aceptar `state` en la forma explicita de capability.
- Hacer que `resolveCapability` y `routeTool` bloqueen `disabled` y `unavailable` antes de policy/provider; `experimental` solo pasa si el perfil la incluye en `enabledCapabilities`.
- Mantener `auto_push` fuera de las capabilities habilitadas por defecto y sin registry adicional.
- Anadir tests focalizados para los cuatro estados, perfiles, validacion YAML y routing fail-closed.

## Non-goals

- Cambiar los 16 IDs canonicos.
- Crear otro sistema de kill switches, un provider nuevo o infraestructura de ejecucion.
- Habilitar `auto_push` por defecto.
- Modificar `Axiom.Spec/specs/00..08`, `Axiom.Spec/context/**` o integrar la spec general.
- Archivar este incremento.

## Acceptance criteria

- [x] El modelo y los schemas aceptan exactamente los cuatro estados y rechazan valores desconocidos.
- [x] El formato actual de listas sigue cargando; el estado se resuelve desde el mismo YAML y `disabled`/`unavailable` tienen precedencia sobre el perfil.
- [x] `experimental` solo se puede enrutar cuando `enabledCapabilities` lo habilita explicitamente; `disabledCapabilities` sigue bloqueandolo.
- [x] `resolveCapability` no devuelve provider ni veredicto permitido para estados bloqueados.
- [x] `routeTool` devuelve un resultado degradado tipado, con `providerEffective` vacio, sin seleccionar ni ejecutar provider para estados bloqueados.
- [x] `auto_push` permanece deshabilitado por defecto y no aparece un registry paralelo.
- [x] Existen tests focalizados para estados, composer, validacion y fail-closed de routing.
- [x] La validacion focalizada y `npm run build` se ejecutan; los fallos se clasifican.

## Risks

- Consumidores que construyen `CapabilityModel` manualmente pueden omitir el nuevo campo; se conservara un default compatible para fixtures legacy.
- Una configuracion con `capabilityStates` incompleto podria dejar capabilities sin habilitacion explicita; el loader aplicara default fail-closed para ese mapa.
- El dispatcher no ejecuta providers por si mismo, pero su salida vacia debe conservarse para que la capa de ejecucion no reciba un provider ficticio.

## Open questions

- Ninguna bloqueante. La forma elegida es un mapa `capabilityStates` dentro del archivo existente; no se modifica `policy-as-code.yaml` porque el bloqueo pertenece al modelo de capability y al perfil.

## Assumptions

- La ruta canonica de incrementos es `Axiom.Spec/specs/increments/`.
- La inclusion en `ResolvedInstallProfile.enabledCapabilities` es la habilitacion explicita del perfil para `experimental`.
- La ausencia de `capabilityStates` conserva el comportamiento legacy de las capabilities declaradas; cuando el mapa existe, una entrada no listada queda bloqueada.

## Implementation notes

- `capabilityStates` es metadata del mismo modelo, no un catalogo paralelo; en la forma explicita, `state` es un campo aditivo.
- Se reutiliza `DegradedReason: capability-missing` para el resultado bloqueado y se corta antes de `walkFallbackChain`.
- El YAML canonico conservara sus 16 IDs y `auto_push` no sera agregado.

## Validation

- `npx vitest run packages/capability-model/tests/loader.test.ts packages/capability-model/tests/resolver.test.ts packages/install-profiles/tests/composer.test.ts packages/config-validation/tests/validator.test.ts packages/tool-routing/tests/route-tool.test.ts`: 5 archivos, 98 tests passed.
- `npx tsc -b packages/config-validation packages/capability-model packages/install-profiles packages/tool-routing --pretty false`: passed.
- `npm run build`: bloqueado por 8 errores preexistentes en `packages/doctor` (`CATEGORY_TOOLCHAIN` no definido y `ToolchainVersionDriftOptions` ausente); no afecta a los paquetes modificados y coincide con el fallo previo del entorno.
- Revisión independiente inicial: detectó que un estado desconocido en la forma explicita podia caer en la rama legacy y permitir un routing fail-open; se corrigió con schemas estrictos y una policy fail-closed.
- Revisión independiente final: confirmó el bloqueo de estados desconocidos y `null`, la compatibilidad legacy y la ausencia de provider efectivo en rutas bloqueadas; recomendación `closed`.
- `get_errors` sobre las cuatro superficies modificadas: sin errores.
- `git diff --check`: passed; los avisos de conversion LF/CRLF son de Git y no errores de whitespace.

## Result

- `CapabilityState` y `capabilityStates` viven en el modelo y archivo YAML existentes; las listas de compliance y la forma explicita con `state` siguen siendo validas.
- El loader aplica `disabled` por defecto a entradas omitidas cuando existe un mapa de estados; sin mapa conserva compatibilidad legacy.
- Policy, resolver, compositor, dispatcher, selector y fallback respetan el bloqueo. `experimental` requiere `enabledCapabilities`; `disabled`/`unavailable` y estados inválidos nunca generan aprobación ni provider.
- El YAML canonico conserva exactamente los 16 IDs; `code.knowledgeGraph` y `code.structureAnalysis` quedan `experimental` y el builder los habilita explicitamente. `auto_push` no se declara ni se habilita.
- Fallos introducidos: ninguno. El fallo del build completo queda clasificado como preexistente en `doctor`.
- Riesgo residual: no hay tests unitarios separados para cada rama interna de `evaluatePolicy`, pero los tests de resolver, routing y la sonda en memoria cubren sus estados observables, incluido `null`.

## General spec integration

No se integra `Axiom.Spec/specs/00..08` ni `Axiom.Spec/context/**` en este incremento, por instruccion explicita.
