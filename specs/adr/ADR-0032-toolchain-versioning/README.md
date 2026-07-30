# ADR-0032: Versionado reproducible de toolchains externas

- **Estado**: aceptado para el incremento `INC-20260730-toolchain-versioning`
- **Fecha**: 2026-07-30
- **Alcance**: `@axiom/toolchain`, `@axiom/doctor` y `axiom toolchain`

## Contexto

Axiom declara herramientas externas en `axiom.config/toolchain.yaml`, pero la
versión instalada no formaba parte de un estado operativo reproducible. Un
marker de filesystem tampoco demuestra que el binario funcione, y el catálogo
no ofrecía un canal explícito para planificar cambios.

## Decisión

1. El catálogo usa schema 2 y puede declarar `versionExtractor` y versiones
   por canal `stable`, `candidate` y `edge`.
2. El estado fijado se almacena en
   `.axiom-state/<project>/toolchain.lock` con escritura atómica. El lockfile
   es estado local generado y permanece ignorado por Git junto con el resto de
   `.axiom-state/`.
3. `toolchain plan` compara el lockfile con el catálogo para el conjunto que
   la CLI resuelve como declarado o lockeado, admite `--id` repetido y no
   escribe nada. La función pura del planner, si no recibe un subconjunto
   explícito, usa las tools ya lockeadas; la mera presencia de una entrada en
   el catálogo no genera una alta implícita.
4. `toolchain upgrade` solo actualiza el lockfile; no instala ni reemplaza
   binarios externos. Requiere `--yes`; sin esa confirmación, o con
   `--dry-run`, solo muestra el plan.
5. El upgrade crea un checkpoint del lockfile y restaura el estado anterior si
   falla la persistencia o la verificación posterior. Si no existía lockfile,
   rollback elimina el lockfile recién creado.
6. `doctor` mantiene TC-020..TC-023 en la ruta síncrona. La comparación real de
   versión instalada contra lockfile se ejecuta de forma opt-in en
   `doctor --deep`, mediante probes inyectables en tests.
7. Los IDs project-scoped, por ejemplo `serena-this-project`, se resuelven
   contra el ID canónico `serena` del catálogo.

## Consecuencias

- El lockfile aporta una representación estable del estado que Axiom intentó
  fijar, pero no sustituye la instalación ni garantiza por sí mismo que un
  binario siga presente.
- Las versiones del catálogo son baselines declaradas por la política de
  Axiom. No se consideran evidencia de una release upstream hasta verificarlas
  contra la fuente de distribución correspondiente.
- Un probe que devuelve una versión distinta bloquea la aplicación del upgrade
  y conserva el lockfile anterior.
- Las tools sin contrato local de probe pueden permanecer declaradas, pero no
  se presentan como `installed-working`.

## No decidido aquí

- Instalación automática, mirrors, side-by-side installs y rollback de
  binarios.
- Descubrimiento automático de releases upstream.
- Matrices firmadas de compatibilidad, SBOM o provenance.

## Fuentes

- `Axiom/packages/toolchain/src/lockfile.ts`
- `Axiom/packages/toolchain/src/catalog.ts`
- `Axiom/packages/toolchain/src/probe.ts`
- `Axiom/packages/toolchain/src/plan.ts`
- `Axiom/packages/toolchain/src/upgrade.ts`
- `Axiom/apps/cli/src/commands/toolchain.ts`
- `Axiom/packages/doctor/src/checks.ts`
- `Axiom/packages/doctor/src/deep-checks.ts`
