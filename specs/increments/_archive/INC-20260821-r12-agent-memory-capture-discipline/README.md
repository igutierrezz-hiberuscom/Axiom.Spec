# r12-agent-memory-capture-discipline

> **Código**: INC-20260821-r12-agent-memory-capture-discipline
> **Estado**: Archivado
> **Fecha de creación**: 2026-08-21
> **Tipo de cambio**: disciplina explícita de agentes y metadata de memoria

## Resumen

Eliminar la ambigüedad sobre decisiones y bugs: ningún runtime los guarda automáticamente. Las skills Kiro de incremento y bug deben ejecutar explícitamente `axiom memory add` al confirmar una decisión o un bug, usando una visibilidad y evidencia adecuadas.

## Contexto y motivación

La política histórica de curación describía decisiones y bugs como auto-persistidos, pero la ruta real exige una llamada de guardado. Esta contradicción impide saber quién es responsable de capturar conocimiento. Además, `private` no es una prohibición de lectura local: solo evita que Knowledge Sync exporte la memoria por Git; por ello la skill debe elegir expresamente qué compartir con el equipo.

## Alcance

### Incluido

- Retirar del runtime los claims y exports de auto/manual-persistencia que sugieren un automatismo inexistente.
- Extender `axiom memory add` con `--visibility project-shared|private`, `--rationale` y `--source`, manteniendo sus defaults actuales cuando la evidencia no se proporciona.
- Preservar la lectura local de entradas `private`; solo `project-shared` explícito es elegible para Knowledge Sync.
- Añadir a las skills y agentes Kiro `axiom-increment` y `axiom-bug` una acción obligatoria de captura explícita para decisiones confirmadas y bugs confirmados.
- Documentar en esas skills qué conocimiento se comparte y qué se mantiene local, y prohibir secretos en ambas visibilidades.
- Añadir pruebas dirigidas de options, metadata y validación de visibilidad.

### Excluido

- No se infieren ni guardan decisiones/bugs automáticamente desde audit, workflow, launcher o runtime.
- No se cambia el mecanismo de Git de Knowledge Sync ni se ejecuta sync/push desde las skills.
- No se convierte `private` en control de acceso local ni se permite guardar secretos, tokens, credenciales o PII.
- No se migra el backend; Engram obligatorio es un incremento posterior.
- No se modifican las copias de workflow de otros harnesses ni el catálogo bundleado de skills que no contiene `axiom-increment`/`axiom-bug`.

## Documentos del incremento

- `01_Requisitos.md`: responsabilidades y reglas observables.
- `02_Cambios_Modelo.md`: options y política retirada.
- `03_Criterios_Aceptacion.md`: pruebas de CLI y disciplina documental.
- `04_Interacciones_UI.md`: ayuda, errores y comandos de captura.

## Dudas abiertas

Ninguna. La captura es obligación de la skill, no una inferencia de runtime.

## Decisiones funcionales cerradas

- Una decisión o bug solo se considera capturado si la skill ejecuta `axiom memory add` y comunica el resultado.
- `project-shared` se usa para decisiones, bugs, restricciones o patrones confirmados y útiles al equipo; requiere contenido libre de secretos.
- `private` conserva contexto local, hipótesis o notas temporales no confirmadas y nunca sale por Knowledge Sync, pero sigue siendo legible localmente y no protege secretos.
- Si el guardado falla, el agente registra el error y no afirma una captura inexistente; no usa fallback automático.

## Consolidación en la spec general

Al cerrar, reconciliar los claims activos de `specs/00..08` y el manual de skills sobre captura explícita, visibilidad y ausencia de automatismo. El contexto técnico se modifica solo si hay conocimiento estable verificable de la integración de runtime.

## Estrategia E2E

Comprobar que `memory add` persiste metadata explícita, rechaza visibilidad inválida, conserva `private` localmente y que Knowledge Sync solo considera exportable `project-shared`. Revisar las cuatro superficies Kiro afectadas para verificar la acción y las reglas de clasificación.

## Trazabilidad y fuentes

- `Axiom/apps/cli/src/commands/memory.ts`
- `Axiom/apps/cli/tests/{memory,knowledge-sync}.test.ts`
- `Axiom/packages/memory/src/{types,index}.ts`
- `Axiom/apps/cli/src/commands/knowledge-sync.ts`
- `Axiom.SDD/.kiro/{skills,agents}/axiom-{increment,bug}*`

## Estado de validación humana

Implementación terminada y evidencia de cierre reunida para el archive gobernado por Axiom CLI.

- Las skills Kiro `axiom-increment` y `axiom-bug` exigen ejecutar de forma explícita `axiom memory add` al confirmar una decisión o bug, y comunicar el id o el error. La captura no se infiere desde audit, workflow, launcher ni runtime.
- `memory add` conserva la lectura local de `private`, limita Knowledge Sync a `project-shared` explícito y registra `visibility`, `rationale` y `source`. `private` no es control de acceso y ninguna visibilidad permite secretos, tokens, credenciales, PII ni datos sensibles.
- `--manual` fue retirado de la CLI: invocar `axiom memory add` ya es una acción explícita. La retirada de `axiom learn` no eliminó el valor semántico general `MemoryKind: 'learning'`, que queda fuera de este alcance.
- La captura explícita de la evidencia de bug del lote se comprobó con `axiom memory add`; el id devuelto fue `mt4025vu-rf1br5n7`.
- La validación dirigida compartida del lote pasó: `npx vitest run packages/providers/tests/stdio-mcp-client.test.ts packages/memory/tests/memory.test.ts packages/memory/tests/evidence.test.ts packages/memory/tests/engram-backend.test.ts apps/cli/tests/memory.test.ts apps/cli/tests/knowledge-sync.test.ts apps/cli/tests/freeze.test.ts packages/doctor/tests/memory.test.ts packages/mcp-tools/tests/registry.test.ts packages/mcp-server/tests/server.test.ts apps/cli/tests/mcp-serve.test.ts --reporter=dot` ejecutó 94 pruebas correctas; `npm run build` pasó; el smoke `node apps/cli/dist/index.js memory query --query "axiom" --limit 1 --path "C:\\repos\\Axiom Workspace\\Axiom"` pasó; `npm run doctor` terminó con `46/61 OK, 0 fail, 4 warnings, 11 skipped`; y `git diff --check` terminó con código 0 (solo avisos CRLF).

## Revisión contra criterios de aceptación

La revisión independiente confirmó que no quedan APIs ni claims runtime de auto-persistencia, que `private` sigue legible localmente pero no exportable por Knowledge Sync y que solo `project-shared` explícito es exportable. Las opciones y metadatos de la CLI están cubiertos por pruebas dirigidas. No se detectaron regresiones nuevas; el riesgo de una prueba E2E compuesta queda cubierto funcionalmente por las suites focalizadas. No se aplicaron mutaciones Git.

## Integración documental y contexto técnico

La reconciliación final ya actualizó los claims activos de `specs/00_Resumen_Ejecutivo.md`, `01_Requisitos_Funcionales.md`, `03_Modelo_Operativo_y_Datos.md`, `05_Interfaces_Operativas.md`, `06_Integraciones_y_Capacidades.md`, `07_Gobierno_y_Seguridad.md`, `08_Glosario.md`, `specs/manuales/04_Generar_Spec_y_Contexto_Tecnico.md` y `specs/manuales/13_Skills_Agentes_y_Roles.md`. También se actualizaron `context/architecture/03-ciclo-de-vida-cli-y-orquestacion.md`, `context/integrations/01-capabilities-providers-y-toolchain.md` y `context/operations/02-doctor-troubleshooting-y-telemetria.md` con conocimiento estable sobre captura explícita, visibilidad y ausencia de automatismo. No se creó documentación técnica ni índices nuevos; cualquier índice derivado se valida o regenera exclusivamente mediante Axiom CLI.
