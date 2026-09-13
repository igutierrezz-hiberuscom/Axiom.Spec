# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos para unificar la fuente de verdad de la versión de runtime (`ACC-085`), eliminando el literal cableado a mano en `packages/versioning/src/version.ts` y derivándolo de la identidad de producto generada en build.

## Requisitos del incremento

- **REQ-085-01: Generación determinista de identidad en `@axiom/versioning`**
  `scripts/generate-cli-build-metadata.mjs` debe generar, además de la metadata del CLI, el módulo `packages/versioning/src/generated-version.ts` conteniendo la versión canónica de producto derivada de `package.json`, release tag, commit, buildId e indicador de desarrollo.

- **REQ-085-02: Consumo en `@axiom/versioning`**
  `packages/versioning/src/version.ts` debe importar la versión de `generated-version.ts`, eliminando el literal hardcodeado `RUNTIME_VERSION = '0.1.0'`.

- **REQ-085-03: Coherencia global de identidades**
  Asegurar que `axiom --version`, `ManagedState.runtime.version`, el manifest de instalación y el valor por defecto de `--target-version` en `axiom upgrade` compartan deterministamente la misma identidad sin divergencias.

- **REQ-085-04: Compatibilidad con estados persistidos en `0.1.0`**
  Preservar el comportamiento de migraciones y compatibilidad para proyectos con `managed-state.json` ya persistidos con versión `0.1.0`.

- **REQ-085-05: Comportamiento sin release publicada**
  Cuando no exista tag de release (`AXIOM_BUILD_RELEASE_TAG`), la versión utiliza el SemVer de `package.json` marcando explícitamente `releaseTag: null` y `isDev: true`.

## Reglas de negocio relevantes

- No se colapsa la separación de ejes: la versión del CLI global y la versión project-scoped del `ManagedState` siguen siendo conceptos formalmente independientes, pero derivados de la misma autoridad.
- La lectura se realiza en tiempo de compilación; ningún módulo lee `package.json` en caliente con I/O síncrono frágil en producción.
