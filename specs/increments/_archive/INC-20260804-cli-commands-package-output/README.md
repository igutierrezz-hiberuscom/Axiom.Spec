# Increment: Salida convencional de `@axiom/cli-commands`

Status: closed
Date: 2026-08-04
Action: ACC-004 / R-01
Dependencies: none

## Goal

Normalizar el paquete `@axiom/cli-commands` para que su API publica se
compile desde una raiz de paquete coherente y se consuma desde
`packages/cli-commands/dist/index.js` y `dist/index.d.ts`, sin cambiar la
semantica de los comandos compartidos ni romper el ownership unico de
compilacion.

## Context

La auditoria R-01 verifico que el paquete usa `rootDir: "../.."` porque su
barrel reexporta fuentes ubicadas en `apps/cli/src/commands/`. El build actual
emite rutas anidadas bajo `dist/packages/...` y `dist/apps/...`, mientras que
`package.json` publica esas rutas internas como `main` y `types`. Forzar
`rootDir: "src"` sin reubicar las fuentes produce `TS6059`. La TUI declara el
mismo tipo de salida anidada aunque sus fuentes ya viven localmente.

El cambio debe conservar la regla de un solo compilador para cada comando.
`apps/cli` debe seguir importando la superficie publica del paquete y nunca
compilar ni cargar fuentes del paquete mediante rutas relativas.

## Scope

- Reorganizar el ownership fisico de los modulos compartidos necesarios para
  que `@axiom/cli-commands` pueda usar una raiz local, o establecer una
  solucion equivalente que no duplique fuentes ni logica.
- Ajustar `tsconfig`, `package.json`, project references, aliases y exports de
  `@axiom/cli-commands`.
- Actualizar los consumidores reales (`apps/cli`, `@axiom/tui`,
  `@axiom/mcp-tools` y cualquier otro encontrado por busqueda) para usar el
  entrypoint publicado.
- Normalizar la salida de `@axiom/tui` al mismo criterio cuando no implique
  mover logica de negocio.
- Limpiar documentacion que afirme rutas `dist/packages/...` como API publica.
- Añadir o adaptar pruebas de build, resolucion runtime y single ownership.

## Non-goals

- No rediseñar el API de los comandos ni cambiar sus resultados.
- No eliminar comandos, la TUI ni adapters; ACC-005 es posterior.
- No copiar fuentes para satisfacer a TypeScript ni mantener dos owners de
  compilacion.
- No cambiar la topologia de repositorios ni introducir un bundler nuevo.

## Acceptance criteria

- [x] El build de `@axiom/cli-commands` genera `packages/cli-commands/dist/index.js`
      y `dist/index.d.ts` como entrypoints publicos reales.
- [x] El paquete no necesita `rootDir: "../.."` ni incluye fuentes externas a
      su raiz, salvo una justificacion documentada y equivalente que mantenga
      una unica salida publica convencional.
- [x] `apps/cli` y los consumidores del paquete resuelven el entrypoint
      publicado en runtime; no existen imports relativos a `packages/*/src`.
- [x] La salida de `@axiom/tui` queda alineada con `main`/`types` y su build no
      reintroduce rutas anidadas publicas.
- [x] `npm run build` pasa desde `Axiom/` y la CLI compilada puede cargar
      `--help`, `configure`, `sync`, `mcp` y los comandos compartidos.
- [x] Las pruebas dirigidas de CLI, TUI y MCP pasan y comprueban que no hay
      doble compilacion ni `MODULE_NOT_FOUND` en ejecucion.
- [x] La documentacion de instalacion y empaquetado ya no presenta las rutas
      internas anidadas como contrato publico.
- [x] Freeze valido posterior a la implementacion y receipt `verify` con hash
  recomputable.
- [x] Review independiente registrada; los hallazgos de runtime quedaron
  resueltos y los dos diagnosticos del editor fuera del diff se mantienen
  como preexistentes.

## Risks

- Mover fuentes puede romper imports relativos o project references que hoy
  solo funcionan porque comparten una raiz artificial.
- Un build verde de TypeScript no garantiza que Node encuentre los modulos en
  `dist`; se exige una comprobacion runtime sobre la CLI construida.
- Cambiar la salida de TUI puede afectar tests o consumidores que aun usan el
  path historico; deben migrarse, no dejar aliases opacos.

## Open questions

No hay preguntas bloqueantes. La decision de ubicacion fisica se resolvera
priorizando ownership unico y el minimo movimiento de codigo.

## Assumptions

- El contrato publico del paquete es su `main`/`types`, no las rutas internas
  emitidas por el compilador.
- Las fuentes de negocio pueden permanecer bajo `apps/cli` solo si el paquete
  no vuelve a depender de una raiz compartida; en caso contrario se moveran
  al owner que corresponda.
- Los artefactos `dist/` son generados e ignorados y no se deben versionar.

## Implementation notes

La estrategia elegida es trasladar al owner fisico
`Axiom/packages/cli-commands/src/commands/` los modulos que el barrel ya
compila en exclusiva: `_shared`, `_spec-scope`, `configure`, `sync`,
`upgrade`, `rollback`, `workspace-adapter-templates`, `model`, `components`,
`index-cmd`, `validate-changes`, `repair`, `toolchain` y `mcp`. El barrel
mantendra las mismas exportaciones desde rutas locales, `apps/cli` conservara
solo los comandos app-side y los tests que ejercian esos modulos pasaran por
el entrypoint publico del paquete. No se copiaran fuentes ni se anadira un
bundler.

La salida publica de `@axiom/cli-commands` se fija en `dist/index.js` y
`dist/index.d.ts`; la salida histórica de `@axiom/tui` quedó normalizada antes
de ACC-005, que retiró su interfaz y paquete. Los consumidores usan referencias
de proyecto y el package entrypoint, no aliases de TypeScript hacia `src`.

El hash del candidato previo
`d398882fa7da98a61186757b1aa5f6e7020d6a5acc467f488f5be79746934830` queda
como evidencia histórica de la revisión anterior. Esta última versión del
README es la entrada que debe cubrir el freeze final y su receipt `verify`
antes del archivado físico; el hash previo no se usa como evidencia de cierre.

## Validation

Ejecutado desde `Axiom/`:

- `npm run build`: pasa (`tsc -b`).
- `npm run build --workspace @axiom/cli-commands`: pasa con output limpio en
  `packages/cli-commands/dist/index.js` y `dist/index.d.ts`.
- La validación histórica de TUI pasó antes de ACC-005; después de la retirada
  no existe workspace `@axiom/tui` que compilar.
- `npm run build --workspace @axiom/mcp-tools` y
  `npm run build --workspace @axiom/cli`: pasan.
- Tests dirigidos de comandos compartidos: 7 archivos, 67 tests, todos pasan.
- Tests de TUI y MCP: 23 archivos, 268 tests, todos pasan.
- Suite de CLI: 133 archivos, 1268 tests, todos pasan con exit code 0.
- Bateria combinada de consumidores: 31 archivos, 402 tests, todos pasan.
- `get_errors` mantiene solo dos diagnosticos preexistentes en
  `apps/cli/tests/e2e/cross-cutting-batch.e2e.test.ts`; las lineas no forman
  parte del cambio y el E2E correspondiente pasa.
- Runtime: `node apps/cli/dist/index.js --help`, `configure --help`,
  `sync --help`, `mcp --help` y `model --help` cargan correctamente sin
  `MODULE_NOT_FOUND`.
- Busqueda estatica: no quedan imports relativos app-side hacia los comandos
  trasladados ni contratos `dist/packages/*` o `dist/apps/*` en el alcance de
  implementacion.
- `npm install --package-lock-only --ignore-scripts`: pasa y actualiza el
  lockfile para reflejar las dependencias del paquete trasladado.
- `git diff --check` sobre el README actualizado: pasa.

## Result

Implementacion completada. Se trasladaron los 14 modulos compartidos al
owner fisico `packages/cli-commands/src/commands/`, se normalizaron los
entrypoints de `cli-commands` y TUI, se retiraron aliases de TypeScript de los
consumidores y se migraron los tests directos al barrel publico. Se
preservaron los cambios locales previos de `sync.ts` y
`workspace-adapter-templates.ts` durante el traslado.

El incremento queda `closed`: la review independiente, la integración
canónica y la preparación del archivado están completadas. El orquestador
debe conservar el freeze final y el receipt `verify` emitidos sobre esta
versión junto al incremento archivado. Los cambios locales previos
preservados en `sync.ts` y `workspace-adapter-templates.ts` quedan
identificados para mantener su procedencia.

## General spec integration

Integrado en la pasada unica del lote: `specs/00..08` conserva el contrato de
entrypoints y ownership unico; `context/architecture/03-ciclo-de-vida-cli-y-
orquestacion.md` y `context/references/01-inventario-de-packages.md` reflejan
la salida publicada y que TUI fue retirada posteriormente por ACC-005.
