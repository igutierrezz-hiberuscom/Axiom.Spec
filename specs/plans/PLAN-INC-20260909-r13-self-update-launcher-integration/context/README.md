# Context

## Propósito

Este directorio puede conservar contexto operativo y evidencia **locales al plan** de integración del self-update en Launcher. Debe ayudar al builder/reviewer a ejecutar gates STOP/GO, confirmar rutas/símbolos y entregar un handoff reproducible al incremento 5 sin convertir notas de trabajo en especificación canónica.

## Qué puede vivir aquí

- Snapshot fechado de las interfaces realmente entregadas por los incrementos 1–3.
- Mapa de rutas/símbolos confirmados frente a los candidatos del plan.
- Matriz de gates G0–G6 con evidencia enlazada y owner de bloqueos.
- Threat model focal: auth/origin, token binding, race/replay, path/command injection, child lifecycle, bounded progress y restart.
- Diseño de fixtures herméticos para bare remotes, clones, homes, prefixes, ports y helper processes temporales.
- Matriz de pruebas HTTP/control-plane/frontend/helper y snapshots de no-mutación.
- Resúmenes sanitizados de comandos, resultados, plataformas, fallos preexistentes/nuevos y review independiente.
- Estrategia de rollback de la capa Launcher y handoff al incremento 5.

Cada evidencia debe indicar fecha, comando o test fuente, entorno/plataforma, resultado exacto y si se observó sobre implementación real o fixture.

## Qué no debe vivir aquí

- Metadata, `plan.metadata.yml`, status, enlaces estructurales, índices, caches generadas o receipts gestionados por Core.
- Copias completas del incremento, plan integral, ADR, decisiones o specs 00..08.
- Código runtime, scripts de helper, repositorios Git, homes/prefixes de test, binarios o `node_modules`.
- Tokens de confirmación, cookies, credentials, PII, URLs con secretos, stacks o outputs sin sanitizar.
- Logs/eventos sin cota o evidencia de red externa.
- Una implementación alternativa del engine o decisiones propietarias de los incrementos 1–3.
- Afirmaciones de PASS sin comando ejecutado ni afirmaciones de que Launcher ya soporta self-update.
- Release CI, packaging o manuales finales propiedad del incremento 5.

## Estructura sugerida

Solo si hace falta durante la ejecución:

- `dependency-contracts.md`: versión/hashes y compatibilidad de outputs 1–3.
- `runtime-map.md`: paths/símbolos confirmados y diferencias con candidatos.
- `gates.md`: STOP/GO y evidencia G0–G6.
- `security-review.md`: amenazas, mitigaciones y findings.
- `test-matrix.md`: casos, fixtures, comandos y resultados.
- `no-mutation-evidence.md`: snapshots antes/después y efectos permitidos.
- `rollback-and-handoff.md`: rollback de integración y paquete para 5/5.

No se crea jerarquía por defecto. Si un dato ya está estable en su artefacto propietario se enlaza en vez de duplicarlo.

## Criterios de calidad del contexto

1. Distinguir claramente **hecho observado**, **ruta probable**, **supuesto** y **decisión pendiente del owner**.
2. Mantener trazabilidad a ACC-077/078, requisitos y criterios CA-LSI.
3. Usar fixtures locales y declarar explícitamente que no hubo red externa.
4. Separar fallos preexistentes de regresiones introducidas.
5. Registrar limitaciones y plataformas no ejecutadas para que 5/5 pueda completar certificación.
6. Integrar conocimiento estable en el archivo canónico propietario solo por el workflow permitido; no editar índices/status manualmente.
