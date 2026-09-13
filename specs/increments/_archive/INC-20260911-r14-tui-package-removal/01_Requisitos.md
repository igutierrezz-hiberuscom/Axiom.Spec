# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos para la retirada física completa y definitiva del paquete residual `@axiom/tui` (`ACC-081`).

## Requisitos del incremento

- **REQ-081-01: Eliminación física de `packages/tui/`**
  Eliminar del repositorio el directorio `Axiom/packages/tui/` completo, incluyendo `package.json`, `tsconfig.json` y `src/flows/preview.ts`.

- **REQ-081-02: Retirada de junctions y enlaces en `node_modules`**
  Eliminar el junction `node_modules/@axiom/tui` generado en el monorepo.

- **REQ-081-03: Saneamiento del lockfile**
  Regenerar `Axiom/package-lock.json` de modo que la entrada de workspace `packages/tui` y sus dependencias internas sean completamente purgadas.

- **REQ-081-04: Prueba positiva de ausencia física**
  Actualizar `apps/cli/tests/tui-retirement.test.ts` para verificar positivamente en disco que `packages/tui` y `node_modules/@axiom/tui` no existen, y que el CLI rechaza cualquier invocación al comando retirado.

## Reglas de negocio relevantes

- No se reabre el alcance funcional de `ACC-005` (la TUI como interfaz ya fue retirada); solo se cierra su residuo físico.
- Se conserva `docs/cli/tui.md` con su condición explícita de documento histórico.

## Fuera de alcance funcional

- Reintroducir previews en terminal: las únicas interfaces interactivas vigentes son las provistas por `axiom app` (launcher web local).
