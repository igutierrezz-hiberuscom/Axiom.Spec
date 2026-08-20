# Context

## Propósito

Conservar las fuentes puntuales usadas para especificar y verificar ACC-041;
el conocimiento estable consolidado vive en `Axiom.Spec/specs/04..07` y
`Axiom.Spec/context/architecture/03-ciclo-de-vida-cli-y-orquestacion.md`.

## Qué puede vivir aquí

- Referencias de implementación: `Axiom/packages/workflow/src/governed-transition-runner.ts`, los callers CLI de lifecycle, `Axiom/packages/mcp-tools/src/transition-handlers.ts` y los bridges `app-api.ts`/`app-launcher.ts`.
- Evidencia de pruebas dirigida y de revisión independiente asociada a este
  incremento.

## Qué no debe vivir aquí

No duplica la spec canónica, no contiene metadata estructural, índices ni una
historia detallada de ejecución; tampoco sustituye los receipts emitidos por el
runtime.

## Estructura sugerida

Si se necesita evidencia adicional, añadir un fichero descriptivo y estable
con fuente, fecha y alcance. La estructura/status del incremento se gestiona
sólo mediante Axiom/Core.
