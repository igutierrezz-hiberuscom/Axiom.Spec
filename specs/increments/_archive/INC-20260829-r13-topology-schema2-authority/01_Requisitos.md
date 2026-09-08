# 01 Requisitos

## Requisitos trazables

- **B-057.1**: loader, tipos, exports, defaults, productores, consumidores y docs activas no contienen schema topológico 1 ni `sddRepo/specRepo/roleCodeRepositories`.
- **B-057.2**: `legacyRepos` admite como máximo una fuente `sdd` y una `spec`, ambas explícitas y read-only.
- **B-058.1**: validator único comprueba kinds, mode, refs, IDs, colisiones físicas, roles, ownership, primary, assignments y shape cerrada.
- **B-058.2**: toda escritura o API sensible invoca ese validator y preserva findings tipados.
- **B-059.1**: solo `<axiomRepo>/axiom.config/topology.yaml` es autoral.
- **B-059.2**: `roles register|assign|unassign` localiza y muta esa autoridad desde cualquier repo resoluble.
- **B-059.3**: `axiom.yaml` de code repo contiene únicamente identidad/puntero gestionado en lo relativo a topología.
- **B-059.4**: el dogfood schema 2 mantiene Axiom.Spec/Axiom/Axiom.SDD físicamente separados y no escribe Axiom.SDD.

## Reglas

Los legacy repos nunca reciben escrituras estructurales. El validator no infiere funciones legacy por ID. No existen fuentes topológicas equivalentes ni fallback a una copia reanclada.

## Fuera de alcance

Bindings locales, locks/writer, registro user-level, transacción de setup y seguridad launcher.
