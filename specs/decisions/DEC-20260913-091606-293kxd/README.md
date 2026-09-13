<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0028-workflow-ux-and-archive-safety-completion.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0028 — Workflow UX and Archive Safety Completion

> **Estado**: archived 2026-06-30

Contenido preservado del documento legacy `decisions/0028-workflow-ux-and-archive-safety-completion.md`.

## Resumen ejecutivo

Se completó la ergonomía de comandos de workflow, el gate visible de archivado con QA paralelo y la consolidación del parser compartido de workflows.

## Lotes

- Los tres intents declarados funcionan como chain wrappers.
- El archive-gate advierte cuando `qaLane=parallel` y QA está pendiente, sin bloquear.
- `loadWorkflowsConfig` centraliza parsing tolerante con errores tipados.

## Métricas

**1015/1015 tests verde**, **`tsc -b` verde** y GATE 0028 honrada.
