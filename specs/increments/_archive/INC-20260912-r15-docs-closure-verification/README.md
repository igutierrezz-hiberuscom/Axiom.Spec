# r15 docs closure verification

> **Código**: INC-20260912-r15-docs-closure-verification
> **Estado**: Closed
> **Fecha de creación**: 2026-09-12
> **Tipo de cambio**: nueva verificación de cierre
> **Acción de origen**: `ACC-088`
> **Plan**: `PLAN-INC-20260912-r15-documentation-single-source` (posición 5 de 6)

## Resumen

Convertir la verificación documental en un paso obligatorio y comprobable del cierre de un incremento o bug, en lugar de una obligación redactada en prosa. Al integrar, la operación recorre la documentación afectada, declara qué se revisó, qué se actualizó y qué quedó igual con su motivo, y avisa o falla de forma accionable cuando algo quedó sin revisar.

## Contexto y motivación

Verificado el 2026-09-12: el cierre documental hoy solo existe como instrucción en dos habilidades del catálogo. `axiom-role-close-doc` cubre el contexto técnico por rol y declara expresamente que no toca la especificación funcional; `axiom-spec-integrator` consolida conocimiento durable en la especificación canónica y archiva. Entre las dos no aparecen los manuales, los README, las plantillas, los índices, las decisiones ni los planes.

El camino ejecutable no comprueba documentación: `apps/cli/src/commands/integrate.ts` selecciona la transición declarada de archive, fija `status: archived` e `integration.status: 'integrated'`, mueve la carpeta a `_archive/` y emite receipts. `axiom.config/workflows.yaml` declara como único efecto el `status` en `metadata.yml` con `requiresApproval: true`. `validate-changes` valida alcance de escritura y legalidad de transición, no frescura documental. Y ninguna prueba, chequeo de doctor ni script valida `Axiom/docs/**`.

Consecuencia observada en esta misma auditoría: la documentación se desalineó del producto en cinco frentes distintos sin que nada fallara. Sin esta verificación, las correcciones de los incrementos 1 a 4 volverán a degradarse.

## Alcance

### Incluido

- Una verificación ejecutable en el cierre e integración de incrementos y bugs, que recorra el alcance documental completo: especificación canónica, manual único de producto, contexto técnico, decisiones, planes y README de runtime y de paquetes.
- Salida explícita y auditable: qué documento entró en el alcance, qué se actualizó, qué quedó sin cambios y con qué motivo.
- Comportamiento accionable ante documentación no revisada: el mensaje nombra el documento y la razón por la que entró en el alcance.
- Integración en el ejecutor de transiciones gobernado existente, sin abrir un segundo camino de archivado paralelo.
- Progresión declarada entre aviso y bloqueo, para que la verificación pueda entrar sin congelar el trabajo en curso y endurecerse después.

### Excluido

- Escribir documentación. Esta verificación comprueba, no redacta.
- Redefinir el contrato de QA ni la semántica de archive. Se consume lo que fijen `ACC-041`, `ACC-043` y `ACC-045`.
- Convertir la verificación en una vía para editar estado a mano o saltarse gates de aprobación.
- Sustituir la revisión humana ni la revisión independiente: es una comprobación de cobertura documental, no de calidad de contenido.

## Documentos del incremento

- `01_Requisitos.md`: qué debe comprobar y cómo debe comportarse.
- `02_Cambios_Modelo.md`: dónde se engancha y qué contratos toca.
- `03_Criterios_Aceptacion.md`: criterios verificables.
- `04_Interacciones_UI.md`: qué ve el operador al cerrar.

## Dudas abiertas

Se cierran dentro de este incremento, porque ninguna afecta a los incrementos anteriores. Las decisiones Core son `DEC-20260913-095637-nqjpm1` (D-02), `DEC-20260913-095637-20w16g` (D-03) y `DEC-20260913-095638-x5idar` (D-04), enlazadas a este incremento y al plan R-15.

- **D-02 alcance por tipo de cambio**: cómo se determina qué documentación entra en el alcance de un cambio concreto, para no exigir revisar todo en cada cierre ni dejar fuera lo afectado.
- **D-03 forma de la declaración**: cómo se declara que un documento fue revisado, de modo que sea auditable y no un simple flag de confianza.
- **D-04 aviso frente a bloqueo**: en qué condiciones avisa y en qué condiciones falla, y cómo se hace la transición de uno a otro sin bloquear el trabajo en vuelo.

## Decisiones de ejecución

- **D-02, alcance:** el gate deriva documentos desde el artefacto cerrado, su
	spec/README/criterios y los paths documentales realmente afectados. No
	recorre todo el repositorio por defecto; un alcance vacío se declara como
	`no-documentation-impact`.
- **D-03, declaración:** el resultado estructurado enumera cada documento con
	`updated`, `unchanged` y motivo, o `unreviewed` y razón accionable. CLI,
	launcher y MCP consumen el mismo resultado, que se resume en el receipt de
	la transición sin editar contenido documental.
- **D-04, progresión:** `warning` es el modo compatible por defecto; informa
	los documentos no revisados y permite continuar. `block` se activa de forma
	explícita y rechaza el cierre solo cuando queda un documento en alcance como
	`unreviewed`; ninguno de los modos cambia la legalidad ni la confirmación de
	la transición.

## Decisiones funcionales cerradas

- La verificación es ejecutable, no una obligación redactada en una habilidad.
- Se engancha en el cierre e integración, no en un comando suelto que se pueda olvidar.
- No abre un segundo camino de archivado.

## Consolidación en la spec general

Conocimiento estable a integrar: el cierre de un artefacto incluye una verificación documental comprobable, con alcance declarado y salida auditable. Afecta a `specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`, a `specs/07_Gobierno_y_Seguridad.md` y al contexto técnico de ciclo de vida.

## Estrategia E2E

- Cierre con documentación afectada sin revisar: la verificación lo detecta y nombra el documento.
- Cierre con toda la documentación revisada: la verificación pasa y su salida queda registrada.
- Cambio que no afecta a documentación: la verificación no exige nada y lo declara.
- Modo aviso y modo bloqueo, con la transición entre ambos probada.
- Ninguna transición ilegal pasa a ser legal por el cambio, y ningún camino permite editar estado a mano.
- `npm run build`, suites de workflow y de cli-commands, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` en verde.

## Trazabilidad y fuentes

- Acción `ACC-088` y sesión R-15 del 2026-09-12 en `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md`.
- Evidencia verificada: `apps/cli/src/commands/integrate.ts`, `axiom.config/workflows.yaml`, `packages/cli-commands/src/commands/validate-changes.ts`, `Axiom/axiom.spec/target-axiom-skills/{axiom-role-close-doc,axiom-spec-integrator}.md`, ausencia de cobertura de `Axiom/docs/**` en pruebas y chequeos.
- Acciones relacionadas: `ACC-041`, `ACC-043`, `ACC-045` de R-10, que este incremento consume sin redefinir.

## Estado de validación humana

OK. La re-review independiente final confirmó el alcance causal, la entrada
estructurada en CLI/launcher/MCP y la ausencia de una ruta paralela de cierre.
El derivador automático se limita al artefacto y su plan; los paths externos se
declaran con `affectedPaths` explícitos.

## Notas de implementación

- La verificación se implementa como un único primitivo de `@axiom/workflow` y
	se evalúa dentro de `runGovernedTransition` antes de cualquier escritura.
- La entrada estructurada puede declarar el conjunto documental y el modo
	(`warning` por defecto o `block`). Sin entrada, el runner deriva los archivos
	documentales del artefacto y el plan asociado; los paths externos se pasan
	como `affectedPaths`; un conjunto vacío se registra como
	`no-documentation-impact`.
- La salida enumera cada documento como `updated`, `unchanged` o `unreviewed`,
	siempre con motivo cuando corresponde. CLI, launcher y MCP reciben la misma
	decisión del runner; no se modifica contenido documental. CLI e `integrate`
	aceptan `--documentation-review <json>`; las acciones de archive del launcher
	aceptan la misma declaración y `documentationMode`, y `--json` expone el
	resultado documental junto al registro de workflow.

## Validación

- `npx vitest run packages/workflow/tests/governed-transition-runner.test.ts apps/cli/tests/integrate.test.ts packages/mcp-tools/tests/transition-handlers.test.ts`: PASS, 51 tests.
- `npx vitest run packages/workflow/tests/governed-transition-runner.test.ts apps/cli/tests/integrate.test.ts apps/cli/tests/axiom-increment.test.ts apps/cli/tests/axiom-bug.test.ts packages/mcp-tools/tests/transition-handlers.test.ts`: PASS, 75 tests antes de la ampliación pública.
- `npx vitest run packages/workflow/tests/governed-transition-runner.test.ts`: PASS, 34 tests después de ampliar D-02 y el parser.
- `npx vitest run apps/cli/tests/app-launcher.test.ts`: PASS, 78 tests con transporte launcher y declaración estructurada.
- Ejecución focal conjunta final: 6 archivos, 155/155 tests PASS.
- `npx vitest run apps/cli/tests/docs-command-coverage.test.ts`: PASS, 5/5; las opciones públicas del gate están documentadas.
- `npx vitest run packages/workflow/tests/governed-transition-runner.test.ts`: PASS, 34/34 tras acotar el derivador automático.
- `npm run typecheck`: PASS.
- `npm run build`: PASS.
- `npm run doctor`: PASS, 0 fallos; 2 advertencias y 11 checks omitidos preexistentes.
- `npm run readiness:first-project`: PASS.
- `git diff --check`: PASS.

La suite global `npm test` no se usa como evidencia de cierre porque quedó
repitiendo `apps/cli/tests/workspace-step-reconciliation.test.ts` en una
ejecución previa; se detuvo sin transición parcial y queda como advertencia
de validación del worktree, no como regresión de ACC-088.

## Resultado

Implementación completada. El runner común, la derivación causal de alcance,
el parser JSON y la propagación launcher/CLI/MCP están implementados; los
blockers de la review inicial quedaron resueltos y la re-review final fue OK.
Queda un warning no bloqueante sobre la suite global no concluyente. El
receipt final, freeze y archive se completaron mediante Core; el archive usó
la declaración documental explícita en modo `block`, con 11 documentos
`updated`/`unchanged` y ninguno `unreviewed`.

## Integración de spec general

No se actualizan las specs canónicas generales ni el contexto desde este worker;
el orquestador consolidará únicamente conocimiento estable tras la revisión.
