# 02 Cambios de Modelo

## Objetivo del documento

Detallar las interfaces y contratos del aviso de recomendación de modelo.

## Tipos e interfaces

- En `@axiom/model-routing`:
  - `ModelRecommendation`:
    ```typescript
    export interface ModelRecommendation {
      readonly slot: SlotId;
      readonly recommendedClass: ModelClass;
      readonly effectiveClass: ModelClass;
      readonly supportLevel: SupportLevel;
      readonly notice?: string;
    }
    ```
  - Función `formatModelNotice(target: string, slot: SlotId, recommendedClass: ModelClass): string | undefined`.

- En `@axiom/launcher`:
  - `CraftPromptOptions` y `craftPrompt`:
    Integra el aviso en el header del prompt generado cuando la acción tiene un `SlotId` asociado y el adapter no es `multi-mode`.
