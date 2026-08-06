# Incremento: reconciliacion documental builder/local-only

Status: closed
Date: 2026-08-06
Requested work item: ACC-019
Depends on: INC-20260805-225413-6fx4hs
Runtime repository: `Axiom/`
Canonical spec repository: `Axiom.Spec/`

## Goal

Alinear toda la documentacion activa y las plantillas materializables con el
runtime vigente: perfil funcional `builder` implicito, politica operativa
`local-only` implicita, providers locales, audit trail transversal y brokers
MCP reales. Las superficies retiradas solo pueden aparecer en historia
explicita o en notas de compatibilidad legacy.

## Scope

### Incluido

- Actualizar manuales y README activos bajo `Axiom/docs/` para que `init`,
  `start`, `doctor`, perfiles, providers, telemetry, troubleshooting,
  installation, usage y readiness no indiquen flags, comandos o estados
  retirados.
- Actualizar templates y baseline product-owned bajo `Axiom/axiom.spec/` que
  se materialicen en proyectos, especialmente install-profile,
  getting-started, discovery providers, product skills y telemetry.
- Corregir hechos documentales: `structural-code-intel` es valido para cmm;
  `installProfile` usa defaults solo cuando profiles.yaml falta; el modelo
  separa 16 capabilities provider-routed de 3 MCP-only; `CC-004` distingue
  requeridas, opcionales e inactivas y en Axiom queda warning por 13/16
  servidas; el audit trail usa P365D; MCP no es gateway.
- Marcar como historical/legacy las referencias inevitables a estados antiguos
  en vez de presentarlas como opciones activas.

### Excluido

- Cambiar comportamiento de runtime o configuracion ejecutable; eso pertenece
  al increment runtime dependiente.
- Editar durante el apply los canónicos `Axiom.Spec/specs/00..08` o
  `Axiom.Spec/context/**`; el autopilot los integrara una sola vez al final.
- Cambiar el workflow SDD, el launcher, MCP handlers, providers o tests.
- Reescribir historia de incrementos/bugs archivados.

## Acceptance criteria

- [ ] Ningun manual o README activo anuncia `--profile`, `--overlay`,
      `--gateway`, `--no-gateway`, `axiom gateway`, `gateway-state.json`,
      `standard`, `enterprise`, `product-owner` o `generated-snapshots` como
      superficie vigente.
- [ ] Las plantillas activas materializan builder/local-only implicitos y no
      contienen selects o tablas de perfiles/overlays retirados.
- [ ] La documentacion explica los 10 targets, los roles de equipo/fases como
      ejes distintos, los cuatro providers locales y los tres brokers MCP.
- [ ] La documentacion explica el audit trail append-only, sidecar SHA-256,
      `axiom audit` read-only y retencion P365D sin gate enterprise.
- [ ] La cobertura de capabilities documenta 16 provider-routed + 3 MCP-only;
      `CC-004` trata tres opcionales sin provider como warning en Axiom, no como
      un fallo oculto ni como soporte inventado.
- [ ] La validacion documental (`git diff --check` y assertions focalizadas o
      grep de claims retirados) pasa.

## Assumptions and decisions

- Las menciones en `Axiom.Spec/specs/increments/_archive/**`, bugs archivados
  y el historial de este plan se conservan; no son claims activos.
- Los filtros de compatibilidad de runtime pueden mencionarse solo como
  migracion de estados legacy, no como seleccion operativa.
- El texto `enterprise` usado como adjetivo general de producto se revisa para
  no confundirlo con el overlay retirado.

## Implementation notes

- Se actualizaron únicamente manuales activos bajo `Axiom/docs/` y baseline
  materializable bajo `Axiom/axiom.spec/`.
- La documentación ahora presenta `builder` y `local-only` como implícitos,
  elimina selectors y gateway, y conserva el nombre físico
  `local-overlay-policy.yaml` solo como referencia de compatibilidad de ruta.
- Se documentaron los diez targets, los siete materializadores actuales de
  `sync`, los cuatro providers locales, las 16 capabilities provider-routed,
  las 3 MCP-only, el warning honesto de `CC-004`, los tres brokers MCP y el
  audit trail transversal.
- No se editaron runtime, configuración ejecutable, handlers, tests, workflow,
  canónicos `Axiom.Spec/specs/00..08`, contexto ni archivos archivados.

## Validation

Aplicado y validado localmente; review independiente e integración única del
autopilot completadas.

- `git diff --check -- docs axiom.spec`: pasa sin errores; Git solo reporta
  avisos informativos de normalización LF/CRLF en archivos ya tocados.
- Assertion focalizada de superficies retiradas en `docs/**` y
  `axiom.spec/**`: no quedan `--profile`, `--overlay`, `--gateway`,
  `--no-gateway`, `axiom gateway`, `gateway-state.json`, `standard`,
  `enterprise`, `product-owner` ni `generated-snapshots` activos.
- Assertion focalizada de profile triple y terminología de overlay: sin
  referencias activas a `profile triple`, `functionalProfile`,
  `operationalOverlay`, `profileTriple` u `overlay activo`.
- Assertion de hechos obligatorios: confirma builder/local-only implícitos,
  diez targets, los cuatro providers, `structural-code-intel`, 16 + 3
  capabilities, `CC-004`, audit trail, `P365D` y los tres brokers MCP.
- `js-yaml` parsea las seis plantillas YAML activas comprobadas, incluida
  `increment-metadata-template.yaml`.
- Comprobación de enlaces relativos Markdown: 119 archivos sin destinos
  inexistentes.
- La ayuda compilada de la CLI confirma que `init` ya no expone selectors de
  profile/overlay, `start` es filesystem/local-only y no existe el comando
  gateway.
- Validación runtime posterior a la integración: `npm run build`,
  `npx vitest run` (326 archivos/3313 pruebas), `npm run doctor` (PASS, 45/60,
  0 fallos) y `npm run readiness:first-project` (PASS).

El apply documental no modificó handlers ni tests; la validación final del
lote sí ejecutó build, suite completa, doctor y readiness tras la integración
canónica.

## Result

Apply documental completado para ACC-019. Los manuales activos y las
plantillas materializables describen `builder` y `local-only` implícitos,
eliminan selectors y gateway, enumeran los diez targets, separan roles de
equipo y fases SDD, y documentan providers, capabilities, MCP y audit trail
según el runtime vigente.

Los seis criterios de aceptación quedan satisfechos y el incremento queda
`closed` tras la review independiente y la integración canónica única.

Riesgos residuales: `sync` materializa hoy siete de los diez targets; los otros
tres siguen disponibles en el registry y quedaron documentados como targets
sin rama de sync. Los cuatro nombres de YAML retirados de los enlaces no se
crearon porque no existen en la baseline ejecutable actual; sus manuales ahora
declaran explícitamente ese límite.

## General spec integration

Integrada en una única pasada del autopilot sobre `Axiom.Spec/specs/00..08` y
`Axiom.Spec/context/**`.
