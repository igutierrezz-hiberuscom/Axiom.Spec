# 01 Requisitos

## Objetivo del documento

Especificar el comportamiento exigible al quinto incremento R-13.5 para que ACC-080 funcione como certificación final. Identity, state/CLI, updater/helper y Launcher pertenecen a los incrementos 1–4; 5/5 solo valida sus contratos y proyecta en evidencia la autoridad de versión creada por 1/5.

## Requisitos del incremento

### A. Dependencias, secuencia y frontera

- **REQ-080-001 — Secuencia obligatoria.** El orden es 1 identity → 2 state/CLI → 3 updater/helper → 4 Launcher → 5 certification. El gate solo puede dar GO si los cuatro predecesores entregaron contratos compatibles y evidencia revisable.
- **REQ-080-002 — Sin quinta implementación.** CLI y Launcher deben invocar los mismos servicios de check, plan, apply y recover. 5/5 no crea una identidad, manifest, updater, helper o runner paralelo ni reinterpreta outcomes.
- **REQ-080-003 — Frontera de autoridad.** La autoridad única de versión es un handoff de 1/5. 5/5 puede leerla, compararla y proyectarla en el resultado de certificación; no migra consumidores, no cambia su owner y no introduce un literal o artefacto alternativo como autoridad.

### B. Identidad y convención de release

- **REQ-080-004 — Unidad de release.** La unidad certificada será el monorepo Git completo de Axiom en un commit exacto; `@axiom/cli` aislado no será artefacto de release.
- **REQ-080-005 — Ref certificable.** `productVersion` será SemVer estricto sin `v` y `releaseRef` será exactamente `refs/tags/v<productVersion>`. La ref obtenida del remoto canónico deberá apuntar a un objeto Git de tipo `tag` anotado. Un tag lightweight, cuyo objeto sea directamente `commit`, se rechazará aunque el nombre y commit sean correctos.
- **REQ-080-006 — Commit y remoto canónicos.** El validator hará fetch explícito de la ref exacta desde el remoto canónico configurado por 1/5, comprobará el tipo del objeto antes de hacer peel y comparará el commit completo resultante con el candidato. Un cache local no sustituirá esta comprobación.
- **REQ-080-007 — Proyecciones de la autoridad heredada.** Versión raíz, workspaces gobernados, CLI compilada, build identity, plan, manifest, status, provenance y Launcher deberán concordar con la autoridad entregada por 1/5. Estas son proyecciones verificadas, no autoridades adicionales.
- **REQ-080-008 — Discordancia y ownership.** Cualquier mismatch ref↔SemVer, tipo de tag, ref↔commit o autoridad↔proyección bloqueará la release antes de publicar o activar. Si corregirlo requiere migrar un consumidor, el defecto vuelve a 1/5 o al predecesor propietario y no se corrige dentro de 5/5.
- **REQ-080-009 — Identidad reproducible.** Versión, ref y commit no dependerán de timestamps o datos host-dependent. El resultado de certificación referenciará la identidad heredada; no persistirá una segunda identidad durable.

### C. Compatibilidad y guard de distribución

- **REQ-080-010 — Contrato Node/npm cerrado.** El repositorio declarará `engines.node` con `>=20.14.0 <23` y `engines.npm` con `>=10.7.0 <11`. La certificación mínima tendrá cuatro celdas obligatorias: Windows y Linux, cada uno con Node 20.14.0/npm 10.7.0 y Node 22.14.0/npm 10.9.2. Cada celda registrará `node --version` y `npm --version` efectivos.
- **REQ-080-011 — Cambio explícito del contrato.** Si una celda obligatoria falla, el resultado será STOP. No se ajustarán rangos, versiones o matriz silenciosamente en implementación o CI; cualquier cambio exige primero una modificación explícita y revisada de la spec.
- **REQ-080-012 — Lockfile.** Toda construcción de release partirá de `package-lock.json` y ejecutará `npm ci`. Un lockfile desactualizado, modificado o incompatible con el npm de la celda será STOP, sin fallback a `npm install`.
- **REQ-080-013 — Monorepo privado.** La raíz y todos los workspaces gobernados conservarán `private: true`. Un guard automático rechazará la retirada de ese valor, un script de publicación soportada o el uso de `npm pack`/`npm publish` como canal de certificación.

### D. Gate sobre clean clone

- **REQ-080-014 — Clone limpio.** El gate trabajará en un directorio temporal creado desde el commit candidato, sin `node_modules`, `dist`, cache incremental ni archivos no trackeados del checkout productor.
- **REQ-080-015 — Secuencia mínima.** En el clone se ejecutarán, en orden controlado, `npm ci`, typecheck, build, baseline estándar, suites focales de self-update/installer/Launcher/versioning/doctor, el installer nativo y los smokes instalados.
- **REQ-080-016 — Binario correcto.** `--version`, `--help` y doctor se ejecutarán por ruta absoluta del entry candidato y por el shim/binlink del prefix temporal. El harness comprobará el target real para impedir que PATH encuentre otra instalación.
- **REQ-080-017 — Evidencia y residuos.** Cada comando registrará argv, cwd, entorno relevante redactado, exit code y stdout/stderr. El gate verificará que el host queda sin cambios fuera del temp root.
- **REQ-080-018 — Fallo cerrado.** Exit distinto de cero, timeout, signal, versión inesperada, mutación fuera del temp root o falta de evidencia detendrá la certificación.

### E. Validación estándar y entregables nuevos

- **REQ-080-019 — Baseline existente identificada.** Los comandos y suites que ya existan al iniciar se inventariarán y ejecutarán como baseline completa. La ausencia de scripts/tests/workflow de release aún no creados no se clasificará como fallo preexistente: son entregables de 5/5.
- **REQ-080-020 — Runners explícitos.** El installer existente `node --test scripts/install-global.test.mjs` se integrará en la validación estándar mediante nuevos scripts raíz, manteniendo comandos focales diferenciados para Vitest y `node:test`.
- **REQ-080-021 — Certificación invocable.** 5/5 entregará validator y tests, harness clean-clone, E2E, scripts raíz de agregación/certificación y workflow Windows/Linux. Todos serán portables y se documentarán como futuros hasta que el diff los implemente; no son precondiciones de entrada.

### F. E2E hermético y resiliencia

- **REQ-080-022 — Fixture A/B anotado.** El E2E generará un remoto bare local y dos tags anotados SemVer A/B que resuelvan a commits distintos. Una fixture negativa lightweight demostrará el rechazo. Ninguna ref de fixture representará un tag upstream.
- **REQ-080-023 — Aislamiento.** `HOME`, `USERPROFILE`, `npm_config_prefix`, `npm_config_cache`, `PATH`, `TMPDIR`, `TEMP`, `TMP` y rutas Axiom user-level se fijarán dentro del temp root. La red Git externa no será alcanzable por diseño.
- **REQ-080-024 — Recorrido real.** Desde A se ejecutarán status/check, plan, confirmación y apply de B con fetch, staging, `npm ci`, build, smokes y activación reales; después se observará B por ruta absoluta y shim/binlink.
- **REQ-080-025 — Provenance.** Manifest y status contendrán la versión B, `releaseRef`, `releaseCommit`, remoto/procedencia verificable y timestamps definidos por 2/5. Los campos corresponderán a lo ejecutado, no solo al target solicitado.
- **REQ-080-026 — No-op.** Repetir B→B producirá `unchanged`/`noop`, no ejecutará fases mutantes y no alterará bytes durables salvo campos de check expresamente permitidos.
- **REQ-080-027 — Fallos preactivación.** Fallos de fetch, tipo de tag, identidad, lock, `npm ci`, build o smoke dejarán A ejecutable y el manifest previo intacto.
- **REQ-080-028 — Fallos postactivación.** Fallos de activación parcial, persistencia o limpieza restaurarán commit, build/dependencias, entry activo y manifest de A. Si la restauración no puede demostrarse, el outcome será `recovery-required`, nunca `installed`.
- **REQ-080-029 — Recover.** Un reinicio detectará journal/estado de recuperación y `recover` alcanzará un estado conocido o conservará `recovery-required` con diagnóstico accionable. La recuperación será idempotente.
- **REQ-080-030 — Paridad CLI/Launcher.** Ambos canales mostrarán el mismo inventario, plan, confirmación, progreso acotado, outcome, provenance y recovery. Launcher cerrará/reiniciará según 4/5 para no mezclar módulos A/B.
- **REQ-080-031 — Plataformas y toolchain.** Windows validará `.cmd`, quoting y paths con espacios; Linux validará binlink, permisos y realpath. Las cuatro celdas de REQ-080-010 ejecutarán el gate mínimo común.

### G. Documentación, evidencia y cierre

- **REQ-080-032 — Documentar después de GO técnico.** Ningún claim runtime o manual se actualizará antes de que AC-080-01..18 tengan evidencia técnica suficiente y la baseline completa esté verde.
- **REQ-080-033 — Reconciliación mínima.** Después de GO técnico se reconciliarán `README.md`, `docs/README.md`, `docs/installation.md`, `docs/overview.md`, `docs/cli/README.md`, la nueva `docs/cli/self-update.md`, los claims conflictivos de `docs/cli/tui.md` y `docs/cli/doctor.md`, y `specs/manuales/03_Actualizar_Versiones.md`.
- **REQ-080-034 — Claims vigentes.** Los documentos no presentarán package npm, `npx axiom`, tarball de `@axiom/cli` ni TUI como rutas vigentes. Las referencias históricas se rotularán inequívocamente.
- **REQ-080-035 — Ledger.** Cada gate tendrá evidencia trazable por criterio, plataforma, Node/npm efectivo, ref/tag object/commit, comando, exit code y artefactos. Se conservarán fallos y reintentos; no se fabricarán PASS.
- **REQ-080-036 — Revisión independiente.** Una persona o agente distinto del implementador revisará diff, ledger, baseline, aislamiento, rollback/recovery, ausencia de segunda identidad, documentación y las cuatro celdas. Un blocker implica STOP.
- **REQ-080-037 — Integración estable.** Tras GO técnico y documental se integrará conocimiento consolidado en los archivos propietarios de `specs/00..08`, `specs/manuales/03_Actualizar_Versiones.md` y `context/`; no se copiarán logs ni detalles efímeros.
- **REQ-080-038 — Gobierno por Core.** Cierre, enlaces, receipts, índices y archivo se realizarán solo mediante Axiom Core. La edición de esta spec no autoriza cambios manuales de metadata o status.

### H. Baseline completa roja

- **REQ-080-039 — Bloqueo y clasificación.** La baseline completa existente se ejecutará antes del diff y antes del gate final. Si está roja, se conservará evidencia, se clasificará cada fallo y se resolverá mediante el predecesor o bug propietario en un diff separado. 5/5 no ocultará el fallo ni ampliará su diff para corregir bugs ajenos. El gate final exige una nueva ejecución completa verde.
- **REQ-080-040 — Sin evasiones.** No se obtendrá verde mediante skips, exclusiones, cambio de expectativas, supresión de logs, reintentos sin causa o reducción de matriz. Un fallo nuevo introducido por 5/5 se corrige dentro de su alcance; un fallo preexistente se resuelve fuera de su diff.

## Reglas de negocio relevantes

1. Actualizar el CLI global afecta a todos los proyectos locales; no crea una instalación por proyecto.
2. `publishedVersion`, `downloadedVersion` e `installedVersion` son ejes distintos heredados; ninguno crea autoridad paralela.
3. Check no aplica; plan no muta; apply requiere confirmación; dry-run no alcanza rutas mutantes.
4. Un exit cero del instalador no basta: versión, ayuda, módulos, doctor, launcher target y provenance deben verificarse.
5. Un no-op no es una instalación y nunca se registra como `installed`.
6. No se hacen stash, reset, force checkout ni clobber implícitos para salvar un candidato.
7. Si rollback no restaura un estado ejecutable conocido, se conserva evidencia y se exige recovery.
8. Los tests no tocan home/PATH reales ni dependen de red Git externa.
9. `private: true` es un guard de arquitectura; npm instala dependencias pero no entrega el producto.
10. Un tag con nombre correcto no certifica si no es anotado.
11. Una celda Node/npm fallida exige cambio explícito de spec antes de cambiar el contrato.
12. 5/5 no absorbe defectos de sus predecesores ni bugs ajenos.

## Fuera de alcance funcional

- Diseñar registry, canales npm, package standalone o publicación automática.
- Distribuir binarios nativos o installers MSI/pkg/deb/rpm.
- Firmado criptográfico, SBOM o attestation externa no exigidos por otra spec.
- Actualización automática, silenciosa o sin confirmación.
- Side-by-side de CLIs globales o runtimes por proyecto.
- Nueva TUI o rediseño visual de Launcher.
- Migrar consumidores de versión o cambiar contratos funcionales de 1–4 para hacer pasar el E2E.
- Corregir bugs preexistentes ajenos dentro del diff self-update.
- Crear tags upstream o certificar tags lightweight.
- Cerrar, archivar o modificar metadata sin Core.