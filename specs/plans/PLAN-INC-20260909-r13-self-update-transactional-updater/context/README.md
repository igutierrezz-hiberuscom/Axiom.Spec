# Context

## Propósito

Definir el espacio de evidencia de ejecución del plan del incremento 3/5. Su función es ayudar al builder y al reviewer a conectar tareas, código real y resultados reproducibles sin convertir este directorio en una spec paralela ni afirmar que el trabajo ya está implementado.

## Qué puede vivir aquí

Solo evidencia producida durante la ejecución autorizada del plan y vinculada a un commit:

- mapeo B0 entre rutas/símbolos previstos y el layout real del runtime Axiom;
- baseline de build/tests y clasificación de fallos preexistentes;
- matriz de fault injection por frontera con outcome y snapshots;
- matriz Windows/POSIX de canonicalización y activación atómica;
- descripción del harness de repos Git locales y dos updaters;
- evidencia sanitizada de rollback, recovery idempotente y retry;
- review contra ACC-078/079, blast radius y rollback de implementación;
- handoff de API para incremento 4 y evidence packet para incremento 5.

Cada evidencia debe indicar comando, SO, commit, fecha, criterio/tarea y resultado. Los logs deben estar recortados y sanitizados.

## Qué no debe vivir aquí

- Metadata, `plan.metadata.yml`, status, receipts, índices o transiciones gestionadas por Core.
- Código, binaries, `node_modules`, clones Git, staging, locks o journals reales del updater.
- Tokens, credenciales, URLs autenticadas, variables sensibles o logs completos.
- Copias de requisitos/criterios ya canónicos o narrativa final para usuarios.
- UI del Launcher 4, artifacts del pipeline/docs 5 o trabajo de instalación inicial.
- Instrucciones de borrado manual, stash/reset/clean o recuperación no verificada.

## Estructura sugerida

Si el flujo Axiom autoriza persistir evidencia, mantener un conjunto mínimo:

- `implementation-map.md`: mapping B0 y blast radius real;
- `validation-matrix.md`: comandos, SO, Git states y resultados;
- `fault-boundaries.md`: punto inyectado, journal, outcome y estado pre/post;
- `review-and-handoff.md`: findings resueltos, decisión GO/STOP y contratos para 4/5.

Los repos Git, candidates y journals usados por tests viven en directorios temporales y no se copian aquí. Un resultado estable se integra después en la spec general propietaria mediante Axiom/Core; la evidencia efímera se conserva solo si aporta auditabilidad y no contiene información sensible.
