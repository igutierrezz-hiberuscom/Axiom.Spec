# 01 Requisitos

## Objetivo del documento

Fijar un único borde de I/O para transiciones SDD y hacer observables sus decisiones y recuperaciones.

## Requisitos del incremento

1. El runner usa exclusivamente el resolvedor fail-closed de ACC-045.
2. Centraliza legalidad, preview, confirmación, efectos locales, QA, persistencia y receipts.
3. CLI, launcher y MCP no reimplementan transiciones ni persisten una ruta parcial.
4. Archive/integrate coordina estado, metadata, integración, movimiento y compensación ante fallo.
5. `requiresApproval` exige confirmación explícita en todos los canales y no se bypass con `--force`/`--no-verify`.
6. El grafo activo no declara el tracker sin dispatcher `close-external-work-item`.

## Reglas de negocio relevantes

Una previsualización nunca escribe. Confirmación explícita es independiente de verify/force. La máquina de estados sigue siendo una función pura.

## Fuera de alcance funcional

No se cambia tracker general, no se añade I/O a la máquina, no se agregan estados y no se implementa integración externa automática.
