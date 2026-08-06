# 01 Requisitos

## Objetivo del documento

Fijar el contrato efectivo del resolver de proyectos después del retiro de los
modos de resolución obsoletos.

## Requisitos del incremento

- `ProjectResolution.mode` expone únicamente `local-only`.
- La lectura de `axiom.yaml` v1 y v2 acepta los literales históricos
	`gateway` y `hybrid` como entrada de compatibilidad.
- La normalización de cualquiera de esos literales devuelve `local-only` y no
	activa providers, permisos, discovery ni comandos remotos.
- Los consumers pueden conservar el campo `mode` sin ampliar su API pública.
- `TopologyManifest.mode`, los modos de worktree y el routing de herramientas
	siguen siendo contratos independientes.

## Reglas de negocio relevantes

- La política efectiva de runtime es `builder` + `local-only` + target.
- Los valores raw legacy se conservan únicamente en `rawConfig` o en
	documentación histórica explícita; no se propagan al modelo normalizado.
- Un modo raw desconocido no crea una nueva variante operativa.

## Fuera de alcance funcional

- No se reintroducen gateway, overlays remotos, providers enterprise ni flags
	de gateway.
- No se eliminan los modos de topología `single-repo`/`multi-repo` ni los
	layouts heredados que todavía tengan consumidores propios.
