# 01 Requisitos

- **D-063.1**: resolución desde axiomRepo y cada code repo usa puntero+topology aunque `projects.yml` no exista o esté stale.
- **D-063.2**: catálogo opcional es proyección estructural; divergencia se informa y no domina el grafo.
- **D-063.3**: setup/adopt/incremental/member install/launcher/MCP funcionan con `--no-register`.
- **D-064.1**: Commander/help/código/tests/docs no exponen `repo attach`.
- **D-064.2**: repo/role add solo agrega code repo con ownership; control/spec/axiom/legacy se rechazan sin mutación.
- **D-069.1**: todos los consumidores usan un loader/schema de workspace JSON y errores tipados.
- **D-069.2**: add adapter/provider es atómico, lockeado, conserva campos/timestamps y distingue created/unchanged.

## Reglas

La resolución local no consulta el catálogo como autoridad ni infiere kinds por nombres. Ausencia de workspace.json es distinta de corrupción.

## Fuera de alcance

Reconciliación granular ACC-068, preflight/transacción estructural y control plane launcher.
