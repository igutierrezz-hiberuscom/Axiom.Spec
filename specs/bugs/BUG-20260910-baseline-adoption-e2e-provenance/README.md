# baseline adoption e2e provenance and eject convergence

> **Código**: BUG-20260910-baseline-adoption-e2e-provenance
> **Estado**: Reportado
> **Fecha de reporte**: 2026-09-10
> **Severidad**: (pendiente de completar)
> **Prioridad**: (pendiente de completar)

## Resumen del defecto

Los flujos E2E de adoption, provenance, eject y worktree no completan el
encadenamiento esperado desde un proyecto legacy hasta el repo Axiom nuevo.
Son fallos de integración que deben conservar la evidencia original aunque
compartan síntomas con setup.

## Contexto conocido

Fallaron `adopt-creates-axiom-repo.e2e.test.ts`, `adopt-upgrade.e2e.test.ts`,
`axiom-role-worktree.e2e.test.ts`, `cross-cutting-batch.e2e.test.ts`,
`eject.e2e.test.ts`, `north-star-bundle.e2e.test.ts`,
`provenance-lifecycle-manifest.e2e.test.ts` y `workspace-adopt.e2e.test.ts`.

## Clasificación funcional

Convergencia E2E de adoption/provenance/eject/worktree; puede depender de
setup, pero requiere validación propia y no se atribuye automáticamente a una
causa preexistente.

## Comportamiento actual

Los comandos encadenados terminan con exit 1 o dejan resultados incompletos:
repo Axiom no creado, init gate rechazado, estado/worktree ausente o
provenance/manifest no observable.

## Comportamiento esperado

Adoption deja las fuentes legacy byte-identical, crea el destino Axiom válido,
registra provenance, permite upgrade/eject y mantiene idempotencia; los flujos
worktree cierran sin pérdida y preservan evidencia.

## Reproducción

(pendiente de completar)

### Precondiciones

(pendiente de completar)

### Pasos

(pendiente de completar)

### Resultado observado

Aserciones `expected 1 to be +0`, campos `undefined` y errores de setup de repo
axiom en los E2E citados.

## Superficie de regresión

Handlers de adoption/eject, manifest de provenance, workspace/worktree,
implementation context y los wrappers CLI que encadenan esos flujos.

## Estructura mínima del bug

Separar el arreglo de los contratos unitarios de setup cuando sea posible; cada
E2E debe comprobar fuentes legacy, destino, registry y cleanup.

## Material adicional

Log: `%TEMP%\\axiom-r13-baseline-20260910.log`.

La lista completa de suites fallidas está en la sección `file-level failures`
del log.

## Trazabilidad y fuentes

Baseline global del runtime Axiom del 2026-09-10. No se mezcla con el runner,
journal, locks ni recovery de R13-3.

## Estado de validación humana

Pendiente: reproducir primero el happy path de adoption, después upgrade/eject,
provenance y worktree, registrando qué fallos son downstream de setup.
