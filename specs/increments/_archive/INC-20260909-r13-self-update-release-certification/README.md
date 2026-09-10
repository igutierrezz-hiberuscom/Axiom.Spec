# r13-self-update-release-certification

> **Código**: INC-20260909-r13-self-update-release-certification
> **Estado**: En Borrador
> **Fecha de creación**: 2026-09-09
> **Tipo de cambio**: certificación final de release, pruebas E2E y alineación documental

## Resumen

Este incremento es el quinto y último del lote R-13.5 para ACC-080. Define el gate que acepta o rechaza una release Git del monorepo completo y certifica el recorrido real del único CLI global: release A → descubrimiento/check → plan → apply de release B → verificación de versión, manifest y provenance → rollback o recovery ante fallo.

La secuencia y la propiedad de contratos son estrictas:

1. **identity** crea la autoridad única de versión y resuelve remoto, tag y commit;
2. **state/CLI** proyecta esa identidad en inventario, manifest, operaciones, JSON, locks y persistencia;
3. **updater/helper** implementa plan, apply, staging, activación, rollback y recovery;
4. **Launcher** consume el mismo contrato y gobierna confirmación, progreso y reinicio;
5. **certification** valida las cuatro entregas juntas y proyecta su identidad en evidencia, sin crear otra identidad ni migrar consumidores.

Este incremento no modifica qué módulo posee la versión ni reescribe consumidores para que adopten esa autoridad. Si una proyección —CLI compilada, workspace, manifest, Launcher u otro consumidor— diverge, el gate emite STOP y devuelve el defecto al incremento 1 o al predecesor propietario. La certificación puede leer y comparar la autoridad entregada por 1/5; no puede sustituirla con un literal, un módulo generado alternativo, el manifest instalado o un valor del job.

La unidad entregable es un commit exacto alcanzado por un **tag anotado** del remoto canónico con nombre completo `refs/tags/v<SemVer>`. Un tag lightweight se rechaza aunque su nombre y commit coincidan. No se certifican ramas, refs ambiguas, un tarball aislado de `@axiom/cli`, un package npm standalone ni una TUI.

## Contexto y motivación

Las pruebas unitarias de los incrementos 1–4 no demuestran por sí solas que un clean clone pueda instalarse, compilarse, activar el shim/binlink correcto ni pasar de A a B. Tampoco demuestran que la autoridad entregada por 1/5, el tag anotado, el commit, la CLI ejecutada y la provenance instalada sigan concordando en Windows y Linux. ACC-080 aporta esa demostración final sin absorber la implementación de sus predecesores.

## Alcance

### Incluido

- Validación read-only de la identidad heredada de 1/5:
  - `productVersion` es SemVer estricto sin `v` y procede de la autoridad única creada por identity;
  - `releaseRef` es exactamente `refs/tags/v<productVersion>`;
  - la ref remota apunta a un objeto Git de tipo `tag` anotado;
  - `releaseCommit` es el object ID completo del commit obtenido al hacer peel del tag;
  - versión, ref, commit y todas las proyecciones observables concuerdan.
- Rechazo explícito de tags lightweight, ramas, refs abreviadas, targets libres y valores de cache/manifest usados como autoridad.
- Contrato inicial de toolchain:
  - `engines.node`: `>=20.14.0 <23`;
  - `engines.npm`: `>=10.7.0 <11`;
  - celdas mínimas en Windows y Linux para Node 20.14.0/npm 10.7.0 y Node 22.14.0/npm 10.9.2.
- Guard de distribución que conserva `private: true` y rechaza npm/tarball como canal de release.
- Gate reproducible sobre clean clone con `npm ci`, typecheck, build, baseline completa, suites focales, installer, `--version`, `--help` y doctor.
- Nuevos scripts, tests y workflow de certificación como **entregables de 5/5**, no como precondiciones que deban existir al iniciar.
- E2E hermético A→B con remoto bare local, tags anotados de fixture, homes/cache/prefix/PATH temporales y ejecución por entry absoluto y shim/binlink.
- Happy path, no-op, matriz de plataformas/toolchain y fault injection; rollback completo o `recovery-required` cuando no pueda demostrarse restauración.
- Paridad observable CLI/Launcher sobre el mismo plan, confirmación, outcome, provenance y recuperación.
- Evidence ledger, revisión independiente y decisión STOP/GO.
- Reconciliación documental únicamente después de GO técnico.
- Integración de conocimiento estable y operaciones de cierre únicamente mediante el flujo autorizado de Axiom Core.

### Excluido

- Crear otra autoridad de versión, mover la autoridad creada por 1/5 o migrar consumidores a ella.
- Reimplementar resolución SemVer/remoto/cache de identity, manifest/CLI, updater/helper o Launcher.
- Corregir dentro del diff self-update fallos preexistentes y ajenos detectados por la baseline completa.
- Publicar en npm, retirar `private: true`, construir un package/tarball standalone o diseñar un canal de packages.
- Crear releases o tags upstream; los tags A/B del E2E son locales, anotados y efímeros.
- Incorporar una TUI, auto-update al arranque, actualización silenciosa o bypass de confirmación.
- Ejecutar transiciones lifecycle o editar metadata, status, receipts, enlaces, índices o carpetas manualmente.

## Contratos cerrados

### Identidad y tags

Solo puede certificar una release una ref completa `refs/tags/v<SemVer>` obtenida del remoto canónico cuyo objeto sea un tag anotado y cuyo peel resuelva al commit candidato. El validator inspecciona el tipo del objeto antes de hacer peel. Un objeto `commit` alcanzado directamente por la ref identifica un tag lightweight y produce rechazo sin build, publicación ni activación.

Los fixtures positivos A/B se crean con tags anotados. La suite negativa incluye al menos un tag lightweight con nombre SemVer válido para demostrar que el nombre correcto no basta.

### Node/npm

| Plataforma | Node | npm | Carácter de la celda |
|---|---:|---:|---|
| Windows | 20.14.0 | 10.7.0 | límite inferior obligatorio |
| Windows | 22.14.0 | 10.9.2 | versión reciente obligatoria |
| Linux | 20.14.0 | 10.7.0 | límite inferior obligatorio |
| Linux | 22.14.0 | 10.9.2 | versión reciente obligatoria |

Los rangos normativos son `engines.node >=20.14.0 <23` y `engines.npm >=10.7.0 <11`. Las cuatro celdas deben registrar versiones efectivas y pasar el mismo gate obligatorio. Si una falla, el resultado es STOP. No se estrecha, amplía, intercambia ni omite una celda en scripts o CI para obtener verde: cualquier cambio del contrato exige primero un cambio explícito de spec revisado.

### Baseline completa

Antes del diff de 5/5 se captura y ejecuta la validación completa ya existente. La ausencia inicial de los nuevos scripts/tests/workflow de release no es un fallo de baseline porque son entregables de este incremento.

Si la baseline existente está roja:

1. se conserva la evidencia del fallo;
2. se clasifica como defecto de un predecesor o bug ajeno;
3. se asigna al owner correspondiente y se resuelve en un cambio separado del diff self-update;
4. se vuelve a ejecutar la baseline completa hasta obtener verde;
5. solo entonces puede alcanzarse el gate técnico final.

No se permite ocultar el rojo con skips, exclusiones, cambios de expectativas, reintentos sin causa o reclasificación a warning. La clasificación tampoco autoriza ampliar 5/5 para corregir bugs no relacionados.

## Entregables de certificación

El baseline observado ya dispone de comandos como `npm run typecheck`, `npm run build`, `npm test`, `npm run readiness:first-project` y el comando focal `node --test scripts/install-global.test.mjs`. El plan deberá reconfirmarlos antes de usarlos.

Los scripts raíz de agregación/certificación, los tests del validator y E2E, el harness clean-clone y el workflow Windows/Linux son entregables nuevos. No pueden usarse como condiciones de entrada ni afirmarse existentes antes de su implementación. Su contrato exacto y la distinción entre comandos existentes y futuros están en el plan asociado.

## Reconciliación documental posterior a GO técnico

El GO técnico habilita, pero no sustituye, la fase documental. Después de ese GO se reconciliarán al menos:

- claims runtime: `README.md`, `docs/README.md`, `docs/installation.md`, `docs/overview.md` y `docs/cli/README.md`;
- guía nueva: `docs/cli/self-update.md`;
- claims relacionados cuando entren en conflicto: `docs/cli/tui.md` y `docs/cli/doctor.md`;
- manual canónico: `specs/manuales/03_Actualizar_Versiones.md`;
- conocimiento estable propietario en `specs/00..08` y `context/`, sin copiar logs ni historial efímero.

La documentación no puede adelantarse al runtime, presentar TUI/npm package como rutas vigentes ni usar tags de fixture como releases reales. Tras el diff documental se repiten las validaciones aplicables antes del review final.

## Documentos del incremento

- `README.md`: frontera, secuencia y contratos cerrados.
- `01_Requisitos.md`: requisitos exigibles del gate.
- `02_Cambios_Modelo.md`: vistas heredadas, evidencia y estados observables.
- `03_Criterios_Aceptacion.md`: condiciones de STOP/GO.
- `04_Interacciones_UI.md`: proyección coherente en CLI y Launcher.
- `context/README.md`: material auxiliar permitido.
- Plan asociado: `PLAN-INC-20260909-r13-self-update-release-certification`.

## Decisiones funcionales cerradas

- Hay un solo producto, una sola autoridad de versión creada por 1/5 y un solo CLI user-level.
- 5/5 no migra consumidores ni crea identidad, manifest, updater o runner alternativos.
- La release es Git y abarca el monorepo completo.
- Solo tags anotados `refs/tags/v<SemVer>` pueden certificar; lightweight se rechazan.
- El toolchain inicial y sus cuatro celdas mínimas son los valores literales indicados arriba.
- Un fallo de celda exige STOP y cambio explícito de spec; nunca ajuste silencioso.
- `private: true` permanece; npm es toolchain, no canal de distribución.
- El E2E no usa red Git externa ni home, prefix, cache o PATH reales.
- CLI y Launcher consumen los mismos servicios y outcomes; la TUI no participa.
- La documentación se modifica solo después del GO técnico.
- Baseline roja, ledger incompleto, rollback no demostrado o review con blockers impiden el GO final.

No quedan decisiones abiertas sobre identidad, tipo de tag, rangos Node/npm, celdas mínimas, secuencia o frontera de responsabilidad.

## Estrategia E2E y rollback

Cada escenario crea dentro de un temp root un remoto bare local, productor, instalación A, candidato B, staging, `HOME`, `USERPROFILE`, npm prefix/cache, temporales y PATH controlado. A y B usan versiones SemVer exclusivas y tags anotados; un fixture lightweight separado cubre el rechazo.

El happy path ejecuta A, descubre B, produce un plan inmutable, aplica B con updater/helper real, activa el shim/binlink y verifica por ruta absoluta y por launcher efectivo: versión B, ayuda, doctor, commit/ref, manifest y provenance. B→B debe devolver `unchanged`/`noop` sin fases mutantes ni falsa instalación.

La fault matrix cubre fetch, identidad, lock, `npm ci`, build, smokes, activación, persistencia, interrupción, rollback y cleanup. Antes de activar, A queda intacta. Después de activar, el sistema restaura completamente A o devuelve `recovery-required` con evidencia y `recover` idempotente. Nunca se declara `installed` por intención ni se elimina evidencia necesaria para recuperación.

## Estado de validación

Esta redacción no afirma ejecuciones, tags upstream, commits candidatos, PASS, receipts ni cierre. El incremento solo podrá alcanzar GO técnico con baseline completa verde, handoffs 1→4 válidos, entregables nuevos implementados, cuatro celdas Node/npm verdes y E2E/faults reproducibles. El GO documental requiere la reconciliación posterior; el GO final requiere ledger completo, review independiente, integración estable y operaciones estructurales aceptadas por Core.