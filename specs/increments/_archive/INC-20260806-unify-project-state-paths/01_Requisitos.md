# 01 Requisitos

## Objetivo del documento

Definir una única convención física para el estado runtime ligado a un
proyecto y una migración compatible desde las rutas históricas.

## Requisitos del incremento

- Todo writer project-bound nuevo escribe bajo
	`.axiom-state/<projectKey>/`.
- En v2, `projectKey` es `axiom.yaml#projectId`; en v1 es el slug estable de
	`project.name`.
- El scope público `config` puede conservarse, pero no crea un directorio
	físico `config`.
- `memory/`, `mcp/`, `outputs/` y `executions/<executionId>/` solo separan
	familias o ejecuciones dentro de sus fronteras definidas.
- `.axiom-state/local/` queda reservado para datos repo/operador-locales.
- Los lectores aceptan direct, `config`, scope y local legacy conocidos con
	precedencia determinista, migración atómica, idempotencia y warnings de
	conflicto.

## Reglas de negocio relevantes

- El namespace canónico gana siempre frente a copias legacy.
- Una copia legacy seleccionada se mueve al destino canónico y no se vuelve
	a escribir en la ruta antigua.
- Dos proyectos, aunque compartan un nombre humano parcial o una raíz de
	trabajo, no pueden compartir el namespace de estado.
- El estado de ejecución no se mezcla con el estado project-bound.

## Fuera de alcance funcional

- No se cambia `axiom.config/`, `axiom.spec/` ni `~/.axiom/`.
- No se añade una base de datos, índice global ni integración empresarial.
- No se reescribe historia archivada; solo se migra estado runtime presente.
