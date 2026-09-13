# 02 Cambios de Modelo

## Objetivo del documento

Detallar los cambios en los artefactos de metadatos de versión.

## Nuevos archivos generados en build

- `packages/versioning/src/generated-version.ts`:
  ```typescript
  export const GENERATED_RUNTIME_PACKAGE = '@axiom/versioning' as const;
  export const GENERATED_RUNTIME_VERSION = '0.1.0' as const;
  export const GENERATED_RELEASE_TAG = null as string | null;
  export const GENERATED_COMMIT = null as string | null;
  export const GENERATED_BUILD_ID = 'axiom-cli-0.1.0' as string;
  export const GENERATED_IS_DEV = true as boolean;
  ```

## Módulos modificados

- `packages/versioning/src/version.ts`:
  Reemplaza el literal estático por la exportación de las constantes provistas por `generated-version.ts`.
- `scripts/generate-cli-build-metadata.mjs`:
  Extendido para emitir `generated-version.ts` en paralelo con `generated-build-metadata.ts`.
