# Doctor, troubleshooting y telemetría

Fuente: `Axiom/docs/cli/doctor.md`, `Axiom/docs/troubleshooting.md`, `Axiom/docs/configuration/telemetry-and-isolation.md`, `Axiom/docs/configuration/files/telemetry-sinks.md`.

## `axiom doctor`

Familias de checks: boundaries (separación entre scopes documentales y runtime), policies (`integrations.yaml`, `policy-as-code.yaml` presentes), manifests (`axiom.yaml` válido, `.sdd/local/` protegida), isolation (invariantes project-scoped, restricciones MCP), capability model (consistencia del modelo declarativo), gateway (`gateway-state.json` vs drift contra `providers.yaml`/`profiles.yaml`/`install-profile.json`). Salida `--json` disponible. No muta nada.

## Troubleshooting por comando (categorías documentadas)

**`axiom init`**
- `Cannot find module '@axiom/orchestrator'`: binario compilado en `apps/cli/dist/` no resuelve packages workspace → revisar `workspaces` en `package.json`, `npm install`, rebuild.
- `invalid-config`: YAML de `axiom.yaml` malformado → corregir sintaxis/indentación.
- Conflicto con `axiom.yaml` existente sin `--force`.
- Nombre de proyecto inválido (no cumple regex).

**`axiom join`**: falta `--member` en modo no interactivo.

**`axiom configure`**
- No encuentra `init.json` → ejecutar `axiom init` primero.
- Falla al escribir surfaces de Copilot → revisar `axiom.spec/templates/copilot-instructions.template.md`, `product.manifest.yaml`, `axiom.yaml`, `providers.yaml`, overlay local.

**`axiom sync`**
- `telemetrySinkMissing`: overlay exige señales mínimas sin sink habilitado → revisar `telemetry-sinks.yaml`, overlay activo, re-ejecutar `configure`.
- `adapterGenerationFailed`: falta `install-profile.json` (se corrió `sync` antes de `configure`).

**`axiom start`**
- Conflicto overlay/flags de gateway (`enterprise` prohíbe `--no-gateway`; `standard` lo trata como opt-in; `local-only` siempre filesystem).
- Warning por capability model no cargado (`capabilities.yaml`/`providers.yaml` faltantes o incompletos).

**`axiom audit`**
- `violation`: rewrite externo detectado del audit trail → tratar como señal de integridad rota.
- Warnings de retención: revisar política espejo en `telemetry-sinks.yaml`.

**`axiom doctor`**: correr con `--json`, corregir primero fallos estructurales, dejar warnings para una segunda pasada.

## Telemetría (`telemetry-sinks.yaml`)

Impacta: `sync` (valida gate antes de regenerar outputs), `audit` (usa mirror de retención), boot del CLI (carga sinks habilitados).

- `dataSensitivityBoundaries` por overlay: tags permitidos, tags redactados, nivel de redacción, flujo cross-project permitido, ventana de retención.
- `sinks`: `null-sink`, `log-sink`, `remote-sink`, `audit-trail-sink`.
- Regla práctica: si un overlay exige señales mínimas (`minimumSignals`) y no hay sink habilitado que las cubra, `sync` aborta **antes** de escribir.
- No documentado explícitamente: un mecanismo formal de opt-out completo de telemetría. Tratar como ausente, no como implementado silenciosamente.

## Aislamiento (`@axiom/isolation`, doctor)

`doctor` valida: que las rutas project-scoped contengan el `projectId`; que `backend-mcp`/`frontend-mcp` sigan bloqueados en MVP; que el modo local pueda operar sin gateway cuando corresponde.

`integrations.yaml` declara: si el sistema está scoped por proyecto, cómo se resuelve el proyecto, catálogo y modo de política, defaults gestionados por Axiom, decisiones de instalación aprobadas, y el bloque `projectIsolation` (qué contaminación cross-project debe bloquearse).

## Gobernanza mínima verificada por doctor

`integrations.yaml` existe; `policy-as-code.yaml` existe; `axiom.yaml` es válido; `.sdd/local/` no queda expuesta accidentalmente al versionado.
