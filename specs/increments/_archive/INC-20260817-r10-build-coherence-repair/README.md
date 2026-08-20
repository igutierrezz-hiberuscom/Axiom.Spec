# R-10 reparación de coherencia de build

> **Código**: INC-20260817-r10-build-coherence-repair
> **Tipo de cambio**: modificar
> **Origen**: validación global obligatoria del lote R-10

## Objetivo

Restaurar `npm run build` sin reabrir decisiones funcionales cerradas: conservar la retirada de LiteLLM y del alias `copilot-vscode`, y alinear los contratos MCP y Visual Studio que el compilador detectó divergentes.

## Alcance

- Eliminar imports, dispatches y referencias tipadas activas a `@axiom/adapters-litellm` y `litellm`.
- Retirar el alias activo `copilot-vscode` de los dispatches de workspace/process surfaces, conservando sólo los adapters vigentes.
- Alinear los consumidores MCP con `McpServerContext` y el protocolo vigente, manteniendo la restricción de seguridad project-scoped.
- Alinear el caller de Visual Studio con la API actual de su generator.
- Actualizar únicamente las pruebas necesarias y ejecutar build global más suites focalizadas.

## No alcance

No restaurar LiteLLM, no crear aliases nuevos, no cambiar catálogos/política de adapters, no rediseñar MCP ni alterar comportamiento de lifecycle QA/plan. No modificar decisiones históricas ni documentación canónica hasta la consolidación final.

## Criterios de aceptación

- `npm run build` termina con exit 0.
- Ninguna fuente activa importa o despacha LiteLLM o `copilot-vscode`.
- La validación MCP conserva el control de paths dentro del proyecto y usa los campos existentes del contexto/protocolo.
- La generación Visual Studio usa sólo su contrato público actual.
- Pruebas focalizadas de sync/workspace adapters/process surfaces/MCP pasan sin regresiones.

## Decisión

Los doce errores son residuos de contratos ya retirados o tipos desalineados, no evidencia para revertir R-07. Se corrigen por eliminación/alineación mínima y no se añade una ACC nueva.

## Validación

Se retiraron del mapa activo `GENERATED_FILES_BY_TARGET` las entradas residuales `copilot-vscode` y `litellm`, junto con los comentarios que las describían como targets activos. El registro conserva sin cambios los ocho targets vigentes: `opencode`, `claude-code`, `antigravity`, `visual-studio-2026`, `cursor`, `github-copilot`, `vscode` y `codex`.

Se añadió una prueba focalizada que fija este contrato: los dos identificadores retirados no se exponen y la lista de targets vigentes permanece intacta.

- `./node_modules/.bin/vitest.cmd run packages/installer/tests/installer.test.ts`: correcto — 1 archivo y 17 pruebas pasaron.
- `npm run build`: correcto — `tsc -b` y la copia de `workflows.yaml` finalizaron con exit 0.

No se modificaron los artefactos estructurales del incremento ni se realizó transición de estado, cierre o archive.

## Evidencia final de cierre

La verificación independiente eliminó además una mención textual residual de `copilot-vscode` en el comentario vivo del registry; no quedan LiteLLM ni el alias retirado como entrada o claim activo. La prueba de installer pasó `2/2` archivos y `22/22` tests; la evidencia global actual es build, suite `330/330` y `3289/3289`, doctor PASS `45/60` (0 fallos) y readiness PASS. Receipt Core de verify: SHA-256 `e3107f7a7ee34dec6eac29a340e4459512afe6150e169b848868cf4ce0de377e`.
## Cierre Core

El Core archivó este incremento en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-10-19.634Z-increment-archive-success.json` lo confirma. La cifra `3289/3289` anterior es una fotografía previa: la validación global final es `330/330` archivos y `3294/3294` tests, con build, doctor y readiness PASS.