# r11-mcp-documentation

> **Código**: INC-20260819-r11-mcp-documentation
> **Estado**: Gestionado por Axiom/Core; consultar `metadata.yml` para el estado de lifecycle.
> **Fecha de creación**: 2026-08-20
> **Tipo de cambio**: documentar

## Resumen

Reconcilia la documentación activa de MCP con el runtime: Axiom gestiona un único broker project-scoped, `axiom-mcp-broker`, iniciado exclusivamente por `axiom mcp serve --kind axiom`.

## Contexto y motivación

ACC-046 de R-11 detectó que código, configuración y pruebas ya convergen en el broker único, mientras partes del plan y comentarios históricos aún presentan `sdd-mcp-server`, `spec-mcp-broker` o el kind `memory` como procesos activos.

## Alcance

### Incluido

- Corregir comentarios y documentación operativa activa del runtime que contradigan el broker único.
- Reconciliar los claims activos de `Axiom.Spec/specs/03`, `05` y `06`, el contexto técnico pertinente y el registro del plan de revisión.
- Distinguir explícitamente historia/IDs de limpieza de procesos activos, y describir Engram como MCP local de tercero y backend opcional de memoria.
- Preservar los artefactos archivados y receipts como historia inmutable.

### Excluido

- Cambiar el protocolo MCP, el conjunto de herramientas registrado o el routing de providers.
- Reescribir receipts, metadata estructural o documentación histórica archivada.
- Introducir brokers, MCPs obligatorios o integraciones externas.

## Documentos del incremento

- `01_Requisitos.md`: contrato documental verificable.
- `02_Cambios_Modelo.md`: superficies y compatibilidad.
- `03_Criterios_Aceptacion.md`: criterios y validación.
- `04_Interacciones_UI.md`: sin interfaz nueva.

## Dudas abiertas

Ninguna bloqueante. La revisión independiente confirmó el binding efectivo `.axiom/mcp.yml`, el protocolo `server/discover`, el registro completo de 25 handlers y el despacho directo a `invokeMcpTool`.

## Decisiones funcionales cerradas

1. Solo `axiom-mcp-broker` es un proceso MCP gestionado de Axiom.
2. `sdd-mcp-server`, `spec-mcp-broker` y `memory` solo pueden aparecer como historia o IDs de limpieza compatibles.
3. Engram no es broker gestionado por Axiom.
4. Se conservaron correcciones textuales en writers/plantillas bajo `apps/cli/src/commands/` porque materializan documentación operativa; no alteran protocolo, handlers, registro de tools ni semántica CLI.

## Consolidación en la spec general

Se releyeron `specs/03`, `05` y `06` y `context/integrations/01-capabilities-providers-y-toolchain.md`: ya expresan el broker único, kind `axiom`, los 25 handlers, `.axiom/mcp.yml` y Engram opcional. No requieren reescritura. El conocimiento estable nuevo se integra al plan como supersesión explícita de la afirmación histórica de tres brokers.

## Estrategia E2E

No hay UI nueva. Se ejecutó la batería MCP del runtime (23 archivos, 165 tests) y `npm run build`; la segunda review independiente quedó sin desvíos funcionales.

## Trazabilidad y fuentes

ACC-046 del `PLAN-REVISION-INTEGRAL-AXIOM.md`; `packages/mcp-server`, `packages/mcp-tools`, `apps/cli/src/commands/{mcp-serve,workspace-mcp,native-mcp-config}.ts`, las pruebas MCP focalizadas, `mcp-manifest.yaml`, `integrations.yaml` y `.axiom/mcp.yml`.

## Estado de validación humana

Implementación terminada. Validación independiente: 23 archivos/165 tests MCP PASS y `npm run build` PASS. Review independiente re-ejecutada: sin blockers funcionales; la consolidación documental y el cierre Core se realizan en esta fase.
