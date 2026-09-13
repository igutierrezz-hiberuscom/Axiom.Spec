<!-- AXIOM:MIGRATED — source: Axiom.Spec/decisions/0030-operator-app-plugins-and-external-bridge.md; migrated for INC-20260912-r15-canonical-roots-dedup via Core. -->

# Incremento 0030 — Operator App Plugins and External Bridge

> **Estado**: archived 2026-06-30

Contenido preservado del documento legacy `decisions/0030-operator-app-plugins-and-external-bridge.md`.

## Resumen ejecutivo

Se completó el sistema de plugins project-scoped para la app portable y el bridge declarativo de Azure DevOps.

## Lotes

- `loadAppPlugins` descubre `.axiom-state/<project>/app-plugins/*.json`, tolera malformaciones y rechaza duplicados con warning.
- El plugin `azure-devops` declara tabs `work-items`/`mapping` y cuatro acciones, con mutaciones externas protegidas por `confirmed: true`.
- `GET /api/projects/:id/plugins` expone plugins del filesystem y el plugin Azure DevOps por defecto.

## Métricas

**1000/1000 tests verde**, **`tsc -b` verde**, GATE 0025 y GATE 0030 honradas.
