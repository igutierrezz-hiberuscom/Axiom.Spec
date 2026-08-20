# 01 Requisitos

1. El contenido mínimo del plan se valida una vez en la operación canónica.
2. `axiom-plan approve` usa `runGovernedTransition` y exige `--confirm`.
3. Launcher previsualiza y confirma la misma operación; no reproduce reglas de contenido.
4. `axiom-role start` bloquea todo estado distinto de `plan-approved`.
5. No se crean estados adicionales ni transiciones alternativas.
