# 03. Actualizar versiones

Cómo actualizar el runtime de Axiom instalado y cómo revertir si algo sale mal.

## `axiom upgrade`

Aplica el plan de migraciones del runtime de versionado (`@axiom/versioning`) sobre el proyecto activo, con contrato **rollback-first**:

1. Lee (o inicializa) el `ManagedState` actual.
2. Calcula las migraciones aplicables entre versión actual y target.
3. Crea un **checkpoint** pre-upgrade (snapshot de `init.json`, `install-profile.json`, `managed-state.json` y cualquier `touchedFiles` adicional que declare la migración) — se conservan los últimos 5.
4. Aplica cada migración en orden. Si alguna falla, **restaura el checkpoint automáticamente** y re-lanza el error.
5. Persiste el nuevo `ManagedState`.
6. Corre `axiom sync` y `axiom doctor` post-upgrade (salvo `--no-sync`/`--no-doctor`), restaurando el checkpoint si cualquiera de los dos falla.

```bash
axiom upgrade --dry-run                         # muestra el plan sin mutar nada
axiom upgrade                                   # aplica el upgrade real
axiom upgrade --from-checkpoint <id>             # usa un checkpoint existente como restore-point
axiom upgrade --target-version <v>               # apunta a una versión concreta
axiom upgrade --no-sync --no-doctor             # salta los pasos post-upgrade
```

**Precondición crítica**: `axiom upgrade` exige que el proyecto tenga un `managed-state.json` materializado (es decir, que `axiom configure` o `axiom sync` hayan corrido al menos una vez). Si falta, aborta con `managedStateMissing` antes de mutar nada.

### Upgrade multi-repo (fan-out cross-repo)

Ejecutado desde el **repo de control** de un proyecto multi-repo, `axiom upgrade` itera automáticamente TODOS los repos de la topología (`sddRepo`, `specRepo`, cada repo de rol), migrando/checkpointeando cada uno por separado, con `sync`+`doctor` corriendo una sola vez a nivel workspace y un reporte por-repo (`repoId`/`role`/`path`/versión origen→destino/checkpoint/ok\|failed). `--repo-only` fuerza el modo per-repo clásico (o basta con correrlo desde un repo que no sea el de control). Un proyecto single-repo se comporta exactamente igual que antes (el fan-out solo se dispara en multi-repo).

## Rollback manual (`axiom rollback`)

Además del restore AUTOMÁTICO ante fallo de upgrade, hay un comando de operador para restaurar deliberadamente un checkpoint conocido:

```bash
axiom rollback --list              # enumera los checkpoints disponibles
axiom rollback <checkpointId> --dry-run   # previsualiza qué archivos se restaurarían
axiom rollback <checkpointId>              # restaura el ManagedState + init.json + install-profile.json desde ese checkpoint
```

Un `checkpointId` inexistente da un error claro SIN mutar nada (el chequeo de existencia ocurre antes de cualquier escritura). Es una acción de recuperación deliberadamente NO gateada por el orchestrator (bloquearla podría impedir recuperarse de un fallo).

## `axiom self-update`

Actualiza el propio binario/instalación de la CLI de Axiom (distinto del `ManagedState` del proyecto, que gestiona `axiom upgrade`). Útil cuando hay una versión nueva del paquete `@axiom/cli` disponible. También accesible desde el menú de bootstrap `setup` de la TUI ("Actualizar Axiom").

## Relacionado

- [02_Configuracion.md](02_Configuracion.md)
- [01_Que_Es_Cada_Cosa.md](01_Que_Es_Cada_Cosa.md)
- [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md)
