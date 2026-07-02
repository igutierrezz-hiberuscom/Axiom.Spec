# 04 Flujos SDD y Ciclo de Vida

## Flujo base de este workspace (dogfooding, vigente)

1. especificar en `Axiom.Spec/` (specs numeradas, incrementos, bugs);
2. operar el workflow desde `Axiom.SDD/` (`AGENTS.md`: entender → localizar/crear spec → refinar con goal/scope/non-goals/acceptance criteria → implementar → validar → revisar → cerrar → integrar conocimiento estable en la spec);
3. implementar en `Axiom/`;
4. validar resultado (orden de descubrimiento: README → package scripts → task runners → test configs → build configs);
5. cerrar y reintegrar conocimiento en la spec. Un incremento/bug solo puede marcarse `closed` si: el objetivo es claro, hay acceptance criteria, se implementó (o hay justificación explícita de no-code), se corrió la validación disponible, se revisó contra el intent original y se integró conocimiento estable. Si falta algo, queda `status: pending` con motivo explícito.

## Ciclo de vida real del producto (lo que el runtime ejecuta para un proyecto adoptante)

```
init → join → configure → sync → start → audit → doctor
                                              ↓
                                          upgrade (cuando cambia la versión target)
```

Verificado por el script de smoke `Axiom/scripts/verify-first-project-readiness.mjs` (`npm run readiness:first-project`), que ejecuta exactamente esta secuencia extendida contra un proyecto temporal: `init → configure → toolchain repair → sync → start → gateway start → gateway status → audit → doctor`, y falla si algún paso devuelve exit code distinto de 0, falta un artefacto esperado, o `gateway status` no reporta `state: active`.

Contrato por comando (lee/escribe) detallado en [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md) y en `context/architecture/`.

## Baseline operativa actual

1. un solo rol funcional cubierto en profundidad (`functionalProfile: builder`);
2. soporte multi-repo dentro de un proyecto vía `@axiom/topology` (`installed-multi-repo` layout), no todavía como separación de repos de Axiom-el-producto;
3. ejecución local (`local-only`) o mediada por gateway (`standard`/`enterprise`), y adapters hacia 6 targets con package dedicado + 3 targets `fallback-only` sin adapter propio.

## Fuera de la baseline inicial (documentado explícitamente como no-goal del MVP)

Según `Axiom/docs/first-project-readiness.md`: overlays `standard`/`enterprise` como camino inicial obligatorio, `visual-studio-2026` como baseline de primer arranque, providers post-MVP (`engram`, `codegraph`, `graphify`) como requisito de entrada, bridges externos/plugins/lanes paralelos avanzados, e instalación user-level del binario como paso obligatorio.