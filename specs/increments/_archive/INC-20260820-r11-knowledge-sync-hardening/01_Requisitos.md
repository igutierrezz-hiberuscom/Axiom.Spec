# 01 Requisitos

## Objetivo del documento

Definir un intercambio de memoria compartida seguro, explícito e idempotente.

## Requisitos del incremento

1. `knowledge pull` no declara ni acepta `--increment`; previsualiza todos los chunks pendientes sin mutar y los importa todos solo con `--confirm`.
2. `knowledge sync --increment --phase` conserva su filtro, previsualiza por defecto y requiere `--confirm` para escribir chunks/manifest y crear commit; `--push` solo puede enviar remoto con confirmación explícita.
3. El proceso no usa comandos Git construidos por interpolación de shell.
4. Sync serializa evidencia completa de `MemoryEntry`, incluidos `rationale`, `source` y metadata estable, y exporta únicamente `visibility: project-shared`.
5. Entradas privadas, sin visibilidad o con secretos en cualquier valor textual serializado se omiten y se cuentan por motivo.
6. Manifest, chunks y entradas se validan antes de persistir/importar; toda escritura local usa patrón atómico.
7. El marker de import se guarda project-scoped con helpers canónicos, migra lectura legacy de `.engram/.imported` y nunca se incluye en contenido Git compartido.
8. Un chunk corrupto, inválido o parcialmente fallido queda pendiente con diagnóstico; solo se marca después de que todas sus entradas válidas se guardan correctamente.

## Reglas de negocio relevantes

Los chunks y manifest bajo `.engram/` son append-only y versionables. La memoria personal y el estado de importación no lo son. La operación es idempotente incluso tras reintentos de un fallo parcial.

## Fuera de alcance funcional

No se modifica el contrato general de memoria ni se implementa promoción automática a contexto/skills.
