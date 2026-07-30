# INC-20260730-knowledge-harvest-freeze

Status: pending
Date: 2026-07-30
Plan: F1.5

## Goal

Endurecimiento del harvest de conocimiento ("Freeze" de entradas). Debemos garantizar que, antes de recolectar conocimiento (harvest), se congelen de forma determinista y verificable sus inputs (tópicos, paths, metadatos y hashes), invalidando y descartando explícitamente a los candidatos "stale" (obsoletos).

## Context

Actualmente en Axiom, los procesos de consolidación de Specs y memoria no protegen formalmente el conjunto de cambios bajo revisión frente a la mutación en curso. Gentle-AI propone el "Candidate freeze", donde un set inmutable asegura que lo que se cosecha coincide 1:1 con lo validado, y no puede ser alterado en background por otro agente/proceso antes de cerrarse el flujo.

## Scope

- Explorar el pipeline actual de conocimiento/harvest (e.g. `packages/knowledge` o adaptadores de `Axiom.SDD` si existen para el comando `axiom knowledge harvest`).
- Asegurar que antes del harvest, todos los inputs sean congelados (topics, paths, hashes).
- En caso de mutación, el candidato es calificado explícitamente como "stale".

## Non-goals

- Cambiar cómo se guardan o versionan los engrams/specs de producto a largo plazo, solo aislar y hacer determinista el snapshot de entrada para el harvest.

## Acceptance criteria

- [ ] Existe un mecanismo que toma el conjunto de paths+hashes y declara el "freeze" del knowledge harvest candidate.
- [ ] La recolección (harvest) rechaza o alerta de cambios detectados contra los hashes congelados (estado stale).
- [ ] Todo sigue el patrón `fail-closed`.
- [ ] Pruebas automatizadas validan la inmutabilidad y detección de los cambios de estado del candidato.

## Validation

- Testing asociado en el package correspondiente.
