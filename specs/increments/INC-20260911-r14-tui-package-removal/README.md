# r14 tui package removal

> **Código**: INC-20260911-r14-tui-package-removal
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-11
> **Tipo de cambio**: eliminación de residuo estructural
> **Acción de origen**: `ACC-081`
> **Plan**: `PLAN-INC-20260911-r14-operator-control-surfaces` (posición 1 de 7)

## Resumen

Retirar físicamente el paquete residual `@axiom/tui`, que dejó de ser una superficie operativa con `ACC-005` pero sigue presente en el árbol, en `node_modules` y en el lockfile.

## Contexto y motivación

`ACC-005` quedó `validado`: el subcomando `axiom tui`, el driver, las pantallas, los flujos y las pruebas dedicadas se retiraron, y `packages/tui/src/index.ts` se borró en `c4df64c`. La auditoría R-14 del 2026-09-11 verificó que el residuo sigue ahí:

- `git ls-files` devuelve `packages/tui/package.json`, `packages/tui/tsconfig.json` y `packages/tui/src/flows/preview.ts`, este último con commit posterior al lote de retirada (`ff25c77`).
- `node_modules/@axiom/tui` existe como junction al paquete.
- `package-lock.json` conserva su entrada de workspace con ocho dependencias internas.
- El paquete no está en las project references de `tsconfig.json` ni en los alias de `vitest.config.ts`, así que hoy no se compila ni se testea.
- `apps/cli/tests/tui-retirement.test.ts` pasa porque `require.resolve('@axiom/tui')` falla al no existir `packages/tui/dist/`, no porque el paquete no exista.

Es residuo inerte, pero mantiene viva una superficie retirada en el manifiesto de dependencias y hace que la prueba de retirada afirme algo distinto de lo que comprueba.

## Alcance

### Incluido

- Eliminación de `Axiom/packages/tui/` completo.
- Salida del workspace `packages/*` y regeneración del `package-lock.json`.
- Retirada del junction `node_modules/@axiom/tui`.
- Barrido de alias, project references, dependencias declaradas e imports activos.
- Sustitución de la aserción indirecta de `tui-retirement.test.ts` por una comprobación positiva de ausencia del paquete y del subcomando.

### Excluido

- Reabrir `ACC-005`: la retirada funcional ya está validada y no se reaudita.
- La retirada documental de la TUI como ruta vigente, que pertenece a `ACC-080`.
- `docs/cli/tui.md`, que se conserva como página explícitamente histórica.
- Reintroducir lógica de preview: las previews vigentes son las del launcher.

## Dudas abiertas

`D-05` del plan: si la regeneración del lockfile se acepta dentro de este incremento y con qué comprobación se sustituye la aserción indirecta de la prueba de retirada. Ambas se resuelven dentro del propio incremento; no dependen de las decisiones transversales de la fase 0.

## Decisiones funcionales cerradas

- El paquete se elimina, no se archiva ni se marca como deprecado.
- La página histórica de documentación se conserva.

## Consolidación en la spec general

Al cierre, reconciliar cualquier afirmación activa que aún trate `@axiom/tui` como paquete existente. Si no queda ninguna, declararlo explícitamente.

## Estrategia E2E

Prueba positiva de ausencia del paquete y del subcomando; `npm run build`, suite completa, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` en verde tras regenerar el lockfile.

## Trazabilidad y fuentes

Auditoría R-14 del 2026-09-11 y acción `ACC-081` en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`; antecedente `ACC-005` y su incremento `INC-20260804-tui-retirement`.

## Estado de validación humana

Pendiente. El incremento está en `specifying` y no se ha implementado nada.
