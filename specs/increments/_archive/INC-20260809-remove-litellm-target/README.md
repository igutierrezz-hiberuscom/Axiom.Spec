# Retirar LiteLLM como destino

> **Codigo**: INC-20260809-remove-litellm-target
> **Estado**: closed
> **Fecha de creacion**: 2026-08-09
> **Tipo de cambio**: eliminar

## Objetivo

Retirar LiteLLM del contrato de destinos de Axiom porque no forma parte de la
experiencia de agente/editor que el producto va a mantener.

## Contexto y motivacion

El repositorio contiene un adapter `@axiom/adapters-litellm` que escribe
`litellm.config.json`, pero el registry del instalador y las pruebas de
`installProfile` lo describen como proxy sin archivos. La implementacion debe
resolver esa contradiccion eliminando el target activo, no dejando un fallback
silencioso.

## Alcance tipado

- Repositorio de codigo: `Axiom/`.
- Configuracion y tipos: `axiom.config/profiles.yaml`, capability/model routing
	y cualquier lista canonica de targets que incluya `litellm`.
- Dispatchers: `packages/cli-commands/src/commands/configure.ts`,
	`packages/cli-commands/src/commands/sync.ts` y
	`apps/cli/src/commands/workspace-adapters.ts`.
- Registry, support matrix, docs y tests que traten `litellm` como target.
- Paquete `packages/adapters/litellm/` solo si queda sin consumidores despues
	de retirar el contrato.

## No incluido

- No retirar ni cambiar el model routing de OpenCode, Claude Code u otros
	destinos conservados.
- No eliminar referencias historicas en incrementos archivados.
- No modificar MCP ni los providers locales salvo que una referencia activa a
	LiteLLM lo exija.

## Criterios de aceptacion

1. `litellm` no aparece como target aceptado en configuracion, tipos, aliases,
	 support matrix, installer registry, CLI ni workspace setup.
2. Ningun dispatcher intenta generar `litellm.config.json`.
3. El paquete LiteLLM se elimina o queda claramente fuera del build por no
	 tener consumidores; no queda un paquete muerto declarado como adapter activo.
4. Un estado legacy con `adapterTarget: litellm` migra o falla con un mensaje
	 explicito y seguro; nunca se convierte silenciosamente en otro target.
5. Las pruebas focalizadas y el build de Axiom pasan; las referencias
	 historicas permanecen solo en secciones historicas.

## Validacion prevista

- Buscar referencias activas y comprobar ausencia de `litellm` en los contratos.
- Ejecutar tests de installer, install-profiles, model-routing, adapters y CLI
	afectados.
- Ejecutar `npm run build` desde `Axiom/`.

## Integracion de spec y contexto

No se integraron specs generales ni contexto tecnico; este incremento solo
retira un contrato activo del repositorio `Axiom`.

## Notas de implementacion

- LiteLLM fue retirado de `profiles.yaml`, `DEFAULT_PROFILES`,
	`CANONICAL_ADAPTER_TARGETS`, la support matrix, el registry del installer,
	los dispatchers de `sync` y workspace setup, la superficie CLI, docs y
	fixtures afectados.
- Se agrego `REMOVED_ADAPTER_TARGETS` junto con
	`RemovedAdapterTargetError`: la normalizacion y composicion rechazan un
	`adapterTarget: litellm` legacy de forma explicita. `sync` preserva ese
	error al leer `install-profile.json` y no cae silenciosamente a `opencode`.
- `packages/adapters/litellm/` y sus referencias de workspace, TypeScript,
	npm y lockfile fueron eliminados. Los demas adapters y el model routing
	general permanecen sin cambios de contrato.

## Validacion

- `npx vitest run packages/adapters packages/installer/tests/installer.test.ts packages/install-profiles/tests/default-profiles.test.ts packages/model-routing/tests/support-matrix.test.ts` — 15 archivos, 175 tests OK.
- `npx vitest run apps/cli/tests/configure.test.ts apps/cli/tests/sync.test.ts apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/workspace-mcp.test.ts apps/cli/tests/native-mcp-config.test.ts apps/cli/tests/context.test.ts apps/cli/tests/document-bootstrap.test.ts apps/cli/tests/e2e/adapters.e2e.test.ts` — 8 archivos, 111 tests OK.
- `npx vitest run apps/cli/tests/sync.test.ts` — 13 tests OK, incluido el rechazo legacy sin marker parcial.
- `npm run build` — OK (`tsc -b`).
- `npm install --package-lock-only --ignore-scripts` — OK; `npm ls @axiom/adapters-litellm --all` no encuentra el package.
- El barrido de referencias activas deja LiteLLM únicamente en la lista de
	targets retirados y en las pruebas del fallo legacy explícito.

## Resultado

Los cinco criterios de aceptacion se cumplen. No se observaron fallos de
tests o build preexistentes; `npm install` sigue reportando 8 vulnerabilidades
del arbol de dependencias, sin relacion causal con este incremento.

## Trazabilidad final

- Freeze vigente: `51416d18813d09a92094eaf5a0035e1fa508325e31df4900af8e51cb5ffeafeb`.
- Receipts finales `freeze-final` y `verify-final` validados por SHA-256 por el
	orquestador; el runtime fue comprobado después del freeze vigente.
- Integración general y contexto técnico: aplicada por el orquestador; no se
	conserva LiteLLM como contrato activo.

## General spec integration

- Specs `00..08`: targets activos, LiteLLM retirado, aliases legacy y salidas
	de adapters reconciliados.
- `context/architecture/01-02`, `context/architecture/04`,
	`context/integrations/01`, `context/references/01` y `context/TECHNICAL_CONTEXT.md`:
	conteo de packages y estado de adapters actualizado.
