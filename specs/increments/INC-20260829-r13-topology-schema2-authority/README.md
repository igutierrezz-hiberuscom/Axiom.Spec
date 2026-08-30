# Topología schema 2 y autoridad única R-13

> **Código**: INC-20260829-r13-topology-schema2-authority
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-057..ACC-059

## Objetivo

Eliminar el contrato topológico v1, completar las invariantes de schema 2 y establecer un único manifest autoral en `axiomRepo`.

## Revalidación

El 2026-08-29 `Axiom/axiom.config/topology.yaml` sigue en schema 1; `@axiom/topology` normaliza `sddRepo/specRepo/roleCodeRepositories`, refleja ambos shapes y emite `deprecated-legacy-shape`. Roles y setup escriben copias locales. Ningún cambio local previo resuelve estas acciones.

## Alcance

- ACC-057: solo `schemaVersion: 2`, con `axiomRepo`, `codeRepos`, `legacyRepos`, roles, assignments y QA lane.
- ACC-058: validator semántico único y findings tipados para todas las invariantes.
- ACC-059: manifest autoral solo en `axiomRepo`; `axiom.yaml` de code repos conserva identidad y puntero mínimo; roles y lectores convergen en la autoridad.
- Actualización de productores/consumidores runtime, dogfood, tests y docs operativas activas.
- Proyecciones locales solo si son necesarias, marcadas como derivadas y nunca aceptadas como autoridad.

## No objetivos

- Bindings, envelopes CLI y writer concurrente completo pertenecen a C.
- Resolución sin catálogo y superficie incremental pertenecen a D.
- Transacción estructural multiarchivo y preservación seccional de `axiom.yaml` pertenecen a E.
- No se migra schema 1 ni se soporta una base instalada inexistente.
- No se implementa R-13.5/self-update.

## Decisiones cerradas

1. `TopologyManifest` público y persistido es exclusivamente schema 2; se eliminan campos espejo y normalizador legacy.
2. `axiomRepo.kind` debe ser `axiom`; `codeRepos[].kind` debe ser `code`; `legacyRepos[].kind/mode` deben ser `legacy/read-only-source` y cada legacy declara `legacyFunction: sdd | spec`.
3. IDs de repos y roles son globalmente únicos; refs no vacíos ni físicamente convergentes.
4. Cada code repo tiene al menos una assignment y exactamente una `primary:true`; un rol tiene como máximo un code repo primario. Assignments solo apuntan a code repos y roles declarados.
5. `single-repo` solo admite `axiomRepo` sin code/legacy repos; cualquier repo adicional exige `multi-repo`.
6. Un topology ausente solo puede sintetizar un default schema 2 en el propio axiomRepo single-repo; desde un code repo sin puntero es error, no fallback.
7. El dogfood de tres repos queda representado sin mover ownership: `Axiom.Spec` es `axiomRepo` y autoridad de conocimiento; `Axiom` es `codeRepo` primario del rol `builder`; `Axiom.SDD` es fuente legacy SDD read-only. No se modifica Axiom.SDD.
8. La copia actual en `Axiom/axiom.config/topology.yaml` deja de ser autoral. Si debe existir transitoriamente, contiene marker/puntero derivado y el loader nunca la acepta frente al manifest de `Axiom.Spec`.
9. Cada `axiom.yaml` de code repo conserva `projectId`, `repoId`, `kind: code` y un puntero canonicalizable a `axiomRepo`; no duplica repos, roles ni assignments.

## Riesgos

- Amplio fan-out de tipos v1 en doctor, MCP, upgrade, freeze, knowledge, planes, launcher y write-scope.
- Bootstrap del dogfood durante la transición de autoridad; los tests deben usar fixtures schema 2 desde el primer commit lógico.
- Proyecciones stale podrían parecer autorales; marker y resolución fail-closed son obligatorios.
- No tocar cambios locales de R-11/R-12 en consumidores compartidos.

## Compatibilidad

Rechazo explícito: schema 1, aliases de campos, normalización y warning de deprecación se retiran. Un manifest v1 devuelve `unsupported-schema-version`/`invalid-manifest` tipado y no se reescribe.

## Validación prevista

Tests de loader/validator por cada invariante; productores y consumidores multi-repo; resolución de autoridad/roles desde repos; dogfood `topology validate`, build, doctor/readiness focalizados y `git diff --check`.

## Integración estable

Diferida al cierre final del lote; el apply no edita `specs/00..08` ni `context/**`.
