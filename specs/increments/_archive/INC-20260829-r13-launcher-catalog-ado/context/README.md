# Context

## Propósito

Conservar el contrato técnico estable de catálogo, routing, identidad lifecycle y bridge ADO de R-13 sin duplicar la spec normativa ni registrar metadata estructural.

## Hechos estables reconciliados

- `ACTION_RECONCILIATION` es la fuente declarativa compartida por catálogo, routing y previews; los comandos publicados son invocaciones reales `axiom ...`.
- Un adapter desconocido no recibe fallback para acciones lifecycle declaradas; solo acciones sintéticas/no lifecycle pueden usar fallback clipboard explícito.
- Toda lane lifecycle exige ID caller-owned y rechaza ausencia/mismatch antes de receipt, transición o artefact mutation.
- ADO es local-first y opcional: el resultado local permanece independiente del resultado remoto; los tests usan bridge fake sin red.
- URL externa solo se convierte en enlace para `http:`/`https:` sin userinfo, query o fragment.

## Evidencia

La matriz conjunta ACC-076 ejecuta `ACC-073-01..03` y `ACC-074-01..02` dentro de 35 PASS, 0 FAIL, 0 TIMEOUT y 14 checks de no mutación. Fuentes runtime: `packages/launcher/src/action-catalog.ts`, `adapter-routing.ts`, wrappers lifecycle, `app-launcher-ado.ts` y `_external-url.ts`.

## Qué no debe vivir aquí

No deben vivir metadata, status, IDs, links, receipts manuales, índices generados, secretos ni la spec normativa completa. El lifecycle se gestiona con Axiom Core.
