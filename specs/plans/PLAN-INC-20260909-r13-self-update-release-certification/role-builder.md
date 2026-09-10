# Rol de plan: builder

## Rol

- nombre del rol: builder
- repositorio(s) asignado(s): axiom-code-builder
- responsabilidad principal: implementar y demostrar el gate final ACC-080 sin absorber la lógica propietaria de los incrementos 1–4
- handoff obligatorio: reviewer independiente, integrador canónico y operador Axiom/Core

## Objetivo del rol en este plan

Construir una certificación reproducible de release Git para el monorepo completo y producir evidencia suficiente para una decisión STOP/GO. El builder debe unir version identity, CLI/state, updater y Launcher existentes; validar clean clone y E2E A→B en Windows/POSIX; integrar el installer en la validación estándar; y corregir documentación runtime solo después del GO técnico.

No se espera que el builder cierre artifacts, edite metadata/receipts/índices ni publique una release.

## Alcance incluido

### 1. Preflight y handoff

- Reconfirmar los cuatro incrementos predecesores y mapear sus boundaries reales.
- Ejecutar pruebas focales de identidad, manifest/CLI, updater y Launcher.
- Devolver defects al owner correcto en vez de duplicar implementación.
- Registrar la matriz 1→5 y las condiciones STOP.

### 2. Compatibilidad

- Probar y proponer rangos exactos Node/npm.
- Declarar los rangos y, si se aprueba, `packageManager` en la autoridad versionada.
- Mantener la matriz CI sincronizada con la declaración.
- No inferir soporte desde `@types/node`, lockfile o entorno local.

### 3. Release identity y guard

- Hacer de la versión raíz la única autoridad de producto.
- Eliminar literales independientes en la CLI/runtime mediante proyección o generación determinista.
- Implementar `scripts/release/validate-release.mjs` y tests.
- Validar `refs/tags/v<SemVer>`, fetch del remoto canónico, peel a commit y coherencia con candidate/CLI/manifest.
- Implementar guard de `private: true` en raíz/workspaces y rechazo de package/tarball standalone.
- Asegurar que el validator solo observa y nunca crea tags, publica o corrige el source tree.

### 4. Validación estándar y clean clone

- Separar `test:vitest` y `test:installer` y agregarlos bajo `npm test`.
- Mantener `node --test scripts/install-global.test.mjs` como comando focal.
- Crear `scripts/release/verify-clean-clone.mjs` con temp root, clone limpio, entorno allowlisted, `npm ci`, typecheck/build/suites e installed smokes.
- Verificar launcher target real y ausencia de residuos host.
- Mantener lógica portable en Node, no en shell específico.

### 5. E2E y resiliencia

- Crear remoto bare local y releases A/B de fixture; no usar ni afirmar tags upstream.
- Aislar HOME, USERPROFILE, npm prefix/cache, PATH y temporales.
- Ejecutar discover/check/plan/apply con updater real.
- Verificar B por entry absoluto y shim/binlink, manifest y provenance.
- Probar B→B sin mutación.
- Implementar fault matrix F-01..F-18 con test hooks no disponibles como bypass de producción.
- Demostrar rollback completo o `recovery-required` y recover idempotente.
- Probar paridad CLI/Launcher y restart sin módulos mezclados.

### 6. Plataforma y CI

- Ejecutar la matriz mínima Windows/Linux × Node/npm inferior/reciente.
- Reutilizar workflow existente equivalente; si no existe, proponer/crear el workflow mínimo indicado por el plan.
- Mantener permisos read-only y sin credenciales npm.
- Hacer que CI invoque los mismos scripts que local.

### 7. Documentación y evidencia

- Tras GO técnico, actualizar los archivos runtime exactos descritos en el plan y crear/indexar `docs/cli/self-update.md`.
- Diferenciar instalación soportada de setup de desarrollo.
- Retirar npm package, tarball CLI y TUI como rutas vigentes; conservar historia claramente rotulada cuando aporte valor.
- Completar E-01..E-19 con comandos, plataformas, Node/npm, refs/commits de fixture/candidate y resultados reales.
- Preparar un handoff autocontenido al reviewer sin declarar cierre.

## Fuera de alcance

- Reescribir discovery/SemVer/cache de ACC-077.
- Reescribir manifest v2, writer/lock, JSON o parser CLI de ACC-079.
- Reescribir planner/apply/staging/rollback/recover de ACC-078.
- Reescribir endpoints/runner/progreso/restart del Launcher.
- Publicar npm, ejecutar `npm pack` como release, retirar `private: true` o diseñar un package standalone.
- Crear tags/releases upstream, push, merge o cambios Git destructivos.
- Restaurar TUI o añadir update automático.
- Inventar PASS, tags, commits, outputs o receipts.
- Modificar manualmente metadata, status, enlaces, receipts, índices o carpetas archivadas.
- Ejecutar el cierre/archivo Core; el builder solo entrega evidencia al operador autorizado.

## Dependencias

### Bloqueantes funcionales

- Handoff de `INC-20260909-r13-self-update-release-identity`.
- Handoff de `INC-20260909-r13-self-update-state-cli-contract`.
- Handoff de `INC-20260909-r13-self-update-transactional-updater`.
- Handoff de `INC-20260909-r13-self-update-launcher-integration`.
- Decisión exacta Node/npm aceptada antes de fijar la matriz.

### Superficies runtime

- `package.json`, `package-lock.json`, manifests workspace y `vitest.config.ts`.
- `scripts/install-global.mjs`, `scripts/install-global.test.mjs`, `scripts/verify-first-project-readiness.mjs`.
- `apps/cli/src/index.ts` y comandos/tests de self-update.
- user-workspace manifest, updater/versioning, Launcher y doctor.
- docs runtime enumeradas en el plan.

### Dependencias de cierre

- Reviewer independiente distinto del builder principal.
- Integrador con autoridad sobre Axiom.Spec para conocimiento estable.
- Operador Axiom/Core para receipts, índices, close y archive.

## Orden de trabajo del builder

1. Emitir preflight/handoff o STOP.
2. Cerrar decisión Node/npm con probes clean-clone.
3. Implementar autoridad de versión y validator/package guard.
4. Integrar installer en `npm test` y conservar comandos focales.
5. Implementar clean-clone harness.
6. Implementar E2E A→B happy/no-op.
7. Añadir fault/recovery matrix.
8. Añadir/ajustar CI y ejecutar matriz Windows/POSIX.
9. Resolver todos los fallos nuevos; clasificar blockers preexistentes por owner.
10. Obtener GO técnico.
11. Actualizar docs runtime y volver a validar.
12. Completar ledger y entregar a review independiente.
13. Resolver hallazgos con nuevas ejecuciones; no sobrescribir evidencia fallida.
14. Entregar propuesta de integración estable y handoff Core.

No se adelantan los pasos 10–14 si un gate anterior falla.

## Validaciones y evidencias esperadas

### Comandos obligatorios

```text
npm ci
npm run typecheck
npm run build
npm test
node --test scripts/install-global.test.mjs
npx vitest run apps/cli/tests/self-update.test.ts
npx vitest run packages/user-workspace/tests/self-update.test.ts
npx vitest run packages/installer/tests
npx vitest run packages/launcher/tests/launcher.test.ts
npx vitest run packages/versioning/tests apps/cli/tests/rollback.test.ts
npx vitest run packages/doctor/tests
npm run readiness:first-project
npm run test:release-validator
npm run release:clean-clone
npm run test:release-e2e
npm run release:certify
git diff --check
```

Los últimos cuatro scripts son entregables del plan y no se presentan como existentes antes de implementarlos. `release:validate` requiere la identidad candidate en entorno cuando se ejecuta en modo release; los tests usan solo refs locales de fixture.

### Evidencia por fase

- Handoff: rutas/símbolos/contracts de 1–4 y resultados focales.
- Compatibility: versiones exactas por celda y decisión aprobada.
- Identity: casos positivos/negativos del validator, JSON/exit codes y guard private.
- Clean clone: transcript completo, tracked tree, launcher real y smokes.
- E2E: topología A/B, plan, apply, manifest/provenance y no-op.
- Faults: snapshots before/after, outcome y recovery por F-01..F-18.
- Platform: Windows/Linux y launcher específico, con Node/npm efectivos.
- Isolation: environment redactado, network guard y ausencia de paths externos.
- Docs: diff, links y clasificación de claims vigentes/históricos.
- Review: comandos reproducidos, hallazgos y resolución.

Cada evidencia enlaza AC-080 y E-ID, conserva fallos/reintentos y evita datos sensibles. Los receipts oficiales se generan por Core, no por el builder.

## Criterios personales de STOP/GO

### El builder emite STOP si

- falta un predecesor o necesita duplicar su mecanismo;
- no puede fijarse compatibilidad Node/npm exacta;
- el candidate no tiene identidad Git verificable;
- `private: true` o el modelo monorepo completo se debilitan;
- clean clone no reproduce el build;
- installer/launcher encuentra recursos del host;
- un false positive de apply/no-op sigue posible;
- rollback/recovery no convergen en ambas plataformas;
- CI requerida no puede ejecutar Windows/POSIX;
- docs requieren afirmar algo que el gate no demostró.

### El builder entrega GO técnico si

AC-080-01..18 tienen evidencia reproducible, no hay blockers conocidos y el diff no amplía el alcance. Ese GO habilita docs/review; no autoriza cierre.

### El builder entrega handoff final si

- docs runtime están alineadas y revalidadas;
- ledger E-01..E-19 está completo;
- rutas temporales/artefactos de test fueron limpiados;
- no hay cambios manuales de lifecycle;
- el paquete de review contiene comandos exactos y riesgos residuales.

## Bloqueos y observaciones

- La inspección preparatoria no observó un contrato Node/npm ni CI equivalente; ambos deben reconfirmarse al empezar.
- La versión de producto aparecía duplicada y el installer test quedaba fuera de Vitest. Son objetivos del gate, no evidencia de fallo futuro.
- El self-update auditado antes de los incrementos 1–4 no demostraba un cambio real de release ni rollback POSIX completo. Si esos gaps persisten, pertenecen a los predecesores y bloquean 5/5.
- “Launcher” puede referirse a la superficie de aplicación o al shim/binlink. El builder debe nombrarlos de forma explícita en código/tests para no confundir sus responsabilidades.
- Los fixture tags A/B son locales y efímeros. Ningún log o documento debe presentarlos como release upstream.
- La documentación solo se toca tras GO técnico, excepto notas internas necesarias para implementar.
- El reviewer y Core son gates reales; el builder no puede autoaprobar ni archivar su propio resultado.