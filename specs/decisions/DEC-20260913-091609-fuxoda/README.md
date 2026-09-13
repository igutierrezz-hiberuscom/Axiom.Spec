<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0032-axiom-spec-boundary-and-runtime-baseline.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# ADR-0032 — Boundary entre `Axiom.Spec/` y `Axiom/axiom.spec/`

> **Estado**: accepted, verificado el 2026-08-03 como resultado de ACC-003 de R-00.

Contenido preservado del documento legacy `decisions/0032-axiom-spec-boundary-and-runtime-baseline.md`.

## Decisión

`Axiom.Spec/` es el repositorio canónico de especificación del workspace; `Axiom/axiom.spec/` es contenido versionado product-owned y baseline de materialización para proyectos adoptantes. No se fusionan ni se mueven sus cinco familias (`increments`, `plans`, `target-axiom-agents`, `target-axiom-skills`, `templates`).

Los comandos que resuelven un spec repo dedicado operan en `specs/increments/` y las demás raíces canónicas de ese repositorio; el runtime single-repo puede seguir usando `axiom.spec`. La documentación debe conservar ambas grafías y sus responsabilidades distintas.

## Consecuencias

Se mantiene la separación de ownership y no se modifica runtime, catálogos ni el contenido de `Axiom/axiom.spec/`. La resolución del choque con `ADR-0032-toolchain-versioning` se registra en INC-20260912-r15: este documento histórico tiene el ID gestionado `DEC-20260913-091609-fuxoda`, mientras el ADR de toolchain conserva su ID distinto.
