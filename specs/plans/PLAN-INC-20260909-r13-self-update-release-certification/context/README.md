# Context

## Propósito

Esta carpeta contiene, cuando sea necesario, material operativo acotado del plan ACC-080: decisiones previas al código, topología de pruebas, matrices de ejecución y notas para el handoff. No sustituye el plan, la spec del incremento, los receipts de Core ni el contexto técnico canónico.

La redacción inicial no afirma ejecuciones ni resultados. Cada archivo futuro debe distinguir claramente diseño, observación y evidencia, y enlazar un AC-080/E-ID.

## Qué puede vivir aquí

- `dependency-handoff.md`: tabla de contratos recibidos de los incrementos 1–4, rutas, owners y blockers.
- `node-npm-compatibility.md`: probes, decisión exacta de rangos y matriz adoptada.
- `clean-clone-topology.md`: entorno, allowlist y algoritmo de aislamiento.
- `e2e-platform-matrix.md`: partición de escenarios entre Windows/POSIX y versiones Node/npm.
- `fault-injection-map.md`: hook de test, fase durable y outcome esperado F-01..F-18.
- `evidence-ledger.md`: índice legible E-01..E-22 que referencia outputs/receipts gestionados, sin fabricar estados.
- `review-handoff.md`: instrucciones de reproducción y hallazgos del reviewer independiente.
- `canonical-integration-checklist.md`: `specs/00..08/context`, marcado como actualizado o revisado sin cambio después de GO.

Solo se crean los archivos que aporten información real y no quepan de forma clara en el plan o en los tests.

## Qué no debe vivir aquí

- `metadata.yml`, `plan.metadata.yml`, status, enlaces, índices o receipts copiados/editados.
- Clones Git, `node_modules`, `dist`, cache npm, homes, prefixes, shims o backups de E2E.
- Logs completos con datos del host, variables de entorno, tokens o credenciales.
- Tags/commits upstream inventados o resultados PASS no ejecutados.
- Una implementación alternativa de identity, manifest, updater o Launcher.
- Prosa canónica duplicada de `specs/00..08` o manuales runtime.
- Diseño de package npm/tarball standalone o TUI.
- Comandos destructivos, push, publicación o instrucciones para editar lifecycle manualmente.

## Estructura sugerida

```text
context/
  README.md
  dependency-handoff.md
  node-npm-compatibility.md
  clean-clone-topology.md
  e2e-platform-matrix.md
  fault-injection-map.md
  evidence-ledger.md
  review-handoff.md
  canonical-integration-checklist.md
```

La estructura es máxima, no obligatoria. Se prefieren menos archivos con ownership claro. Los resultados ejecutables permanecen en temp roots o en el sistema de evidencia aprobado; el ledger conserva hashes/rutas/referencias, no copias masivas.

Al cerrar el lote, el integrador extrae solo decisiones estables al archivo propietario bajo `Axiom.Spec/context/`. Axiom Core valida/reconstruye índices y archiva artifacts; esta carpeta no gestiona transiciones.