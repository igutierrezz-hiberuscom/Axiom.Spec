# 02 Cambios de Modelo

## Objetivo del documento

Describir el cambio de contrato entre la entrada raw del manifiesto y la
resolución pública consumida por el runtime.

## Entidades o estructuras afectadas

- `ProjectMode` en `@axiom/project-resolution` queda cerrado a `'local-only'`.
- `ProjectResolution.mode` conserva su posición y nombre para compatibilidad.
- `normalizeMode` sigue leyendo `project.mode` v1 y `mode` v2, pero devuelve
	siempre la política efectiva local-only.
- Las pruebas de resolver cubren v1/v2 con `gateway`, `hybrid` y ausencia de
	modo.

## Contratos o estados afectados

- `status: resolved` permanece válido para manifiestos que contienen un modo
	raw histórico permitido por el schema de entrada.
- `IC-003` de doctor valida la política efectiva local-only; no existe una
	rama activa para modo remoto.
- La declaración compilada del paquete no contiene `gateway` ni `hybrid` en
	la union pública `ProjectMode`.

## Notas de compatibilidad

La compatibilidad es de lectura, no de comportamiento: un manifiesto antiguo
se puede abrir y normalizar, pero la resolución resultante no conserva un
selector remoto. Las menciones de `gateway`/`hybrid` fuera de esa frontera
deben estar marcadas como legacy o históricas.
