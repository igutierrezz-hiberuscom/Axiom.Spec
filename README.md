# Axiom.Spec

Este repo es la fuente de verdad documental de Axiom.

## Propósito

Aquí vive la definición del producto y de su modelo operativo:

1. spec general;
2. incrementos y bugs;
3. decisiones arquitectónicas;
4. contexto técnico estable;
5. artefactos y prompts de apoyo.

## Reglas de ownership

1. este repo no contiene runtime del producto;
2. este repo no contiene agents ni skills operativas del workflow SDD;
3. cualquier comportamiento de Axiom debe describirse aquí antes de consolidarse en runtime.

## Estructura

1. `specs/`: especificación canónica numerada y artefactos funcionales; incluye `specs/increments/`, `specs/bugs/`, `specs/decisions/`, `specs/adr/`, `specs/plans/` y `specs/archive/`.
2. `context/`: conocimiento técnico estable y compartido.
3. `technical-context/`: índices derivados del contexto técnico; las fuentes narrativas viven en `context/`.
4. `plans/`: planes de implementación y coordinación entre repositorios.
5. `prompts/`: prompts de especificación o refinamiento documental.
6. `specs/decisions/`: única raíz activa de decisiones gestionadas; `specs/adr/` conserva artefactos ADR gestionados con IDs propios.

Las antiguas raíces top-level `bugs/`, `increments/` y `decisions/` no forman parte del contrato activo. Los nombres históricos solo aparecen en trazabilidad explícita.

## Relación con los otros repos

1. `Axiom/` implementa el runtime.
2. `Axiom.SDD/` contiene agents, skills e instrucciones para trabajar sobre Axiom.
3. este repo no sustituye el código ni el workflow operativo; los gobierna documentalmente.