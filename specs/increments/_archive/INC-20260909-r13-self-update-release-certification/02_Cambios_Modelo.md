# 02 Cambios de Modelo

## Objetivo del documento

Definir las vistas y resultados que ACC-080 necesita para certificar una release sin crear un segundo modelo de identidad. La autoridad de versión y `ProductReleaseIdentity` pertenecen a 1/5; inventario/manifest a 2/5; transacción/journal a 3/5; DTO y ciclo de proceso de Launcher a 4/5. 5/5 los consume en modo read-only y añade únicamente evidencia de certificación.

## Modelos heredados, no creados por 5/5

### `ProductReleaseIdentity` — propiedad de identity 1/5

| Campo heredado | Regla que certifica 5/5 |
|---|---|
| `productVersion` | SemVer estricto sin `v`, leído de la autoridad única creada por 1/5. |
| `releaseRef` | Exactamente `refs/tags/v<productVersion>`. |
| `releaseCommit` | Object ID completo del commit obtenido al hacer peel del tag anotado. |
| `canonicalRemote` | Remoto normalizado y autorizado por 1/5. |
| `sourceTree` | Monorepo completo en `releaseCommit`, nunca `apps/cli` aislado. |

ACC-080 no vuelve a declarar ni persistir estos campos como otra identidad. El validator referencia la identidad heredada y compara sus proyecciones. La autoridad versionada sigue siendo la definida por 1/5; si una CLI, workspace, build o manifest no la consume correctamente, 5/5 emite STOP y devuelve el defecto al owner. No edita ese consumidor para hacer pasar el gate.

### Política cerrada del objeto tag

`releaseRef` solo es certificable cuando, después de fetch exacto desde `canonicalRemote`, la ref apunta a un objeto Git de tipo `tag`. El validator inspecciona ese objeto antes de resolver `releaseRef^{}` al commit. Una ref que apunta directamente a `commit` es lightweight y se rechaza.

El tipo de objeto es una condición de validación, no un nuevo campo de identidad ni otra autoridad. El tag anotado A/B del fixture es efímero; no se persiste como claim upstream.

### Proyecciones heredadas

5/5 compara, sin redefinir ownership:

- versión raíz y manifests gobernados entregados por 1/5;
- `axiom --version` y build identity entregados por 1/5;
- inventario, plan, manifest, status y provenance de 2/5;
- target ejecutado, journal y outcome de 3/5;
- DTO, progreso y proceso reiniciado de 4/5.

Una proyección puede aparecer en el ledger como valor observado. Esa copia de evidencia no es estado operativo ni fuente para resolver una release futura.

## Estructuras propias de certificación

### `NodeNpmCompatibility`

Contrato normativo y cerrado:

| Campo | Valor |
|---|---|
| `nodeRange` | `>=20.14.0 <23` |
| `npmRange` | `>=10.7.0 <11` |
| `requiredCells` | Windows y Linux × {Node 20.14.0/npm 10.7.0, Node 22.14.0/npm 10.9.2}. |
| `failurePolicy` | Cualquier celda fallida produce STOP y exige cambio explícito de spec antes de modificar el contrato. |

`@types/node`, `lockfileVersion`, la versión del builder o una etiqueta móvil de CI no satisfacen el contrato. La certificación registra versiones efectivas. No se usa `packageManager` como segunda fuente del rango soportado.

### `ReleaseCertificationResult`

Resultado del validator/gate, separado del lifecycle del incremento:

| Campo | Regla |
|---|---|
| `schemaVersion` | Versión cerrada del formato de evidencia. |
| `candidateRef` | Referencia a la `ProductReleaseIdentity` heredada; no crea una identidad durable. |
| `tagObjectType` | Evidencia observada; debe ser `tag`, nunca `commit`. |
| `peeledCommit` | Commit observado al hacer peel y comparado con `releaseCommit`. |
| `platform` | SO, arquitectura y familia de launcher. |
| `nodeVersion` / `npmVersion` | Valores efectivos de una celda obligatoria. |
| `steps` | Comando/probe, cwd, exit code, duración y referencias a stdout/stderr. |
| `checks` | Resultados por criterio; un fallo obligatorio no puede degradarse a warning. |
| `artifacts` | Hash/ruta relativa de manifest, build identity, launcher y logs autorizados. |
| `outcome` | `passed` o `failed`; no cambia metadata o status lifecycle. |

El resultado redacta paths/datos host innecesarios. Los receipts oficiales, si corresponden, los crea Core; el test no los escribe a mano.

### `BaselineClassification`

Registro de preflight para cada fallo de la baseline completa existente:

| Campo | Regla |
|---|---|
| `command` | Comando existente ejecutado antes del diff. |
| `observedFailure` | Exit code y evidencia preservada, sin ocultar el rojo. |
| `classification` | `predecessor` o `unrelated-existing-bug`; un fallo introducido después se clasifica como `introduced-by-5`. |
| `owner` | Incremento o bug que debe resolverlo. |
| `resolutionChange` | Cambio separado del diff self-update para fallos preexistentes. |
| `rerun` | Ejecución completa posterior; debe quedar verde antes del gate final. |

La ausencia inicial de los nuevos validator, harness, E2E o workflow no genera una entrada roja: esos elementos son entregables de 5/5, no comandos existentes del baseline.

### `PackageDistributionGuard`

Regla de validación, no artefacto publicable:

- raíz y workspaces gobernados conservan `private: true`;
- el gate no usa `npm pack` o `npm publish` como unidad entregable;
- dependencias workspace y assets proceden del clone completo;
- cualquier package standalone se deriva a otro incremento.

### `HermeticReleaseFixture`

Modelo efímero de test:

- temp root;
- remoto bare Git local;
- commits A/B y tags **anotados** `v<SemVer>`;
- tag lightweight negativo separado;
- instalación A, staging B y launcher temporal;
- HOME/USERPROFILE, npm prefix/cache, PATH y temporales aislados;
- fault injector limitado a tests;
- snapshot antes/después para demostrar ausencia de mutación host.

## Modelos heredados de estado y transacción

### `InstalledReleaseProvenance` — propiedad de 2/5

ACC-080 solo comprueba coherencia observable de versión ejecutada, ref, commit, remoto, entry activo, tipo de launcher, versión anterior, transaction ID y timestamps. El writer, schema, lock y migraciones pertenecen a state/CLI. El gate no introduce otro manifest.

### Outcomes y journal — propiedad de 3/5

ACC-080 consume `installed`, `unchanged`, `failed` y `recovery-required` tal como los entrega updater/helper. No añade un quinto estado de actualización ni convierte el resultado `passed` de certificación en outcome transaccional.

### DTO y reinicio — propiedad de 4/5

Launcher proyecta el mismo plan/outcome/provenance que CLI y reinicia después de activación. ACC-080 compara esos observables; no agrega un runner o bus de eventos paralelo.

## Secuencia de estados observables

1. 1/5 produce una identidad con tag anotado, ref y commit verificables.
2. 2/5 representa esa identidad en inventario, plan, manifest y status.
3. 3/5 usa la misma identidad en plan/apply y devuelve un outcome transaccional.
4. 4/5 muestra el mismo DTO y reinicia sin mezclar módulos A/B.
5. 5/5 compara todas las proyecciones y emite `ReleaseCertificationResult`.

Transiciones E2E:

- A instalada → B disponible: check actualiza cache sin activar B.
- B disponible → plan B: read-only e inmutable.
- plan B → B instalada: confirmación, transacción, smokes, activación y persistencia.
- B instalada → B sin cambios: no-op sin fases mutantes.
- fallo preactivación → A verificada: `failed`, sin drift.
- fallo postactivación con rollback → A verificada: `failed`.
- rollback no demostrable → `recovery-required` con journal/backup.
- recover → A o B observada y coherente, de forma idempotente.

## Invariantes

- `releaseRef == refs/tags/v${productVersion}`.
- `objectType(releaseRef) == tag`.
- `peelToCommit(releaseRef) == releaseCommit`.
- Un tag lightweight se rechaza antes de build/activación.
- `rootAuthority == compiledCliVersion == observedInstalledVersion` para una release instalada, como proyecciones de 1/5.
- La provenance describe el commit que ejecuta el launcher, no solo el solicitado.
- `installed` implica smokes y persistencia exitosos.
- `unchanged` implica ausencia de mutación.
- `recovery-required` implica exit no cero y evidencia conservada.
- Un manifest ausente/corrupto no inventa versión ni autoridad.
- `engines.node == ">=20.14.0 <23"` y `engines.npm == ">=10.7.0 <11"`.
- Las cuatro celdas obligatorias deben pasar; un cambio requiere modificar primero la spec.
- Baseline completa roja impide el gate final hasta resolución separada y rerun verde.
- Ningún resultado de certificación promueve por sí mismo el status lifecycle.

## Compatibilidad y evolución

- La migración a la autoridad única es responsabilidad de 1/5. ACC-080 rechaza divergencias; no las repara.
- Tags anotados son obligatorios tanto para candidatos como para fixtures positivos. Lightweight existe solo como caso negativo.
- Los rangos y celdas Node/npm están cerrados. No hay selección pendiente durante implementación.
- Windows y Linux materializan launchers distintos, pero deben observar la misma identidad y outcomes.
- `private: true` es compatible con instalación desde clone Git y no se relaja.
- La evolución de manifest corresponde a 2/5; fault boundaries y recovery a 3/5; reinicio a 4/5.
- Nuevos scripts, tests y workflow pertenecen al entregable de certificación y no se asumen en el baseline inicial.
- Logs, snapshots y resultados son evidencia efímera. Solo reglas estables se integran tras GO en specs, manual y contexto propietarios.