# R-10 ACC-045 unify workflow resolution

> **Código**: INC-20260817-r10-acc045-unify-workflow-resolution
> **Estado**: Archivado mediante Core
> **Fecha de creación**: 2026-08-17
> **Tipo de cambio**: modificar

## Resumen

Hacer de `Axiom/axiom.config/workflows.yaml` la fuente declarativa canónica, derivar el default distribuible de ella y resolver configuración con un único contrato fail-closed.

## Contexto y motivación

El YAML, `DEFAULT_WORKFLOWS` y mapas locales de subcomandos se mantienen a mano. Distintos consumidores divergen ante YAML inválido: algunos silenciosamente usan default. Esto impide que CLI, launcher, MCP e integrate compartan el mismo grafo.

## Alcance

### Incluido

- Un único parser/resolvedor consumido por CLI, launcher, MCP e integrate.
- Default empaquetado o generado desde `axiom.config/workflows.yaml`, sin segunda representación editable.
- Semántica: YAML ausente → default; YAML válido → proyecto; YAML inválido/schema no soportado → error explícito, nunca default.
- Verificación de paridad entre subcomandos alcanzables y grafo declarado.
- Pruebas de ausencia, validez, error, schema futuro, override y paridad.

### Excluido

- Implementar el executor de transiciones, confirmación, archive o QA común (ACC-041/042/043).
- Introducir workflows nuevos o modificar el grafo salvo retirar efectos no ejecutables que pertenezcan a ACC-041.

## Decisiones funcionales cerradas

`workflows.yaml` del runtime es la fuente canónica; el default es un artefacto derivado y el resolver no hace fallback silencioso para configuración presente inválida.

## Consolidación en la spec general

Consolidar al final del lote el contrato estable: YAML canónico, default derivado y fallback solo por ausencia; el detalle de copiado del asset no pertenece a la spec de producto.

## Trazabilidad y fuentes

Plan R-10 ACC-045; `axiom.config/workflows.yaml`, `packages/workflow/src/workflow-resolution.ts`, CLI, launcher, MCP e integrate.

## Resultado y validación

Implementado: `DEFAULT_WORKFLOWS` parsea el asset YAML empaquetado por el build; el resolvedor único falla cerrado para YAML presente inválido o schema futuro. CLI, state/validate/integrate, launcher y MCP consumen el contrato. El launcher filtra transiciones eliminadas por un override y su API help se deriva del grafo bundled.

Validación independiente histórica: build de `@axiom/workflow` y tests de resolver/consumers pasaron; tras review se añadieron tests de launcher/parity. Esta evidencia pertenece a ACC-045 y no declara la resolución de compatibilidad de LiteLLM, aliases, MCP ni tipos fuera de su alcance.

## Estado de validación humana

No requiere decisión humana adicional; ACC-041 puede depender del resolvedor entregado.

## Evidencia actual de cierre

El Core archivó ACC-045 con la evidencia indicada en sus receipts. El estado de R-10 y la evidencia de compatibilidad posterior pertenecen al correctivo `INC-20260818-r10-closure-correction`; este README archivado no declara su cierre.
## Cierre Core

El Core archivó este incremento en `specs/increments/_archive/` mediante la cadena legal `specify → plan → plan-approve → verify → archive`; el receipt `2026-08-18T14-10-08.024Z-increment-archive-success.json` lo confirma. Las cifras que acompañaron ese archivado son evidencia histórica de ACC-045; no declaran el cierre de R-10 ni sustituyen la validación final del correctivo posterior.