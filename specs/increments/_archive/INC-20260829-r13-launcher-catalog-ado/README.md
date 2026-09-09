# Catálogo, targeting e integración ADO del launcher R-13

> **Código**: INC-20260829-r13-launcher-catalog-ado
> **Estado lifecycle**: gestionado exclusivamente por Axiom Core; este README no fija status estructural
> **Fecha**: 2026-08-29
> **Acciones**: ACC-073, ACC-074 y pruebas correspondientes de ACC-076
> **Dependencia**: F; reutiliza el workflow canónico vigente de R-10

## Objetivo

Eliminar fallbacks/duplicaciones del catálogo launcher, mostrar comandos CLI realmente ejecutables, exigir identidad explícita fail-closed en mutaciones y alinear ADO con semántica local-first y enlaces seguros.

## Revalidación

`ACTION_RECONCILIATION` omite acciones lifecycle y usa fallback; previews muestran binarios inexistentes sin prefijo `axiom`; runners pueden derivar artefacto activo. Copy ADO promete prioridad/validación remota no ejecutada y `renderAdo` acepta href arbitrario.

## Alcance

- ACC-073: fuente declarativa compartida para catálogo/routing sobre `DEFAULT_WORKFLOWS`; previews `axiom <subcommand>` reales; retirar campos muertos; ID explícito y mismatch fail-closed en increment/bug/plan/role/E2E.
- ACC-074: artefacto local creado/validado primero; ADO opcional después; resultados separados; href solo HTTP(S).
- ACC-076: paridad, ausencia de fallback, comandos ejecutables, mismatch y enlaces mediante server/wrapper real y fixtures herméticos.

## No objetivos

- No redefinir transiciones ni declarar validadas ACC-041/045.
- No convertir ADO en requisito ni hacer llamadas externas reales.
- No compensar destructivamente un artefacto local válido si ADO falla.
- No telemetría H ni R-13.5.

## Decisiones cerradas

1. `@axiom/workflow` (`DEFAULT_WORKFLOWS`/config cargada) sigue siendo única fuente de comandos/transiciones. El catálogo launcher añade presentación/fields, pero referencia IDs canónicos y valida paridad al cargar.
2. No existe routing genérico para action IDs declarados. Acción sin reconciliación exacta falla con error de configuración; fallback solo puede existir para input explícitamente no-lifecycle y probado.
3. Preview CLI usa un único ejecutable: `axiom axiom-increment ...`, `axiom axiom-bug ...`, `axiom axiom-plan ...`, `axiom axiom-role ...`, `axiom axiom-qa-e2e ...` con flags reales.
4. `existingUsMode`/`existingBugMode` y cualquier field sin consumidor se retiran, no se inventa semántica.
5. Toda mutación lifecycle exige `id` en API, launcher y wrappers CLI. Create liga el solicitado; operaciones posteriores verifican que ID coincide con metadata/workflow activo. Singleton no autoriza ignorar mismatch ni derivar target.
6. La validación de ID ocurre antes de transición/receipt/write. Preview también muestra ID explícito.
7. Flujo ADO: (a) preview/token F; (b) crear/validar artefacto local; (c) devolver resultado local; (d) ofrecer create/link ADO opcional; (e) registrar resultado remoto separado. Fallo remoto no cambia éxito local ni borra artefacto.
8. No se promete validación remota previa si no se ejecuta. Existing work item solo se valida al invocar bridge configurado.
9. URLs externas se parsean y admiten solo `http:`/`https:`; inválidas se renderizan como texto, nunca href. Enlaces usan protección de opener/referrer.
10. Tests ADO usan fake tracker/bridge sin red.

## Riesgos

Cambios compartidos en workflow R-10 y compatibilidad de callers sin ID; deben actualizarse todos y reutilizar guard canónico. Cambios locales R-12 en panels.js se preservan.

## Compatibilidad

Se retira derivación implícita de ID, previews con binarios ficticios, fallbacks accidentales y copy ADO-first.

## Validación prevista

Paridad catálogo-workflows, todos adapters/actions, help/previews ejecutables, mismatch por lane, receipts ausentes en rechazo, ADO local/remote split, URL schemes, server real, build y diff-check.

## Resultado de implementación y evidencia (2026-09-08)

- `ACTION_RECONCILIATION` es la fuente declarativa de catálogo/routing; los comandos preview usan invocaciones reales `axiom ...`.
- Un adapter desconocido falla cerrado para cualquier acción lifecycle declarada; el fallback clipboard queda limitado a acciones sintéticas/no lifecycle. ID ausente o mismatch sigue rechazándose antes de receipt, transición o artifact mutation.
- ADO permanece local-first y opcional: preview/local/remote están separados, el bridge de tests es fake y los enlaces solo aceptan `http:`/`https:` sin userinfo, query ni fragment.
- La evidencia conjunta ACC-076 aporta los casos `ACC-073-01..03` y `ACC-074-01..02`; resultado agregado: 35 PASS, 0 FAIL, 0 TIMEOUT, con 14 checks de no mutación.
- Validación observada: 12 suites y 233 tests PASS en la batería focal conjunta; `npm run typecheck` PASS, `npm run build` PASS y `git diff --check` PASS.

## Integración estable

El contrato de catálogo declarativo, routing fail-closed, comandos CLI reales, identidad caller-owned, ADO local-first y URL segura fue reconciliado en `Axiom.Spec/specs/00..08` donde aplica y en `context/**`. El lifecycle, los receipts y el archivado se gestionan exclusivamente mediante Axiom Core.
