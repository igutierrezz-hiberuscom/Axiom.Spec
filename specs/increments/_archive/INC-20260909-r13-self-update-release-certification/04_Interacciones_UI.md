# 04 Interacciones UI

## Objetivo del documento

Definir la experiencia observable que ACC-080 certifica en CLI y Launcher sin crear una TUI ni otro flujo. Las operaciones y estados pertenecen a 2/5, el helper transaccional a 3/5 y la integración visual/reinicio a 4/5. 5/5 compara ambas superficies y proyecta en evidencia la identidad heredada de 1/5; no añade comandos de negocio, migra consumidores ni crea identidad alternativa.

## Propiedad de las superficies

### CLI global `axiom` — contrato de 2/5

Cuando el handoff de 2/5 esté implementado, la CLI expone:

- `status`: `publishedVersion`, `downloadedVersion`, `installedVersion`, ref/commit, procedencia, frescura y recovery;
- `check`: refresca inventario best-effort; no aplica ni modifica checkout;
- `plan`: presenta target validado, pasos, impacto global y rollback; es read-only;
- `apply`: requiere plan vigente y confirmación y delega al helper de 3/5;
- `recover`: ejecuta la recuperación permitida cuando existe estado recuperable;
- `--json`: mantiene el envelope y separación stdout/stderr definidos por 2/5;
- `--dry-run`: no alcanza rutas mutantes.

Estas operaciones son precondición de handoff por pertenecer a 2/5, no comandos creados por certification. La sintaxis exacta la fija 2/5. 5/5 no acepta un `--target-version` libre como autoridad.

### Updater/helper — contrato de 3/5

El helper externo posee preflight, staging, dependencias/build, verify, activate, persist, cleanup, rollback y recover. CLI y Launcher delegan en él; la UI no reimplementa fases ni convierte un fallo en warning.

### Launcher — contrato de 4/5

- Consume el mismo inventario y plan que CLI.
- Puede iniciar check no bloqueante, nunca apply al arrancar.
- Muestra identidad, frescura, impacto global y confirmación.
- Lanza el mismo helper y presenta progreso acotado.
- Cierra/reinicia después de activar B para no mezclar módulos A/B.
- Prioriza recovery cuando el helper devuelve `recovery-required`.

### Certification — responsabilidad de 5/5

El gate compara outputs normalizados, targets ejecutados, outcomes y reinicio. Solo añade harnesses, tests y evidencia. Los scripts/tests/workflow nuevos de certificación no son superficies del usuario ni se presuponen existentes antes de implementar 5/5.

## Identidad visible

La identidad mostrada procede de 1/5. Para una release certificable:

- versión visible: SemVer sin `v`;
- ref visible: `refs/tags/v<SemVer>`;
- ref remota: objeto de tag anotado;
- commit visible: commit completo obtenido al hacer peel;
- manifest/status/Launcher: proyecciones concordantes.

Un tag lightweight se muestra como release inválida y no habilita plan/apply. El tipo de tag es una validación; no se presenta como segunda autoridad.

## Flujo de interacción

### 1. Descubrimiento

1. El usuario inicia CLI o Launcher con A instalada.
2. `check` consulta únicamente el remoto canónico configurado por 1/5.
3. Si B tiene tag anotado, ref/commit/versión válidos, se muestra actualización disponible.
4. Si B es lightweight o hay mismatch, se muestra release inválida y no se habilita apply.
5. Si no hay red, se muestra offline/stale con procedencia y antigüedad del cache; no se certifica una release.
6. No se ejecuta checkout, build o apply en este paso.

### 2. Plan y confirmación

1. Se genera un plan inmutable con identidad heredada, versión instalada, pasos, impacto global y rollback.
2. La UI advierte que existe un solo CLI user-level compartido por todos los proyectos.
3. CLI y Launcher muestran el mismo plan normalizado.
4. Confirmar entrega el plan al helper; cancelar no crea estado durable.
5. Un cambio de identidad entre plan y confirmación invalida el plan; no se recalcula silenciosamente.

### 3. Apply y reinicio

1. El helper ejecuta preflight, staging/fetch, dependencias, build, verify, activate, persist y cleanup.
2. El progreso no promete éxito antes de verificar versión, launcher y manifest.
3. En éxito se informa B instalada con ref/commit y Launcher reinicia según 4/5.
4. En B→B se informa sin cambios; nunca se usa “instalado”.
5. Las versiones Node/npm efectivas son datos de diagnóstico/gate, no targets modificables desde UI.

### 4. Fallo, rollback y recovery

1. Un fallo muestra fase, outcome y siguiente acción sin exponer secretos.
2. Si rollback restaura A, se informa fallo y A activa/verificada.
3. Si no puede demostrarse consistencia, se muestra `recovery-required`; CLI sale distinto de cero y Launcher ofrece recovery.
4. Recover vuelve a observar versión, ref/commit, launcher y manifest; no confía solo en journal.
5. Repetir recover no empeora el estado.

## Estados visibles

| Estado | Mensaje humano esperado | JSON/DTO | Acción |
|---|---|---|---|
| Actualizado | CLI global en release publicada. | versiones/ref/commit concordantes | check |
| Actualización disponible | A instalada; B anotada y validada. | inventario fresco | plan |
| Release inválida | Tag lightweight, ref/commit/versión discordantes. | error tipado | corregir candidato; no override |
| Offline/stale | No se pudo refrescar; cache con antigüedad. | frescura/procedencia | retry/check |
| Aplicando | Fase actual, sin falso porcentaje. | transaction ID/fase | esperar/cancelar solo si contrato permite |
| Sin cambios | B ya instalada. | `noop`/`unchanged` | cerrar/check |
| Falló; A restaurada | B no se instaló; A verificada. | `failed` | retry/diagnóstico |
| Recovery requerido | Estado no certificable automáticamente. | `recovery-required`, exit no cero | recover/troubleshooting |
| Toolchain incompatible | Node/npm fuera de engines o celda fallida. | error de preflight/gate | STOP; no ajustar contrato |

## Baseline y documentación

Una baseline completa roja no se oculta en UI, tests o docs. Se clasifica y resuelve fuera del diff self-update antes del gate final. Certification no añade botones/comandos para saltar pruebas, forzar tags lightweight o ignorar rollback.

Solo después de GO técnico se reconciliarán los claims runtime en `README.md`, `docs/README.md`, `docs/installation.md`, `docs/overview.md`, `docs/cli/README.md`, la guía `docs/cli/self-update.md`, documentos CLI conflictivos y `specs/manuales/03_Actualizar_Versiones.md`. Hasta entonces, la spec describe el comportamiento objetivo pero no afirma que los comandos nuevos de certificación ya existan.

## Comportamiento reactivo y paridad

- Un check actualiza vistas de inventario, pero no habilita apply ante tag lightweight, mismatch o preflight fallido.
- Al adquirir lock, otras ventanas/CLIs muestran ocupado y no lanzan otro updater.
- Launcher consume eventos del helper; no duplica side effects por render o reconexión.
- Tras activación, el proceso A no carga módulos B; Launcher termina/reinicia y reobserva identidad.
- Persistencia fallida dispara rollback/recovery, nunca success con warning.
- Manifest stale/corrupto produce diagnóstico; no sintetiza `0.0.0`.
- `--json` conserva stdout parseable; progreso humano va por el canal definido por 2/5.
- El E2E compara semántica CLI/Launcher en Windows y Linux. Una divergencia bloquea aunque ambas rutas funcionen separadas.
- El gate se ejecuta con las cuatro celdas Node/npm cerradas; un fallo exige STOP y cambio explícito de spec, no una adaptación de mensajes o tests.