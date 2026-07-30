# INC-20260730-candidate-freeze: Freeze de Candidate

## Goal
Implementar un comando `axiom freeze` global que congele todo el estado de un candidate/incremento (memoria local, repo specs, dependencias) para asegurar que el comando `apply` sea determinista.

## Scope
- Nuevo comando CLI: `axiom freeze --increment <id>`
- Genera un archivo `.frozen.json` (o similar) en el directorio del incremento con un snapshot hash.
- Modificar la fase de apply (o el CLI en general) para que antes de aplicar cambios (e.g. en axiom-autopilot o subagentes), verifique que el candidate está frozen y que los inputs no mutaron.

## Non-goals
- No congela estado de repos externos no vinculados.
- No reemplaza al `knowledge freeze` que es específico para la fase de harvest.

## Acceptance Criteria
- Ejecutar `axiom freeze --increment <id>` genera un archivo JSON inmutable en el incremento.
- Se proporciona una API para validar el freeze.
