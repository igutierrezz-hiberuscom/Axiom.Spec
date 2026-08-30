# 01 Requisitos

## Objetivo

Especificar los resultados observables de ACC-050..ACC-056.

## Requisitos

- **A-050**: ninguna fuente, export, comando, test o doc activa del runtime referencia `registry.json`, `Registry`, `ProjectEntry`, `legacyRegistryPath`, `legacy-registry-not-migrated` o `migrateLegacyRegistryV1ToV2`; artefactos archivados quedan históricos.
- **A-051**: `use` solo cambia `lastUsedAt`, `list` usa recencia determinista y `--print-path` es shell-neutral y seguro para cualquier carácter válido del path.
- **A-052**: el loader valida schemaVersion exacta, claves/IDs, strings no vacíos, timestamps ISO, repos no vacíos, roles/keys/paths y unknown keys; todo fallo vuelve como `Result` con error tipado.
- **A-053**: no existen colisiones silenciosas de ID o path ni merge por ID sin identidad común; conflictos no mutan bytes.
- **A-054**: cada read-modify-write del catálogo ocurre bajo el primitive compartido; escrituras concurrentes no pierden updates y no comparten temporal fijo.
- **A-055**: CLI/API/MCP comparten los estados físicos y distinguen resolubilidad cuando la operación la requiere.
- **A-056**: `--json` produce exactamente un documento v1 en stdout para todo resultado y usa stderr solo para diagnóstico humano; código 0 solo en éxito.

## Reglas de negocio

`projects add` cataloga un path y no equivale a `join`. Las entradas no se eliminan por indisponibilidad. Un catálogo corrupto jamás se reemplaza por uno vacío. No existe selección global de proyecto; la resolución sigue basada en cwd o ID explícito.

## Fuera de alcance

Topología schema 2, writer topológico, transacción de setup, seguridad HTTP del launcher y R-13.5.
