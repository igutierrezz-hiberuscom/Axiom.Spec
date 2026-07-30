# INC-20260730-phase-receipts: Receipt Phase

## Goal
Asegurar el gobierno verificable mediante la emisión de "receipts" (recibos) inmutables en JSON al finalizar cada fase SDD.

## Scope
- Implementar mecanismo de escritura de receipts JSON para las fases del ciclo de vida SDD (design, tasks, apply, verify, knowledge, freeze, close).
- Los receipts se escriben en la carpeta `receipts/` dentro del directorio del incremento (`specs/increments/<id>/receipts/`).
- El receipt debe indicar el estado final de la fase (`success` o `failure`), el timestamp, los inputs principales o hash, y cualquier metadato relevante de la fase.

## Non-goals
- No modificar la manera en la que se ejecutan las fases, solo agregar la escritura del receipt al final.
- No reescribir historial pasado.

## Acceptance Criteria
- Las funciones de ciclo de vida en `packages/workflow` (o el equivalente en el CLI de phases) emiten un receipt JSON validado al final.
- Si una fase falla, emite un receipt marcando el error.
- Todos los tests relevantes pasan.
