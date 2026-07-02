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

1. `context/`: conocimiento técnico estable y compartido.
2. `specs/`: spec general numerada y artefactos funcionales.
3. `decisions/`: ADR y decisiones estructurales.
4. `artifacts/`: material de apoyo, análisis y entregables no canónicos.
5. `prompts/`: prompts de especificación o refinamiento documental.

## Relación con los otros repos

1. `Axiom/` implementa el runtime.
2. `Axiom.SDD/` contiene agents, skills e instrucciones para trabajar sobre Axiom.
3. este repo no sustituye el código ni el workflow operativo; los gobierna documentalmente.