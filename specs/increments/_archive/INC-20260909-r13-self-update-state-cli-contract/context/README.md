# Context

## Propósito

Reunir contexto local y evidencia de implementación estrictamente necesarios para ACC-079 sin duplicar la especificación canónica. Este incremento es el segundo de la secuencia R13 y depende de `INC-20260909-r13-self-update-release-identity`.

Actualmente este directorio no contiene evidencia adicional. El presente README fija qué podrá incorporarse durante implementación/review; no afirma que el runtime ya cumpla el contrato.

## Qué puede vivir aquí

- Mapa confirmado de archivos y símbolos runtime que implementen reader, migrator, writer, lock y comando CLI.
- Fixtures anonimizados de shapes v1 realmente soportados y su clasificación, si no pertenecen al repositorio de código.
- Matriz de compatibilidad de filesystem por plataforma, especialmente replace/fsync/recovery.
- Evidencia resumida de pruebas de corrupción, concurrencia, no mutación y wrapper compilado.
- Decisiones locales necesarias para interpretar ACC-079, siempre enlazando la fuente canónica y sin redefinirla.
- Notas de handoff al incremento 3 sobre APIs estables y capacidades deliberadamente no implementadas.

## Qué no debe vivir aquí

- `metadata.yml`, índices, receipts de lifecycle o cambios de status gestionados manualmente.
- Copias completas de specs generales, de release identity o del plan.
- Implementación TypeScript, wrappers compilados o artefactos de build.
- Tokens, paths personales, dumps de instalaciones reales, logs con secretos o provenance sensible.
- Diseño o implementación de Git update, Launcher UI o release CI.
- Afirmaciones de «implementado», «validado» o «cerrado» sin evidencia y transición gestionada por Axiom/Core.

## Estructura sugerida

Solo se crearán archivos cuando exista evidencia estable que lo justifique:

| Archivo opcional | Contenido permitido |
|---|---|
| `runtime-map.md` | Paths/símbolos confirmados y ownership por paquete |
| `compatibility-matrix.md` | v1/v2 y garantías por plataforma |
| `validation-evidence.md` | Comandos, resultados y clasificación de fallos |
| `handoff-inc-3.md` | Contratos consumibles por el motor futuro |

La creación o indexación de contexto seguirá las operaciones estructurales de Axiom; este incremento no necesita crear esos documentos durante la fase de especificación.

## Contexto rector

- `install.json` v2 será cache/receipt, nunca autoridad.
- La identidad real provendrá del incremento 1.
- Ausencia no es `0.0.0`.
- v1 solo se migrará bajo una frontera mutante explícita y tras reconciliar la instalación real.
- `status`, `check` y `plan` serán read-only; `apply` y `recover` fallarán cerrados hasta el incremento 3.
- Lock y atomic write reutilizarán `@axiom/core`; una carencia de garantías produce gate STOP, no una implementación paralela.
- Los incrementos 3–5 quedan habilitados por este contrato, no incluidos en su alcance.

## Handoff esperado

Antes de dar GO al incremento 3 deberá existir evidencia de:

1. schema v2 y envelope v1 cerrados y probados;
2. reader y migrator compatibles con fixtures v1 reales;
3. writer durable con lock user-level, temporales únicos y lost-update protection;
4. operaciones read-only demostradas por snapshots de filesystem;
5. `apply/recover` fallando como `engine_unavailable` sin efectos;
6. wrapper compilado preservando streams y exit codes;
7. build y suites relevantes en verde, más review independiente de alcance.
