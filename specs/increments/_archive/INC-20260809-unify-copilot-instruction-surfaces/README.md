# Unificar superficies de instrucciones de Copilot

> **Código**: INC-20260809-unify-copilot-instruction-surfaces
> **Estado**: Archivado
> **Fecha de creación**: 2026-08-09
> **Tipo de cambio**: normalizar rutas, contenido y preservación de instrucciones generadas

## Objetivo

Hacer que `copilot-vscode` y `github-copilot` produzcan una superficie de instrucciones coherente y compatible con las rutas oficiales de GitHub Copilot y VS Code. La instrucción general del repositorio debe vivir en `.github/copilot-instructions.md`; `.vscode/` queda para configuración y MCP; las instrucciones específicas por ruta deben vivir en `.github/instructions/*.instructions.md`.

## Contexto y motivación

R-06 verificó tres comportamientos divergentes: `workspace setup` emitía `.vscode/copilot-instructions.md`, `configure` escribía `.github/copilot-instructions.md` mediante `@axiom/document-bootstrap` y `sync` usaba el generador dedicado de GitHub Copilot, que sobrescribía el archivo completo sin conservar `TEAM:CUSTOM`. La documentación oficial de GitHub Copilot y VS Code reconoce `.github/copilot-instructions.md` para instrucciones generales y `.github/instructions/*.instructions.md` para instrucciones específicas; no se verificó `.vscode/copilot-instructions.md` como ubicación oficial de instrucciones.

## Alcance

- Centralizar la plantilla canónica de Copilot y su fallback bundleado en `@axiom/document-bootstrap`.
- Hacer que `configure`, `sync`, `workspace setup` y el adapter `github-copilot` usen el mismo contrato de render y escritura.
- Mantener `.github/copilot-instructions.md` como ruta única para las instrucciones generales de ambos targets.
- Reservar `.vscode/settings.json`, `.vscode/extensions.json` y `.vscode/mcp.json` para superficies propias de VS Code.
- Mover las superficies específicas de `copilot-vscode` a `.github/instructions/<id>.instructions.md`.
- Regenerar solo el bloque `AXIOM:GENERATED` y preservar byte a byte `TEAM:CUSTOM` y el contenido humano fuera de ese bloque.
- Migrar de forma explícita una salida legacy existente en `.vscode/copilot-instructions.md`, sin perder personalizaciones y sin dejar dos instrucciones generales activas.
- Alinear el registry, los contadores de archivos, los README/manuales y las pruebas.

## Fuera de alcance

- Cambiar la semántica de MCP: `.vscode/mcp.json` sigue siendo independiente.
- Cambiar el contenido de las superficies portables `.axiom/{agents,commands,skills}`.
- Añadir nuevos targets o soporte específico no verificado para otros IDEs.
- Eliminar el target `github-copilot` o fusionar sus nombres con `copilot-vscode`.
- Reescribir historia archivada salvo referencias activas que describan el comportamiento vigente.

## Decisiones funcionales

- La plantilla activa puede venir de `axiom.spec/templates/copilot-instructions.template.md`; si falta o no se puede leer, se usa la copia bundleada única.
- Ambos targets escriben el mismo documento general y conservan el mismo formato de marcadores. El valor `target.id` puede identificar el target dentro del bloque generado, pero no cambia la ubicación ni el contrato de preservación.
- `github-copilot` mantiene `generateCopilotConfig(args)` y su resultado público; el cambio de implementación no obliga a los consumidores a conocer `@axiom/document-bootstrap`.
- La migración legacy es conservadora: si existe `.vscode/copilot-instructions.md`, se toma como fuente humana, se incorpora a `.github/copilot-instructions.md` con la misma preservación de marcadores y se elimina la copia legacy solo después de escribir correctamente el destino. Si el destino ya existe, se conserva el destino como fuente canónica y se evita borrar contenido no fusionable; el resultado debe advertir del conflicto.
- Si no hay marcadores en una copia legacy, todo su contenido se conserva como preámbulo humano y el writer añade los marcadores canónicos para futuras regeneraciones.

## Criterios de aceptación

- [x] `workspace setup` con `copilot-vscode` escribe `.github/copilot-instructions.md` y no crea `.vscode/copilot-instructions.md`.
- [x] `configure` y `sync` para `copilot-vscode` y `github-copilot` convergen en `.github/copilot-instructions.md` y no generan dos documentos generales.
- [x] `.vscode/settings.json`, `.vscode/extensions.json` y `.vscode/mcp.json` continúan funcionando y no se confunden con instrucciones.
- [x] Las superficies específicas de `copilot-vscode` se escriben en `.github/instructions/*.instructions.md`.
- [x] Los dos targets usan la plantilla/resolución bundleada común y un contenido generado compatible.
- [x] Una regeneración reemplaza el bloque `AXIOM:GENERATED`, conserva byte a byte `TEAM:CUSTOM` y conserva el contenido humano fuera de los marcadores.
- [x] Una copia legacy se migra de forma atómica, idempotente y sin pérdida; no queda una salida legacy activa cuando la migración tiene éxito.
- [x] El registry y la documentación declaran únicamente los archivos que realmente se escriben.
- [x] Las pruebas cubren cold start, convergencia entre entrypoints, migración, conflicto, preservación, idempotencia y ausencia de `.vscode/copilot-instructions.md`.
- [x] `npm run build` y las suites focalizadas de document-bootstrap, adapters, configure, sync, workspace setup y process surfaces pasan.
- [x] La revisión independiente confirma que no se reintroduce una ruta no oficial ni una escritura completa que destruya personalizaciones.

## Fuentes

- `Axiom.Spec/plans/PLAN-REVISION-INTEGRAL-AXIOM.md` — ACC-024 y hallazgos R-06.
- `Axiom/packages/document-bootstrap/src/{writer,idempotency,variables}.ts`.
- `Axiom/packages/adapters/github-copilot/src/{generator,instructions,types}.ts`.
- `Axiom/packages/cli-commands/src/commands/{configure,sync,workspace-adapter-templates}.ts`.
- `Axiom/apps/cli/src/commands/{workspace-adapters,workspace-process-surfaces}.ts`.
- Documentación oficial de GitHub Copilot y VS Code sobre custom instructions.

## Validación prevista

## Notas de implementación

- `@axiom/document-bootstrap` es la única fuente bundleada del template de Copilot y de su resolución on-disk -> bundle.
- `writeCopilotInstructions` reemplaza únicamente `AXIOM:GENERATED`, conserva el contenido humano y `TEAM:CUSTOM` byte a byte, escribe atómicamente y migra `.vscode/copilot-instructions.md` con warning conservador ante conflicto.
- `github-copilot`, `copilot-vscode`, `configure`, `sync` y `workspace setup` convergen en el writer común; `copilot-vscode` conserva `settings.json`, `extensions.json` y `mcp.json` bajo `.vscode` y mueve sus process surfaces a `.github/instructions`.
- Se preservaron las APIs públicas razonables y se actualizaron los conteos/resultados, registry y documentación operativa. No se modificaron MCP ni superficies `.axiom/*`.

## Validación

- `npm run build`: PASS.
- `npx vitest run packages/document-bootstrap/tests packages/adapters/github-copilot/tests apps/cli/tests/document-bootstrap.test.ts apps/cli/tests/workspace-adapters.test.ts apps/cli/tests/workspace-process-surfaces.test.ts apps/cli/tests/sync.test.ts packages/cli-commands/tests packages/installer/tests/installer.test.ts apps/cli/tests/native-mcp-config.test.ts`: PASS, 13 archivos y 146 tests.
- Barrido de referencias activas: no quedan declaraciones de `.vscode/copilot-instructions.md` en runtime, registry o documentación; las únicas coincidencias restantes son el plan de auditoría congelado y los tests que verifican la migración legacy.
- `npm install --package-lock-only --ignore-scripts --no-audit --no-fund`: PASS; el lockfile incluye `@axiom/document-bootstrap` para `@axiom/adapters-github-copilot`.
- `npm ci --dry-run --ignore-scripts`: PASS; no se detectaron incompatibilidades de dependencias.
- Review independiente final: OK, sin blockers. Se corrigió además el caso de cola humana divergente para no borrar una fuente legacy con contenido no fusionado.

## Estado

Status: closed

La implementación, la validación, la revisión independiente y la integración
de conocimiento estable en `Axiom.Spec/specs/00..08` y `context/**` están
completas. La plantilla bundleada y las copias on-disk son idénticas en el
checkout actual. El incremento queda archivado físicamente en
`specs/increments/_archive/`.
