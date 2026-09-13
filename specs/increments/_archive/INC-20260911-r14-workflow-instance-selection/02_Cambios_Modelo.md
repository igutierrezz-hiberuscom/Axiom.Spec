# 02 Cambios de Modelo

## Objetivo del documento

Detallar los cambios en las estructuras y tipos de `@axiom/workflow` para soportar `schemaVersion: 2` con múltiples instancias por workflow.

## Entidades o estructuras afectadas

- `WorkflowStateFile`:
  Pasa a soportar `schemaVersion: 2`. La sección `workflows` puede contener `WorkflowInstanceContainer` para workflows de artefacto:
  ```typescript
  export interface WorkflowInstanceContainer {
    readonly workflowId: string;
    readonly activeInstanceId?: string;
    readonly instances: Readonly<Record<string, WorkflowStateRecord>>;
  }
  ```
- `WorkflowStateRecord`:
  Mantiene `workflowId`, `state`, `vars` y `updatedAt`.
- Entrada de workflows de workspace:
  Sigue soportando `WorkflowStateRecord` directo para `role` y `qa-e2e`.

## Contratos o estados afectados

- `loadWorkflowState(projectRoot, workflowId, instanceId?)`:
  Permite resolver el estado para una instancia concreta. Si no se pasa `instanceId`, resuelve `activeInstanceId`.
- `saveWorkflowState(projectRoot, record, options?)`:
  Persiste el record en el slot de su instancia (`vars.metadataId` o `vars.id`), y actualiza `activeInstanceId`.
- `purgeWorkflowInstance(projectRoot, workflowId, instanceId)`:
  Remueve la instancia de `instances` al archivar.

## Notas de compatibilidad

- Migración automática transparente al leer cualquier `workflow-state.json` con `schemaVersion: 1`.
- Serialización siempre en `schemaVersion: 2`.
