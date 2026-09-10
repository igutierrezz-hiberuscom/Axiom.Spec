# 03 Criterios de Aceptación

## Criterios de aceptación

ACC-080 se satisface únicamente cuando los 24 criterios tienen evidencia nueva, reproducible y revisada. Los resultados históricos no cuentan como ejecución del gate.

| ID | Criterio | Evidencia mínima |
|---|---|---|
| **AC-080-01** | Los handoffs siguen 1 identity → 2 state/CLI → 3 updater/helper → 4 Launcher → 5 certification; 5 no absorbe gaps ni crea mecanismos paralelos. | Matriz 1→5 con owner, rutas, contratos y pruebas focales. |
| **AC-080-02** | El validator acepta solo tags anotados del remoto canónico con ref exacta `refs/tags/v<SemVer>` y commit candidato concordante; rechaza lightweight. | Casos anotado válido, lightweight con nombre válido, ref inválida/ausente, remoto incorrecto y commit distinto. |
| **AC-080-03** | Existe una sola autoridad de versión entregada por 1/5; todas las proyecciones concuerdan y el diff de 5 no migra consumidores ni crea otra identidad. | Tests cruzados y revisión del diff/impacto de ownership. |
| **AC-080-04** | Discordancia de tag, tipo de objeto, commit o versión bloquea antes de build/activación. | Matriz negativa, exit no cero y snapshot sin mutación. |
| **AC-080-05** | Raíz/workspaces mantienen `private: true`; no se certifica package/tarball npm. | Guard automático y casos negativos sobre copias temporales. |
| **AC-080-06** | Se declaran `engines.node >=20.14.0 <23` y `engines.npm >=10.7.0 <11`; pasan las cuatro celdas Windows/Linux × {20.14.0/10.7.0, 22.14.0/10.9.2}. | Manifests, versiones efectivas y ledger por celda. |
| **AC-080-07** | La baseline completa existente se captura antes del diff y queda verde antes del gate final. Un rojo preexistente se clasifica y resuelve en un cambio separado, sin ocultarlo ni ampliar 5/5. | Primera ejecución, clasificación/owner, vínculo a resolución separada y rerun completo verde. |
| **AC-080-08** | Validator/tests, agregadores raíz, clean-clone, E2E y workflow son entregables nuevos; al final existen y los comandos focales/estándar funcionan. | Diff de entregables, tests de agregación y ejecución; inventario que distingue baseline de futuro. |
| **AC-080-09** | Un clean clone ejecuta `npm ci`, typecheck, build, baseline, suites focales, installer y smokes sin outputs previos. | Ledger por comando, tree limpio y snapshot de residuos. |
| **AC-080-10** | El E2E local instala A, descubre/planea B, aplica B y observa B por entry absoluto y shim/binlink. | Remoto bare, tags A/B anotados, transcript y assertions de target/version. |
| **AC-080-11** | Manifest, status y provenance de B coinciden con ref, commit, versión y launcher ejecutado. | Snapshot normalizado contra Git/build identity. |
| **AC-080-12** | B→B produce no-op/unchanged sin instalación ficticia ni fases mutantes. | Comparación byte a byte y traza de fases. |
| **AC-080-13** | Todo fallo preactivación deja A íntegra y ejecutable. | Fault matrix de fetch/tag/identity/lock/npm ci/build/smoke. |
| **AC-080-14** | Todo fallo postactivación restaura A o devuelve `recovery-required`; jamás `installed`. | Fault matrix de activación, manifest, interrupción, cleanup y rollback fallido. |
| **AC-080-15** | `recover` es idempotente y converge a A/B observada o conserva recovery accionable. | Reinicio, dos invocaciones y snapshots before/after. |
| **AC-080-16** | CLI y Launcher presentan el mismo plan/outcome/provenance y no mezclan módulos A/B. | Comparación DTO/envelope y prueba de cierre/reinicio. |
| **AC-080-17** | El E2E aísla home, prefix, cache, temporales y PATH; no usa red Git externa ni deja residuos host. | Environment redactado, network guard y snapshot externo. |
| **AC-080-18** | Windows valida `.cmd`, quoting/paths con espacios; Linux valida binlink, permisos/realpath; cada una ejecuta ambas celdas Node/npm. | Cuatro celdas obligatorias con escenarios críticos comunes. |
| **AC-080-19** | `--version`, `--help` y doctor se ejecutan sobre el candidato instalado, no sobre checkout/CLI host. | Resolución de target por entry absoluto y PATH controlado. |
| **AC-080-20** | Solo después de GO técnico se reconcilian claims runtime, guía self-update y `specs/manuales/03_Actualizar_Versiones.md`. | Orden de gates, diff documental y nueva validación. |
| **AC-080-21** | Docs no ofrecen TUI, `npx axiom`, package npm ni tarball CLI como rutas vigentes. | Búsqueda dirigida y revisión de claims activos/históricos. |
| **AC-080-22** | El ledger enlaza cada criterio con plataforma, Node/npm, tag/ref/commit, comando, resultado y fallos conservados. | Ledger completo sin PASS inventados ni reintentos sobrescritos. |
| **AC-080-23** | Review independiente no deja blockers y comprueba baseline, no duplicación, rollback, cuatro celdas, docs e integración. | Informe, reproducciones y resolución trazable. |
| **AC-080-24** | Conocimiento estable se integra en owners y cierre/indexado/archivo se ejecuta solo mediante Core. | Checklist specs/manual/context y receipts Core; ningún cambio manual estructural. |

## Contrato de matriz obligatorio

| SO | Node | npm | Resultado requerido |
|---|---:|---:|---|
| Windows | 20.14.0 | 10.7.0 | PASS del gate mínimo común |
| Windows | 22.14.0 | 10.9.2 | PASS del gate mínimo común |
| Linux | 20.14.0 | 10.7.0 | PASS del gate mínimo común |
| Linux | 22.14.0 | 10.9.2 | PASS del gate mínimo común |

Un fallo produce STOP. Modificar una versión, rango o celda sin cambiar primero esta spec es incumplimiento aunque CI quede verde.

## Happy path

1. Se verifica handoff 1→4 y se ejecuta la baseline completa existente antes del diff.
2. Si la baseline está roja, se preserva/clasifica el fallo, se resuelve fuera del diff self-update y se repite completa hasta verde.
3. 5/5 implementa sus nuevos scripts/tests/workflow; su ausencia inicial no se usa como fallo de entrada.
4. El gate recibe ref y commit candidatos, obtiene la ref del remoto canónico, confirma objeto `tag` anotado y hace peel al commit.
5. Lee la autoridad de versión de 1/5 y verifica todas sus proyecciones sin migrar consumidores.
6. Comprueba `private: true`, lockfile, engines y versión efectiva de toolchain.
7. Crea clean clone y ejecuta build, baseline, focales, installer y smokes.
8. Instala A en home/prefix temporal; el remoto local anuncia B mediante tag anotado.
9. CLI y Launcher descubren/planean B sin mutación; updater/helper aplica B tras confirmación.
10. Entry absoluto y launcher devuelven B; manifest/status/provenance coinciden.
11. B→B devuelve no-op y conserva estado durable.
12. Se ejecutan faults y las cuatro celdas. Solo con AC-080-01..19 verdes se emite GO técnico.
13. Después del GO técnico se actualizan docs/manual, se revalida y se completa ledger/review/integración.
14. Core, no los Markdown, gobierna cualquier transición estructural.

## Validaciones y errores

- Ref no exacta, objeto lightweight, tag ausente tras fetch, peel a otro commit o versión divergente: rechazo, exit no cero y cero mutación.
- Divergencia de un consumidor de versión: STOP y devolución al owner; 5/5 no lo migra.
- Falta o diferencia en engines, versión efectiva distinta de la celda o celda fallida: STOP, sin ajuste silencioso.
- `npm ci` modifica lockfile/falla: STOP, sin fallback.
- `private: true` ausente o canal npm introducido: STOP de arquitectura.
- Baseline completa preexistente roja: bloqueo del gate final hasta resolución separada y rerun verde.
- Skips, exclusiones, cambios de expectativas, warnings o reintentos sin causa no resuelven un rojo.
- Exit 0 del installer con versión/target/ayuda/doctor incorrectos: fallo y rollback.
- Persistencia fallida: rollback o `recovery-required`, nunca éxito con warning.
- Rollback fallido: conservar journal/backup, exit no cero y diagnóstico de recover.
- Escritura fuera del temp root o acceso Git externo: fallo inmediato.
- Diferencia semántica CLI/Launcher: STOP aunque cada ruta funcione aislada.
- Documentación antes de GO técnico: orden inválido y STOP documental.

## Comandos existentes y entregables futuros

La evidencia debe distinguir:

- **Existentes al baseline, sujetos a reconfirmación:** `npm ci`, `npm run typecheck`, `npm run build`, `npm test`, `node --test scripts/install-global.test.mjs`, `npm run readiness:first-project` y suites focales presentes.
- **Entregables de 5/5:** scripts `test:vitest`, `test:installer`, `test:release-validator`, `release:validate`, `release:clean-clone`, `test:release-e2e`, `release:certify`; archivos del validator/harness/E2E y workflow Windows/Linux.

No se exige que los segundos existan en el preflight; sí deben existir y pasar antes de GO técnico.

## Rollback y efectos observables

| Situación | Outcome | Estado ejecutable | Efecto durable permitido |
|---|---|---|---|
| A instalada, B publicada | update disponible | A | cache de check con procedencia/frescura |
| Plan B | read-only | A | ninguno |
| Apply B exitoso | `installed` | B | launcher/manifest/provenance coherentes |
| B ya instalada | `unchanged`/`noop` | B | ninguno salvo check permitido |
| Fallo preactivación | `failed` | A | diagnóstico; sin drift |
| Fallo postactivación, rollback completo | `failed` | A | evidencia del intento/rollback |
| Rollback no demostrable | `recovery-required` | no se presume A/B | journal/backup/guía conservados |
| Recover exitoso | outcome heredado | A o B observada | manifest/provenance reconciliados |
| Tag lightweight | release rechazada | sin cambio | evidencia del validator |

## Documentación e integración

Después de GO técnico se revisan `README.md`, `docs/README.md`, `docs/installation.md`, `docs/overview.md`, `docs/cli/README.md`, `docs/cli/self-update.md`, claims conflictivos en `docs/cli/tui.md`/`docs/cli/doctor.md` y `specs/manuales/03_Actualizar_Versiones.md`. El conocimiento estable se integra en los owners de `specs/00..08` y `context/` tras review. No se copian logs ni se editan índices/receipts/status manualmente.

El incremento no se acepta por tener documentación completa: requiere baseline verde, identidad anotada, cuatro celdas, E2E/faults, rollback/recovery, ledger y review independiente.