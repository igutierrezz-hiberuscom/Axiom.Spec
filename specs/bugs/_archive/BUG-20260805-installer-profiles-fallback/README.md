# Bug: `installProfile` oculta errores de `profiles.yaml`

Status: closed
Date: 2026-08-05

## Symptom

`installProfile` usa `DEFAULT_PROFILES` cuando `profiles.yaml` no existe,
pero también cuando el archivo existe y no se puede leer o contiene YAML
inválido. El usuario puede recibir un perfil válido con valores distintos de
los que intentó configurar.

## Current behavior

Antes de la corrección, `loadProfilesData` diferenciaba internamente los
errores de lectura y parseo, pero `packages/installer/src/installer.ts`
convertía cualquier resultado fallido en `DEFAULT_PROFILES`. Un archivo
ausente, un archivo ilegible y un YAML malformado producían el mismo resultado
silencioso.

Los consumidores `roles` y `topology` ya son fail-closed para un archivo
presente inválido; `installProfile` no mantiene esa misma regla.

## Expected behavior

- Si `axiom.config/profiles.yaml` no existe, `installProfile` debe usar
  `DEFAULT_PROFILES` para conservar el onboarding de proyectos nuevos.
- Si el archivo existe pero es ilegible, el YAML es inválido o su estructura
  no cumple el contrato, `installProfile` debe devolver un error explícito.
- Un archivo válido del proyecto debe seguir teniendo prioridad sobre el
  fallback bundleado.
- No debe escribirse ni repararse automáticamente `profiles.yaml`.

## Impact

`configure`, `start` o cualquier consumidor que derive el perfil puede
continuar con una configuración diferente de la elegida por el usuario sin
mostrar una advertencia. Hoy no hay proyectos existentes que dependan de este
comportamiento, por lo que la corrección no tiene riesgo de compatibilidad
operativa conocido.

## Reproduction steps

1. Crear un directorio temporal con un `profiles.yaml` ausente y ejecutar
   `installProfile`: debe funcionar con el default.
2. Crear `profiles.yaml` como directorio o sin permisos de lectura y ejecutar
   `installProfile`: actualmente también funciona con el default.
3. Escribir YAML malformado y ejecutar `installProfile`: actualmente también
   funciona con el default.
4. Escribir un YAML parseable pero estructuralmente incompleto: observar que
   el error puede aparecer tarde como `composition-failed`, en lugar de un
   error de configuración uniforme.

## Suspected cause

El caller descarta el `InstallerError` devuelto por `loadProfilesData` con la
expresión `loadResult.ok ? loadResult.value : DEFAULT_PROFILES`, sin distinguir
`file-not-found` de los demás fallos.

## Acceptance criteria

- [x] El archivo ausente conserva el fallback a `DEFAULT_PROFILES`.
- [x] Un error de lectura de un archivo presente devuelve un error y no usa el
      fallback.
- [x] Un YAML malformado devuelve un error y no usa el fallback.
- [x] Un YAML estructuralmente inválido devuelve un error claro antes de la
      composición.
- [x] Un YAML válido personalizado sigue teniendo prioridad.
- [x] `installProfile` no crea ni modifica `profiles.yaml`.
- [x] Existen pruebas focalizadas para los cuatro estados: ausente, ilegible,
      YAML inválido y estructura inválida.

## Fix notes

La corrección distingue la ausencia legítima del archivo de cualquier fallo
de un archivo presente. `loadProfilesData` converge sobre el
`InstallProfilesYamlSchema` existente; `installProfile` solo usa
`DEFAULT_PROFILES` cuando la ruta no existe y devuelve el error en los demás
casos.

## Validation

- `npx vitest run packages/installer/tests/installer.test.ts`: 12 pruebas
   pasaron, incluyendo ausencia, ilegibilidad, YAML inválido, estructura
   inválida, prioridad del override e inmutabilidad del archivo.
- `npx vitest run apps/cli/tests/init.test.ts apps/cli/tests/configure.test.ts
   apps/cli/tests/roles.test.ts apps/cli/tests/topology.test.ts`: 4 archivos y
   52 pruebas pasaron.
- `npm run build`: pasó (`tsc -b`).
- `get_errors`: sin errores en la superficie revisada.
- `git diff --check`: sin errores en el diff focalizado.
- Receipt `verify` final: `receipts/2026-08-05T16-05-39.521Z-verify-success.json`,
  hash `0fea68231aa7b53ee6743374dbaf2f7505f13f6ac2580790191be702f958db7d`.

## Result

El fallback ahora se aplica únicamente cuando `profiles.yaml` está ausente.
Un archivo presente ilegible, malformado o estructuralmente inválido devuelve
`invalid-profiles-yaml`; un archivo válido sigue teniendo prioridad y no se
modifica. La semántica quedó integrada en la spec general y el contexto
técnico.

## General spec integration

Integrado en la pasada única de R-04:

- `specs/03_Modelo_Operativo_y_Datos.md`: el fallback de `installProfile` se
   limita a la ausencia de archivo; los errores de archivos presentes son
   visibles.
- `specs/01_Requisitos_Funcionales.md`: requisito fail-closed para
   `installProfile` junto al contrato de validación de perfiles.
- `context/operations/01-instalacion-y-onboarding.md`: comportamiento visible
   de `configure` ante perfiles ausentes o inválidos.
- `context/architecture/02-modelo-de-datos-y-configuracion.md`: forma
   validada de `profiles.yaml`.
- `context/**`: sin otros cambios requeridos por este bug.

El artefacto queda archivado por el autopilot.
