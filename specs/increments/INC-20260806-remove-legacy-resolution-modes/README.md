# Retire legacy gateway and hybrid resolution modes

> **Código**: INC-20260806-remove-legacy-resolution-modes
> **Estado**: Cerrado pendiente de archivado
> **Fecha de creación**: 2026-08-06
> **Tipo de cambio**: eliminar contrato obsoleto del resolver

## Resumen

`ProjectResolution.mode` debe describir solo la politica efectiva vigente:
`local-only`. Los valores legacy `gateway` y `hybrid` no deben formar parte
del tipo publico ni aparecer como modos operativos activos.

## Contexto y motivación

R-04 retiro los overlays, el provider `axiom-gateway`, los comandos gateway y
las flags de gateway. Sin embargo, `@axiom/project-resolution` aun exporta
`ProjectMode = 'local-only' | 'gateway' | 'hybrid'` y `normalizeMode` conserva
los dos valores retirados. El doctor solo necesita confirmar que el proyecto
queda en local-only, por lo que mantener esos literales en el resolver crea
un contrato activo que contradice el runtime y la documentacion.

La compatibilidad de lectura se conserva: un `axiom.yaml` viejo puede traer
un modo legacy, pero la resolucion externa siempre devuelve `local-only`.

## Alcance

### Incluido

- Reducir `ProjectMode` a `'local-only'` y hacer que toda entrada de modo
	devuelva ese valor efectivo.
- Actualizar `project-resolution` y sus exports publicos.
- Añadir pruebas v1 y v2 para entradas legacy `gateway` y `hybrid`, junto con
	el caso por defecto, verificando que todos resuelven a `local-only`.
- Actualizar consumidores, comentarios, README y claims activos que traten
	`ProjectResolution.mode` como gateway/hybrid.
- Regenerar `dist` mediante el build; no editar salidas generadas a mano.

### Excluido

- Eliminar `TopologyManifest.mode` (`single-repo`/`multi-repo`) o los layouts
	`installed-multi-repo` y `one-axiom-per-product` de `axiom.yaml`.
- Eliminar los modos internos de `tool-routing`, worktree o Git.
- Reintroducir un gateway, broker remoto o proveedor enterprise.
- Cambiar la unificacion fisica de rutas de estado; pertenece a
	`INC-20260806-unify-project-state-paths`.
- Reescribir historia archivada.

## Documentos del incremento

- `Axiom/packages/project-resolution/src/resolver.ts`
- `Axiom/packages/project-resolution/src/index.ts`
- `Axiom/packages/project-resolution/tests/resolver.test.ts`
- `Axiom/packages/doctor/src/checks.ts`
- `Axiom/apps/cli/tests/schemaversion2-e2e.test.ts`
- `Axiom.Spec/specs/03_Modelo_Operativo_y_Datos.md` como destino de la
	integracion final, no durante el apply.

## Dudas abiertas

Ninguna bloqueante. Se decide conservar el campo `mode` en
`ProjectResolution` para no romper consumidores; su union queda cerrada a
`local-only`.

## Decisiones funcionales cerradas

- Un modo raw legacy se tolera al leer, pero no se propaga al contrato
	normalizado ni se usa para seleccionar providers, permisos o discovery.
- `mode` de resolucion y `mode` de topologia son ejes distintos y no se
	mezclan.
- Los valores legacy solo pueden aparecer en normalizadores de compatibilidad
	o en historia explicita.

## Consolidación en la spec general

La integracion final debe actualizar el modelo de datos y el glosario para
decir que la politica efectiva de resolucion es local-only y que gateway/hybrid
son valores legacy de entrada, no modos actuales.

## Estrategia E2E

- Resolver un `axiom.yaml` v1 con `project.mode: gateway` y otro con
	`project.mode: hybrid`; ambos deben devolver `status: resolved` y
	`mode: local-only`.
- Resolver los equivalentes v2 con `mode: gateway` y `mode: hybrid`.
- Confirmar que `axiom doctor` no emite el warning de modo remoto para esas
	entradas legacy y que no reaparece ningun comando gateway.

## Trazabilidad y fuentes

- Accion de auditoria: `ACC-022`.
- Decision previa: `ACC-018` fijo local-only y retiro gateway.
- Codigo observado: `ProjectMode`, `normalizeMode`, check `IC-003` y
	`AxiomYamlSchema` que acepta el campo raw sin convertirlo en enum de runtime.

## Estado de validación humana

Implementación ejecutada y revisada independientemente. Los hallazgos de
review quedaron resueltos; el freeze y los receipts finales se regeneran sobre
esta versión documental antes del archivado.

## Notas de implementación

- `ProjectMode` queda cerrado a `'local-only'` en
	`Axiom/packages/project-resolution/src/resolver.ts`.
- `normalizeMode` sigue leyendo el `mode` raw de v1 (`project.mode`) y v2
	(`mode` top-level), pero siempre devuelve `local-only`; `rawConfig` conserva
	el documento leído para compatibilidad estructural.
- Las pruebas focalizadas cubren `gateway`, `hybrid` y el default sin modo en
	v1 y v2. El consumidor directo `IC-003` de doctor ya no contiene una rama
	para modos remotos y afirma la política efectiva local-only.
- El build regeneró `dist`; no se editaron salidas generadas manualmente. No
	se tocaron `TopologyManifest.mode`, los modos de layout, worktree o
	tool-routing, ni los canónicos 00..08/context.

## Acceptance criteria

- [x] `ProjectMode` solo contiene `local-only`.
- [x] v1 y v2 normalizan `gateway` y `hybrid` a `local-only`.
- [x] No quedan consumidores activos que ramifiquen sobre esos modos del
			resolver.
- [x] Los modos de topologia, worktree y routing permanecen intactos.
- [x] Las pruebas focalizadas, build y doctor del alcance pasan.
- [x] Review independiente, freeze final, receipts e integración canónica
			quedan conservados junto al incremento.

## Risks and mitigations

- Un consumidor podria confundir `ProjectResolution.mode` con el modo de
	topologia. Se cubre con busqueda de usos y pruebas de ambos contratos.
- Las salidas `dist` pueden conservar texto viejo hasta recompilar. Se
	regeneran y se valida el runtime compilado.

## Validation

- `npx vitest run packages/project-resolution/tests/resolver.test.ts packages/doctor/tests/checks.test.ts`: OK.
- `npm run build`: OK (`tsc -b`).
- `npx vitest run`: suite completa verde después de actualizar las siete
			 expectativas de workflow al namespace canónico.
- `npm run doctor`: PASS, 45/60 OK, 0 fallos, 4 advertencias y 11 omitidos.
- `npm run readiness:first-project`: PASS.
- Después del build, `packages/project-resolution/dist/resolver.d.ts` expone
	`ProjectMode = 'local-only'` y no quedan `gateway`/`hybrid` en el `dist` del
	paquete.
- El candidate freeze histórico quedó obsoleto al actualizar esta README; el
			freeze final se emite después de cerrar la documentación y se valida por
			hash en el ledger y los receipts.

## Result

Implementación completada y validada en el runtime. Se cumplen los criterios
funcionales del resolver, la compatibilidad de lectura v1/v2 y el aislamiento
respecto de los modos de topología, layout, worktree y tool-routing. No se
detectaron fallos nuevos; la única expectativa fallida durante el apply fue la
expectativa legacy que aún esperaba `gateway`, y quedó corregida por las
pruebas nuevas. Los hallazgos independientes quedaron resueltos: no hay
ramificaciones activas por gateway/hybrid y los valores solo se toleran como
entrada legacy. El incremento queda `closed` y listo para archivado después de
emitir el freeze/receipts finales.

## General spec integration

Integrado en la pasada única del lote en `Axiom.Spec/specs/00..08` y
`Axiom.Spec/context/**`; las afirmaciones activas describen local-only y dejan
gateway/hybrid solo como compatibilidad de entrada o historia.
