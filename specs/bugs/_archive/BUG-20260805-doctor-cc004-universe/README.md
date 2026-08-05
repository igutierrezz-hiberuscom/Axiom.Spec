# Bug: `CC-004` construye su universo desde los mismos providers

Status: closed
Date: 2026-08-05

## Symptom

Doctor informa `CC-004 PASS` aunque existan capabilities canónicas sin ningún
provider que las sirva.

## Current behavior

`runCapabilityModelChecks` construye `allCapabilities` recorriendo las
capabilities ya declaradas por los providers. Después busca un provider para
cada elemento de ese mismo conjunto. Una capability que no aparece en ningún
provider nunca entra en la comprobación.

En la auditoría inicial se observaban 19 ids en el conjunto combinado, pero
solo 7 ids servidos por providers; 12 no aparecían en el registry de providers.
A pesar de ello `CC-004` pasaba. Tras separar las tres capabilities MCP-only,
la cobertura provider-routed vigente se expresa sobre 16 ids.

## Expected behavior

El universo que comprueba `CC-004` debe venir de una fuente independiente de
la lista de providers, de forma que una capability canónica sin provider entre
en el diagnóstico. La política exacta para capacidades MCP-only, opcionales y
requeridas queda definida en el incremento de cobertura asociado.

## Impact

Doctor puede dar un PASS falso y ocultar que el resolver no encontrará un
provider para una capability que el modelo o un perfil considera disponible.

## Reproduction steps

1. Crear un proyecto con un provider que solo declare `sdd.workflow`.
2. Mantener `sdd.commandResolution` en el conjunto canónico.
3. Ejecutar `runCapabilityModelChecks`.
4. Observar que el código actual no reporta `sdd.commandResolution` como
   ausente porque nunca lo añade a `allCapabilities`.

## Suspected cause

El código de `CC-004` usa el conjunto observado en providers como universo de
referencia en lugar de comparar contra el catálogo canónico o la política de
cobertura declarada.

- [x] `CC-004` ya no comprueba un conjunto construido exclusivamente a partir
      de los providers.
- [x] Una fixture adversarial con una capability canónica sin provider deja de
      producir un PASS falso.
- [x] El diagnóstico identifica el id ausente y conserva evidencia útil.
- [x] La clasificación final de severidad respeta la política del incremento
      de cobertura de capabilities.
- [x] Las comprobaciones CC-001, CC-002, CC-003, CC-005 y CC-006 mantienen su
      comportamiento.

## Fix notes

Este bug se ejecuta después del incremento que formaliza la superficie
MCP-only y antes del incremento que consolida la política completa de
cobertura. No debe inventar excepciones específicas de `axiom.*`.

## Validation

- `npx vitest run packages/doctor/tests/capability-model.test.ts`: 21
      pruebas pasaron, incluyendo requerida, opcional, post-MVP, disabled,
      unavailable, capability ausente, MCP-only y YAML ausente/inválido.
- `npm run build`: pasó (`tsc -b`).
- Doctor real: `CC-004` devuelve `fail` con `Canónicas: 16`, `Servidas: 7`,
      seis requeridas y tres opcionales sin provider.
- `get_errors`: sin errores en la superficie revisada.
- `git diff --check`: sin errores en el diff focalizado.
- Receipt `verify` final: `receipts/2026-08-05T16-05-39.522Z-verify-success.json`,
  hash `2c6cce33a023ca03003976bfe358813b0608762c3a15f9b0f8f862e62893ce9c`.

## Result

La implementación está verificada y la integración canónica quedó consolidada
en la spec y el contexto técnico de R-04.

## General spec integration

Integrado en la pasada única de R-04:

- `specs/01_Requisitos_Funcionales.md`: `CC-004` debe usar el catálogo
      provider-routed canónico y distinguir capabilities sin declarar de providers
      ausentes.
- `specs/02_Requisitos_No_Funcionales.md`: no se permite un PASS falso de
      cobertura; la severidad depende de cumplimiento y estado.
- `specs/06_Integraciones_y_Capacidades.md`: estado real de cobertura 7/16 y
      exclusión de las tres capabilities MCP-only.
- `specs/07_Gobierno_y_Seguridad.md`: regla de severidad y evidencia de
      `CC-004`.
- `context/operations/02-doctor-troubleshooting-y-telemetria.md`: diagnóstico
      operativo y comportamiento ante `capabilities.yaml` ausente o inválido.

El artefacto queda archivado por el autopilot.
