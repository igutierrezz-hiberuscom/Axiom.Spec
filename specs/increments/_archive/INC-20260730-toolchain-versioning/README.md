# INC-20260730-toolchain-versioning

## Metadata

- **ID**: INC-20260730-toolchain-versioning
- **Status**: closed
- **Goal**: Dotar a Axiom de versionado reproducible para toolchains externas con lockfile, canales stable/candidate/edge, detección de drift en doctor, y comandos plan/upgrade.
- **Scope**: `@axiom/toolchain`, `@axiom/doctor`, `apps/cli/src/commands/toolchain.ts`, `axiom.config/toolchain-catalog.yaml`
- **Non-goals**: Instalación automatizada de herramientas externas, rollback de binarios, detección automatizada de releases upstream, Workbench.

## Acceptance Criteria

1. `toolchain-catalog.yaml` ampliado con `channels` (stable/candidate/edge) y `versionExtractor` cuando existe un contrato local de probe; las tools instruction-only lo omiten deliberadamente.
2. `ToolEntry` extendido con `version`, `channel`, `probeOutput`, `probedAt`.
3. `toolchain.lock` creado, leído y escrito atómicamente en `.axiom-state/<project>/`.
4. `extractVersion()` en probe parsea la salida de `--version` usando regex del catálogo.
5. `compareVersions()` implementa comparación semver.
6. Doctor checks TC-020 (lockfile ausente o válido; si existe, schema correcto), TC-021 (versión locked compatible con catálogo), TC-022 (drift: installed vs locked), TC-023 (canal válido).
7. `axiom toolchain show` extendido con columnas VERSION y CHANNEL.
8. `axiom toolchain plan` muestra diff entre lockfile y catálogo (dry-run).
9. `axiom toolchain upgrade` aplica cambios con checkpoint y rollback del lockfile.
10. Build + test + doctor verdes al cierre.

## Implementation Plan

### Fase 1: Modelo de datos y detección

1. Extender `ToolEntry` en `types.ts` con `version`, `channel`, `probeOutput`, `probedAt`.
2. Extender `toolchain-catalog.yaml` con `channels` y `versionExtractor`.
3. Crear `ToolchainLock` schema y loader/writer en `@axiom/toolchain`.
4. Implementar `extractVersion()` en `probe.ts`.
5. Implementar `compareVersions()` en nuevo `semver.ts`.
6. Añadir checks TC-020..TC-023 en `@axiom/doctor`.
7. Extender `toolchain show` y `validate` en CLI.

### Fase 2: Sincronización

8. Implementar `planToolchainUpgrade()` en `plan.ts`.
9. Implementar `executeToolchainUpgrade()` en `upgrade.ts`.
10. Añadir subcomandos `plan` y `upgrade` en CLI.

## Decisiones de implementación

- `plan` compara el conjunto que la CLI resuelve a partir de las tools
	declaradas o ya lockeadas, salvo que el caller solicite explícitamente un
	subconjunto con `--id`. La función pura del planner, sin subconjunto
	explícito, considera solo las tools ya lockeadas. El catálogo no implica por
	sí solo que todas sus tools deban instalarse.
- `upgrade` modifica solo `toolchain.lock`; no instala, descarga ni reemplaza
	binarios externos. Requiere `--yes` para persistir cambios. Sin `--yes`
	funciona como una vista previa equivalente a `--dry-run`.
- El lockfile sigue siendo estado operativo local bajo
	`.axiom-state/<project>/` y permanece ignorado por Git junto con el resto
	del estado generado. La reproducibilidad se obtiene mediante el contenido
	del lockfile en el entorno operativo y su intercambio explícito por los
	mecanismos existentes de estado; no se introduce una segunda ubicación.
- Los IDs project-scoped con sufijos como `-this-project` se resuelven contra
	el ID canónico del catálogo para planificar y validar versiones.
- Los fallos de probe posterior no dejan un lockfile parcialmente actualizado:
	se restaura el checkpoint y, si no existía lockfile previo, se elimina el
	nuevo lockfile.

## Validación y review

- `npm run build`: pasa (`tsc -b`).
- `npx vitest run packages/toolchain/tests/versioning.test.ts`: pasa, 11/11.
- Tests de doctor afectados (`toolchain-versioning`, `checks`, `deep-checks`): pasan, 60/60.
- Tests de CLI afectados (`toolchain-versioning`, `toolchain`): pasan, 18/18.
- Review independiente `axiom-review`: ejecutado en solo lectura. Detectó y se
	corrigió el orden SemVer de prereleases numéricos (`beta.2 < beta.10`); se
	añadió además cobertura directa de `extractVersion()`.
- Suite global: la ejecución independiente del review reportó 328/330 archivos
	y 3425/3427 tests; fallan `packages/doctor/tests/agents.test.ts` y
	`packages/mcp-tools/tests/capability-routing-roundtrip.test.ts`, fuera del
	alcance de este incremento. Después se añadió una prueba focalizada de
	`extractVersion()`; los dos fallos se reprodujeron aisladamente.
- `npm run doctor`: falla únicamente en `TC-011` por `bundleHash` stale de
	`axiom-reviewer`; no se atribuye a este incremento.
- `npm run readiness:first-project`: falla porque su paso `doctor` hereda el
	mismo `TC-011`; los checks de toolchain no bloquean la ejecución cuando no
	existe lockfile en el proyecto temporal.
- Riesgo de cobertura residual no bloqueante: aún no hay un test que invoque
	`plan`/`upgrade` mediante Commander/argv real ni una simulación dedicada del
	fallo de `rename` en la escritura atómica. Los helpers, el build y los tests
	focalizados sí quedan verificados.

## General spec integration

La integración estable se realizó una sola vez en:

- `Axiom.Spec/specs/00_Resumen_Ejecutivo.md`
- `Axiom.Spec/specs/01_Requisitos_Funcionales.md`
- `Axiom.Spec/specs/02_Requisitos_No_Funcionales.md`
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md`
- `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`
- `Axiom.Spec/specs/05_Interfaces_Operativas.md`
- `Axiom.Spec/specs/06_Integraciones_y_Capacidades.md`
- `Axiom.Spec/specs/07_Gobierno_y_Seguridad.md`
- `Axiom.Spec/specs/08_Glosario.md`

Se actualizaron los contratos de catálogo schema 2, lockfile, canales,
planner, upgrade, probes, drift y los checks TC-020..TC-023. También se
reconciliaron claims activos de auto-validación, rutas `axiom.config/` y
`axiom.spec/`, manteniendo los resultados históricos bajo su contexto de
historia.

El contexto técnico actualizado comprende:

- `Axiom.Spec/context/architecture/02-modelo-de-datos-y-configuracion.md`
- `Axiom.Spec/context/operations/01-instalacion-y-onboarding.md`
- `Axiom.Spec/context/operations/02-doctor-troubleshooting-y-telemetria.md`
- `Axiom.Spec/context/integrations/01-capabilities-providers-y-toolchain.md`
- `Axiom.Spec/context/references/02-historial-de-incrementos.md`
- `Axiom.Spec/context/references/03-riesgos-y-brechas-conocidas.md`
- `Axiom.Spec/context/TECHNICAL_CONTEXT.md`

También se actualizó `Axiom.Spec/specs/adr/ADR-0032-toolchain-versioning/README.md`
para fijar el universo CLI frente al planner puro y documentar las fuentes de
catálogo y probes.

## Estado de cierre

El incremento ha sido resuelto y marcado como `closed`. Los bloqueos originales (el `TC-011` stale de `axiom-reviewer` y la incompatibilidad del fixture `axiom.*` en el schema de capabilities) han sido corregidos, por lo que la suite global, `npm run doctor` y `npm run readiness:first-project` ejecutan exitosamente. Procedemos a archivar el incremento.