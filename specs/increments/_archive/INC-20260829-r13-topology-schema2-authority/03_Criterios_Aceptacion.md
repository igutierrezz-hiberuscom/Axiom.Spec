# 03 Criterios de Aceptación

## Happy path

- **CA-B1 / ACC-057**: manifests single y multi-repo schema 2 cargan sin campos v1; todos los productores emiten solo schema 2.
- **CA-B2 / ACC-058**: axiomRepo, codeRepos, legacyRepos, roles y assignments válidos producen `ok:true` sin warnings legacy.
- **CA-B3 / ACC-059**: desde axiomRepo y code repo se lee el mismo archivo autoral y una mutación de roles produce el mismo resultado observable.
- **CA-B4**: el dogfood valida con Axiom.Spec autoridad, Axiom code y Axiom.SDD legacy read-only.

## Negativos

Fallan con finding/error tipado: schema 1/futuro, unknown keys, axiomRepo ausente/duplicado/wrong kind, code wrong kind o sin ownership, legacy wrong mode/function/duplicado, ID/ref vacío, IDs o paths colisionantes, role duplicado, assignment ghost/unknown role, cero o múltiples primary por code repo, múltiples primary por role y mode incoherente.

## Ausencia de mutación

Ningún manifest inválido se escribe. Un code repo no puede convertir su proyección local en autoridad ni modificar un legacy repo.

## Evidencia

Suites de `@axiom/topology`, CLI topology/roles, doctor, MCP, freeze/knowledge/write-scope, workspace fixtures schema 2, build y diff-check.
