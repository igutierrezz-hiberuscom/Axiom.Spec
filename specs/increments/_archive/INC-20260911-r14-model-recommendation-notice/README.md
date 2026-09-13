# r14 model recommendation notice

> **Código**: INC-20260911-r14-model-recommendation-notice
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: contrato de recomendación y aviso al operador
> **Acción de origen**: `ACC-086`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 7 de 8)

## Resumen

Cuando el destino seleccionado no enruta modelo, mostrar el modelo recomendado, declarar que la selección es manual y avisar de ello al copiar el prompt según el adapter. Donde el destino sí enruta, el aviso no aparece.

## Contexto y motivación

Verificado el 2026-09-11:

- `SUPPORT_MATRIX` en `packages/model-routing/src/support-matrix.ts` declara `opencode` como `multi-mode`, `claude-code` como `single-mode` y `github-copilot`, `vscode`, `cursor`, `codex`, `antigravity` y `visual-studio-2026` como `fallback-only`.
- En esos seis destinos, `axiom model set` no tiene efecto sobre la herramienta: el mando existe y no mueve nada.
- `axiom.config/model-routing-policy.yaml` declara `fallback: whenPerSubagentRoutingUnsupported: use-default-and-record-limitation`, pero no se encontró ninguna superficie donde esa limitación quede registrada o se muestre al usuario.
- `packages/launcher/src/prompt-builder.ts` y `adapter-routing.ts` ya varían cabecera/mención y verbo de lanzamiento por destino manteniendo el cuerpo del prompt idéntico, lo que da el punto de inserción natural del aviso sin duplicar lógica.

Decisión de producto tomada por el usuario: no se promete control donde no existe; se ofrece recomendación honesta más una acción manual explícita.

## Alcance

### Incluido

- Cálculo del modelo recomendado por slot, derivado de la política y de los overrides vigentes.
- Aviso dependiente del adapter al copiar el prompt, diferenciado para los tres niveles del `SUPPORT_MATRIX`.
- Ausencia de aviso en los destinos que enrutan por slot.
- Registro efectivo y visible de la limitación que hoy la política solo declara.
- `SUPPORT_MATRIX` como única fuente de verdad del nivel de soporte.

### Excluido

- Convertir un destino `fallback-only` en routing real.
- Prometer o simular comportamiento del IDE.
- Introducir un segundo criterio de soporte paralelo al `SUPPORT_MATRIX`.
- Las pantallas del launcher, que pertenecen a `ACC-082`.

## Dudas abiertas

Ninguna. Cerradas en D-04 (`DEC-20260911-223525-y8j1xc`):
- Se nombra la `ModelClass` recomendada (`cheap`/`medium`/`strong`/`local`).
- Se definen textos tipados para los 3 niveles del `SUPPORT_MATRIX`.
- Inserción en cabecera de prompts en el launcher solo para acciones con slot SDD.
- Coherencia en `axiom model show` y registro visible de limitaciones.

## Decisiones funcionales cerradas

- Selección manual declarada de forma explícita donde no hay enrutado.
- El aviso depende del adapter seleccionado y del slot de la acción.
- Cero promesas falsas de control sobre entornos `fallback-only`.

## Consolidación en la spec general

Al cierre, la spec debe describir el comportamiento por nivel de soporte y el contrato del aviso, sustituyendo cualquier afirmación que sugiera control efectivo en destinos `fallback-only`.

## Estrategia E2E

Los tres niveles de soporte; snapshot por adapter demostrando cuerpo de prompt idéntico y variación solo en cabecera, verbo y aviso; ausencia de aviso en `multi-mode`; presencia y texto correcto en `single-mode` y `fallback-only`; `npm run build`, doctor, readiness y `git diff --check`.

## Trazabilidad y fuentes

Acción `ACC-086`, decisión `DEC-20260911-223525-y8j1xc` y sesión R-14 del 2026-09-11 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.

## Estado de validación humana

Validado. Criterios AC-086-01 a AC-086-05 verificados:
- Creado módulo `recommendation.ts` en `@axiom/model-routing` exportando `formatModelNotice`, `resolveSlotRecommendation`, `resolveAllRecommendations`.
- Avisos configurados según el nivel de soporte de `SUPPORT_MATRIX`: multi-mode (sin aviso), single-mode (aviso de alcance global), fallback-only (aviso de selección manual).
- `craftPrompt` en `@axiom/launcher` integra el aviso en la cabecera para acciones con slot SDD mapeado.
- `axiom model show` enriquecido mostrando recomendación de política y soporte del adapter por cada slot.
- Cobertura completa de tests en `packages/launcher/tests/` (72 tests PASS), snapshots actualizados demostrando invariancia del cuerpo y variación exclusiva en cabecera, verbo y aviso.
- `npm run build` limpio, `npm run doctor` PASS (48/61 OK, 0 fallos), readiness PASS, `git diff --check` limpio.
