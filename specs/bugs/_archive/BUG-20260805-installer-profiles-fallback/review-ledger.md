# Review ledger: BUG-20260805-installer-profiles-fallback

- Fecha de revision: 2026-08-05
- Alcance: README y metadata del bug, diff de Axiom, loader/installer/tests, `@axiom/install-profiles`, consumidores CLI y validaciones focalizadas.
- Metodo: revision independiente en modo lectura; no se hicieron cambios de codigo ni Git mutante.
- Recomendación histórica: `pending`

## Cumplimiento observado

| Criterio | Evidencia | Estado |
| --- | --- | --- |
| Archivo ausente conserva `DEFAULT_PROFILES` | `packages/installer/tests/installer.test.ts`: escenario 6 retorna `ok` y cubre los 10 targets canonicos | verificado |
| Archivo presente ilegible devuelve error | Test con `profiles.yaml` como directorio devuelve `invalid-profiles-yaml` y mensaje de lectura | verificado |
| YAML invalido devuelve error | Test focalizado devuelve `invalid-profiles-yaml` con mensaje `YAML invalido` | verificado |
| Estructura invalida devuelve error antes de componer | `InstallProfilesYamlSchema.safeParse` se ejecuta en `profiles-loader.ts`; el test comprueba `defaultMutationScope` | verificado |
| YAML personalizado gana al fallback | El fixture personalizado omite `builder`; el resultado es `composition-failed` en lugar del `ok` que produciria `DEFAULT_PROFILES` | verificado |
| No se escribe `profiles.yaml` | El camino de fallback no crea `axiom.config/profiles.yaml`; la unica persistencia del instalador es `install-profile.json` | verificado por codigo y test |
| Consumidores CLI | `roles.test.ts`, `topology.test.ts`, `context.test.ts`, `workspace-adapters.test.ts` y `start.test.ts` pasan | verificado |

## Hallazgos de la revisión inicial (históricos)

| id | lens | location | severity | status | evidence |
| --- | --- | --- | --- | --- | --- |
| REVIEW-001 | review | `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md:55` | WARNING | open | La afirmacion activa dice que `installProfile` usa el archivo si es legible y que "si no" cae a `DEFAULT_PROFILES`; no limita el fallback a ausencia y puede conservar la interpretacion antigua para un archivo presente ilegible o invalido. Debe explicitarse: solo ausencia hace fallback; lectura, parseo o schema invalidos devuelven error. |
| REVIEW-002 | review | `Axiom.Spec/specs/bugs/BUG-20260805-installer-profiles-fallback/README.md:3` | WARNING | open | El bug sigue `in-progress`, los siete criterios permanecen sin marcar y las secciones `Validation` y `Result` siguen en `Pendiente`; `metadata.yml:4` tambien conserva `status: in-progress`. La evidencia de tests ya existe, pero el artefacto no documenta el cierre. |
| REVIEW-003 | review | `Axiom/packages/installer/src/types.ts:64` | SUGGESTION | open | La documentacion de `invalid-profiles-yaml` solo menciona lectura o parseo, aunque el loader ahora devuelve el mismo kind para errores de schema. Actualizar el contrato del paquete y su README para incluir validacion estructural. |
| REVIEW-004 | review | `Axiom/packages/installer/tests/installer.test.ts:355` | SUGGESTION | open | La prueba de prioridad personalizada demuestra el override mediante un fallo de composicion, pero no compara bytes antes/despues de un `profiles.yaml` existente. La implementacion no lo escribe; una asercion de inmutabilidad para un archivo presente reduciria el riesgo de regresion. |

## Riesgos residuales

- `installer.ts` decide el fallback con un segundo `fs.existsSync` despues del fallo de `readFileSync`. En condiciones normales cubre el caso requerido y todos los tests pasan, pero una carrera, un enlace simbolico colgante o un directorio padre inaccesible podria parecer ausencia. Preservar el codigo `ENOENT` del primer error seria mas determinista en una revision futura.
- La prueba de archivo ilegible usa un directorio con nombre `profiles.yaml`, una simulacion portatil. No hay una prueba de ACL/permisos de lectura real, por lo que ese caso depende de la semantica del sistema operativo.

## Validacion observada

- `npx vitest run packages/installer/tests/installer.test.ts apps/cli/tests/roles.test.ts apps/cli/tests/topology.test.ts --reporter=verbose`: 3 archivos, 37 tests, todo verde.
- `npx vitest run packages/install-profiles/tests/default-profiles.test.ts packages/install-profiles/tests/composer.test.ts packages/config-validation/tests/validator.test.ts --reporter=verbose`: 3 archivos, 73 tests, todo verde.
- `npx vitest run apps/cli/tests/context.test.ts apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/start.test.ts --reporter=dot`: 3 archivos, 27 tests, todo verde; solo warnings esperados del entorno de fixtures.
- `get_errors` no reporto errores en los seis archivos fuente/test revisados.
- `git diff --check` no reporto errores en el diff focalizado.
- El contexto de sesion registra `npm run build` en `Axiom` con exit code 0.

## Acciones documentales para el orquestador

- Actualizar la afirmacion de `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md:55` para separar explicitamente ausencia de archivo frente a archivo presente invalido.
- Completar primero el README del bug y despues `metadata.yml`: marcar criterios satisfechos, registrar validacion y resultado, y cambiar el estado solo cuando se confirme el cierre.
- No se requiere integrar cambios en `specs/00..08` adicionales ni en `context/**` aparte de la aclaracion puntual anterior; no se recomienda copiar la historia de implementacion a la especificacion canonica.

## Recomendación histórica de cierre

`pending`: el comportamiento implementado y las validaciones observadas cumplen los criterios funcionales, pero el artefacto del bug no ha registrado la evidencia ni el resultado, y la especificacion canonica conserva una formulacion ambigua sobre el fallback. Resolver `REVIEW-001` y `REVIEW-002` antes de marcar `closed`; `REVIEW-003` y `REVIEW-004` son mejoras no bloqueantes.

## Commit sugerido

`fix(installer): fail closed for present profiles.yaml errors`

## Resolución posterior del orquestador

Date: 2026-08-05

| id | status | resolución |
| --- | --- | --- |
| REVIEW-001 | deferred | El runtime ya limita el fallback a archivo ausente. La frase activa de `specs/03_Modelo_Operativo_y_Datos.md` se corregirá durante la integración única del autopilot. |
| REVIEW-002 | resolved | README, criterios, validación, resultado y metadata del bug quedaron actualizados; el estado permanece `pending` solo por la integración canónica. |
| REVIEW-003 | resolved | `InstallerError` y `packages/installer/README.md` documentan que `invalid-profiles-yaml` incluye lectura, parseo y validación estructural. |
| REVIEW-004 | resolved | La prueba del override compara el contenido de `profiles.yaml` antes y después de `installProfile`. |

## Estado actual

La implementación y las pruebas funcionales están verificadas. La afirmación
canónica sobre el fallback quedó reconciliada en `specs/03` y el contexto
operativo; el bug se cierra y se archiva.

Receipt `verify` final: `receipts/2026-08-05T16-05-39.521Z-verify-success.json`.
