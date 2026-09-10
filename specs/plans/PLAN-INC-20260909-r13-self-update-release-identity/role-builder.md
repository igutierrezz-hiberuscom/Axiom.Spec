# Rol de plan: builder

## Rol

- **Nombre:** builder
- **Repositorio de implementación:** `Axiom` (runtime TypeScript: `packages/*`, `apps/cli`)
- **Repositorio de especificación:** `Axiom.Spec`, solo para consulta e integración estable posterior mediante flujo autorizado
- **Artefacto rector:** `INC-20260909-r13-self-update-release-identity`
- **Posición:** 1 de 5; no inicia ni implementa incrementos 2–5

## Objetivo del rol en este plan

Entregar la identidad y consulta que habilitan self-update sin implementar la actualización. El builder convierte la spec en contratos de dominio, adapters Git/cache, metadata del CLI global, evaluación tipada y salida CLI. La consulta no muta Git, instalación o proyecto; solo puede escribir cache user-level tras una observación live válida.

El builder no presume que las rutas candidatas existan. Su primer resultado es un mapa confirmado del runtime y baseline reproducible; solo después empieza código.

## Alcance incluido

1. Revalidar estructura, package boundaries, scripts, tests y convenciones.
2. Localizar fuentes de versión/producto e imports de `@axiom/user-workspace`.
3. Localizar `ManagedState.runtime.version` y mantenerlo project-scoped.
4. Demostrar remoto canónico sin confiar en `origin`.
5. Implementar tipos, provenance enlazada, freshness y reglas SemVer/aliases.
6. Implementar/generar identidad de artefacto y CLI global efectivo.
7. Implementar descubrimiento Git best-effort sin mutación ni optional writes.
8. Implementar cache user-level atómica, saneada e históricamente explícita.
9. Implementar evaluator puro, evidence completa y orden determinista.
10. Registrar status y presenters text/JSON sobre el mismo assessment.
11. Crear tests herméticos con Git local y entorno de usuario aislado.
12. Ejecutar validación, exclusiones y handoff semántico de 48 AC.

## Secuencia operativa por archivos y símbolos

Las rutas definitivas se rellenan tras G0. No adelantar capas consumidoras:

| Orden | Ruta probable a confirmar | Símbolos/acción | Evidencia inmediata |
|---:|---|---|---|
| 0 | `package.json`, README, configs, `packages/*`, `apps/cli` | mapa entrypoint/build/state/Git/cache/commands/tests | baseline build + Vitest |
| 1 | core release/self-update o `packages/core/src/**` | identities, `PublishedObservation`, provenance, diagnostics, states, assessment | typecheck + exhaustive tests |
| 2 | utility SemVer | parser/selector, aliases compatibles/incompatibles | tabla SemVer 2.0.0 |
| 3 | build/CLI package | `BUILD_IDENTITY`, `getInstalledCliIdentity` | artifact smoke y dos cwd |
| 4 | consumidores user-workspace | retirar lookup/literal como autoridad | búsqueda estática |
| 5 | reader/modelo ManagedState | lectura opcional de `runtime.version`; cero escritura | dos proyectos/un CLI |
| 6 | port + adapter Git | target OID/commit proof, local HEAD, `ls-remote`, canonicidad | repos locales + argv/env allowlist |
| 7 | storage/path user-level | cache validate/atomic write/clock/age/stale | live/offline/corrupt/secreto |
| 8 | application | assessment timestamp, evaluator, evidence/orden | matriz estados/provenance |
| 9 | command registry/presenters | status text/JSON | snapshots + exit codes |
| 10 | tests integración | fixtures Git/HOME/no-mutation | focalizadas + suite |

Si la arquitectura usa otros nombres, conservar responsabilidades y actualizar mapa; no crear carpetas duplicadas para coincidir con la tabla.

## Checklist de revalidación (G0)

- [ ] Leer reglas y scripts del runtime.
- [ ] Aislar worktree/diff del incremento.
- [ ] Resolver entrypoint global, registry y exit codes.
- [ ] Inventariar literales/lookups e imports user-workspace.
- [ ] Resolver ManagedState y reader.
- [ ] Resolver Git runner, timeout, no-prompt y optional-lock control.
- [ ] Resolver paths/filesystem user-level.
- [ ] Resolver build/packaging e inyección de metadata.
- [ ] Resolver fuente inequívoca del remoto canónico.
- [ ] Capturar baseline build/tests y fallos previos.

**STOP inmediato:** autoridad canónica ausente, build dependiente del proyecto, optional writes inevitables, diff no aislable o necesidad de apply/manifest v2/Launcher/CI.

## Reglas de implementación

- Mantener puros tipos/evaluator; Git, filesystem, clock y entorno entran por puertos.
- Usar argv estructurado, no shell strings; deshabilitar prompting, mantenimiento y optional locks/refrescos (`GIT_OPTIONAL_LOCKS=0` o equivalente), con timeout.
- Prohibir fetch/pull/merge/checkout/switch/reset/clean/update-ref y cualquier escritura.
- Tratar OID remoto como `GitObjectId`; solo llamar commit al tipo demostrado con objetos existentes. No traer objetos para verificarlo.
- Nunca comparar versiones como texto, coaccionar parciales, usar nombre de branch, desigualdad de SHA u `origin` como prueba.
- Aplicar política de aliases: targets distintos son ambiguous; mismo target elige representación determinista y conserva todos.
- Representar tags locales incompatibles con `ambiguous-tag`, no elegir por enumeración.
- Nunca copiar una identidad ausente desde otra fuente.
- Cada identidad y estado enlaza provenance IDs del mismo assessment; no aceptar referencias huérfanas.
- `observedAt` del assessment es distinto del timestamp remoto/cache.
- Cachear solo live válido mediante temp + replace atómico. Toda cache leída es `cached`, `stale=true`; no hay TTL en este incremento.
- Sanear antes de persistir/presentar; secreto canary no aparece.
- Derivar text/JSON del mismo assessment.
- Estados de dominio son éxito de proceso si el contrato se emitió.
- No añadir dependencia sin comprobar existentes y aprobación; fijar versión exacta si se autoriza.
- No editar metadata/status/índices ni ejecutar transiciones.

## Tests herméticos obligatorios

### Helper Git local

Crear helper dentro del paquete propietario que:

1. inicialice bare remote temporal;
2. inicialice author repo con identidad Git local;
3. cree/pushee commits, tags ligeros/anotados, aliases, tags ambiguos y tag a objeto no-commit;
4. cree checkout sujeto branch/detached/dirty;
5. permita objetos remotos ausentes localmente;
6. capture refs, HEAD, índice, status y objetos antes/después;
7. destruya el temp.

No usar red, GitHub, repos del workspace o credenciales reales.

### Entorno aislado

- HOME, XDG_CONFIG_HOME/XDG_CACHE_HOME, APPDATA y LOCALAPPDATA en temp.
- Clock fijo, artifact identity fixture y canonical repository fixture.
- Runner que registra argv/env, exige optional locks deshabilitados y rechaza mutaciones.
- Offline local determinista; nunca desconectar máquina.

### Matriz mínima

- `v1.9.0`/`v1.10.0`, prerelease/final, invalid tags y build aliases same/different target.
- Annotated/lightweight target; commit demostrado, unverified y non-commit.
- Canonical remote distinto de `origin`, `origin` incorrecto, ausencia de match y salida inválida.
- Checkout tag/ref/commit/ambiguous-tag/unavailable y dirty.
- Siete estados, combinaciones, orden, dedupe y evidence/provenance links.
- CLI global estable entre ManagedState distintos.
- Cache live, offline con `stale=true`, offline sin cache/published unavailable, corrupt/mismatch/clock skew.
- Secret URL, múltiples CLI candidates y build parcial.
- Text/JSON, stdout/stderr, exit codes y assessment timestamp.
- No mutation de refs/índice/worktree/objects e inexistencia de apply/Launcher.

## Dependencias

- Runtime actual validado en G0.
- Acciones ACC-077/080.
- Git local, TypeScript/Vitest y scripts existentes.
- Maintainer para autoridad canónica y metadata de build.

No depende de incrementos 2–5. Si otro incremento cambia estos contratos, detener y secuenciar este primero.

## Validaciones y evidencias esperadas

Desde raíz `Axiom`, ejecutar cada test focalizado confirmado en G0 mediante `npx vitest run` con su ruta real, y después:

```text
npm run build
npx vitest run
git diff --check
```

Añadir smoke no publicador solo si ya existe. El handoff incluye:

- commit/base y mapa real de símbolos;
- baseline previo y archivos modificados;
- comandos/exit codes;
- matriz R13-AC-001…048 → test/evidencia;
- argv/env Git y snapshots no-mutation;
- integridad de provenance IDs;
- búsquedas user-workspace, comandos mutantes, apply, manifest v2, Launcher;
- fallos previos frente a regresiones;
- GO/STOP por gate, riesgos residuales e integración estable propuesta.

No copiar logs completos ni datos sensibles a la spec.

## Gates del builder

- **G0:** arquitectura, autoridades, seams y optional-lock control.
- **G1:** modelo/freshness/evidence/SemVer exhaustivos.
- **G2:** artefacto global independiente de proyecto/user-workspace.
- **G3:** Git local/remoto demostrado no mutante.
- **G4:** cache atómica, histórica y sin secretos.
- **G5:** matriz, provenance y orden sin pérdida.
- **G6:** CLI text/JSON equivalente y sin efectos fuera de cache.
- **G7:** build, focalizadas, suite, 48 criterios y review GO.

No continuar tras STOP sin resolución explícita. Un workaround contrario a la spec no es GO.

## Riesgos y rollback del rol

- Packaging roto: revertir wiring de build y no publicar.
- Ancestry requiere objeto ausente: conservar unknown; nunca fetch.
- Git no puede evitar optional write: STOP G3.
- Cache no atómica: no habilitar escritura.
- Convención CLI distinta: adaptar registro, no duplicar superficie.
- ManagedState exige migración: reader opcional y elevar manifest v2 al incremento 3.

Rollback inverso: command → orchestration → cache/Git → consumers/build identity → core types. No hay rollback de proyecto porque status no lo escribe. Cache namespaced puede quedar ignorada.

## Bloqueos y observaciones

- **Crítico:** remoto canónico indemostrable.
- **Crítico:** installed identity no obtenible del artefacto.
- **Crítico:** operación/optional write Git.
- **Crítico:** tests acceden a red, HOME, cache o CLI reales.
- **Alcance:** necesidad de apply, manifest v2 completo, Launcher o CI final.
- `offline` no significa “sin actualización”; sin cache, published es unavailable.
- `updated` puede coexistir con checkout behind/misaligned.
- OID unverified no es commit.
- No cerrar/cambiar status manualmente; handoff deja pending hasta review, integración y Core posterior.
