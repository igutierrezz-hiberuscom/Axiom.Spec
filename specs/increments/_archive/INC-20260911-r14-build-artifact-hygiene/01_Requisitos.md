# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos para el saneamiento de artefactos compilados versionados, la actualización de reglas de exclusión en `.gitignore`, la refinación de `TC-009` y el establecimiento de un gate anti-regresión (`ACC-084`).

## Requisitos del incremento

- **REQ-084-01: Retirada de artefactos compilados bajo `src/`**
  Eliminar del árbol los 19 artefactos compilados versionados dentro de `src/`:
  - 16 ficheros en `packages/model-routing/src/`: `{assignments,mutate,slots,types}.{js,js.map,d.ts,d.ts.map}`.
  - 3 ficheros en `apps/cli/src/commands/`: `{mcp,repair,toolchain}.d.ts.map`.

- **REQ-084-02: Retirada de artefactos compilados bajo `packages/adapters/*/dist/`**
  Desversionar los 148 archivos compilados bajo las rutas anidadas `packages/adapters/*/dist/`.

- **REQ-084-03: Extensión de `.gitignore`**
  Actualizar `Axiom/.gitignore` para excluir:
  - Salidas anidadas: `packages/*/*/dist/` y `packages/*/*/*.tsbuildinfo`.
  - Emisiones accidentales dentro de `src/`: extensiones `.js`, `.d.ts`, `.map` bajo cualquier subdirectorio `src/` de `packages/` o `apps/`.

- **REQ-084-04: Refinamiento de `TC-009` (`runAdapterRuntimeCoverageCheck`)**
  Refinar la comprobación para que:
  - Distinga entre paquete de adapter ausente (`src/generator.ts` ausente) $\rightarrow$ `FAIL`.
  - Adapter presente pero no compilado (`src/generator.ts` presente pero `dist/index.js` ausente) $\rightarrow$ Señal honesta `WARN` ("no construido; ejecute npm run build") sin bloquear clones limpios no compilados.

- **REQ-084-05: Gate anti-regresión**
  Crear una prueba automatizada (`apps/cli/tests/artifact-hygiene.test.ts`) que verifique que no existen archivos `.js`, `.d.ts` o `.map` bajo carpetas `src/`, ni carpetas `dist/` anidadas que queden desprotegidas por la exclusión.

## Reglas de negocio relevantes

- No se altera la lógica de generación de los adapters.
- La compilación estándar `npm run build` sigue emitiendo en las carpetas `dist/` esperadas.
