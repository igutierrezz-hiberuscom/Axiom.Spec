# TECHNICAL_CONTEXT

Puerta de entrada obligatoria al conocimiento técnico estable del producto Axiom, tal como existe en el código real de `Axiom/` a fecha 2026-07-02. Cada documento enlazado aquí cita su fuente (código, README de package, o `Axiom/docs/`) y distingue explícitamente entre estado verificado y planificación futura.

## Propósito del contexto técnico

Este contexto explica CÓMO está construido el producto Axiom hoy: su arquitectura por capas, su modelo de datos, su ciclo de vida operativo, sus integraciones y sus riesgos conocidos. No es un resumen de todo el código ni documentación ficticia de módulos no implementados — cada afirmación aquí es trazable a un fichero concreto del repo `Axiom/` o de sus docs oficiales.

Para QUÉ debe hacer el producto (requisitos, alcance, principios), ver `specs/00_Resumen_Ejecutivo.md` a `specs/08_Glosario.md`. Este contexto técnico es el complemento de "cómo está hecho" a esa spec de "qué debe hacer".

## Estado real del producto (resumen)

- Monorepo npm workspaces en `Axiom/`: `apps/cli` + 28 packages bajo `packages/*` (6 de ellos sub-packages de `packages/adapters/`).
- Runtime MVP cerrado 2026-06-25; oleadas post-MVP (0019-0039) cerradas hasta 2026-07-01/02.
- CLI real con 36 ficheros de comando (~16.400 líneas), de los cuales solo ~12 tienen página de documentación operativa dedicada en `Axiom/docs/cli/`.
- Historial git real de 40 commits (2026-06-05 → 2026-07-02).
- Existe una discrepancia real no resuelta entre lo que el propio repo `Axiom/` espera de sí mismo (`axiom.spec/`, `AGENTS.md`, `axiom.skills.lock` en su raíz) y lo que existe físicamente hoy (nada de eso está presente). Ver `references/03-riesgos-y-brechas-conocidas.md`.

## Estructura de `context/`

- `architecture/` — cómo está construido el runtime: capas, modelo de datos/configuración, ciclo de vida CLI/orquestación, adapters y model routing.
- `operations/` — cómo se instala, onboarda, diagnostica y audita un proyecto real.
- `integrations/` — capabilities, providers, toolchain externo (Serena, CodeGraph, etc.) y bridges opcionales.
- `references/` — material de consulta puntual: inventario de los 28 packages, historial de incrementos 0015-0030, y riesgos/brechas conocidas verificadas.

### Índice de documentos

1. [architecture/01-vision-general-y-capas.md](architecture/01-vision-general-y-capas.md)
2. [architecture/02-modelo-de-datos-y-configuracion.md](architecture/02-modelo-de-datos-y-configuracion.md)
3. [architecture/03-ciclo-de-vida-cli-y-orquestacion.md](architecture/03-ciclo-de-vida-cli-y-orquestacion.md)
4. [architecture/04-adapters-y-model-routing.md](architecture/04-adapters-y-model-routing.md)
5. [operations/01-instalacion-y-onboarding.md](operations/01-instalacion-y-onboarding.md)
6. [operations/02-doctor-troubleshooting-y-telemetria.md](operations/02-doctor-troubleshooting-y-telemetria.md)
7. [integrations/01-capabilities-providers-y-toolchain.md](integrations/01-capabilities-providers-y-toolchain.md)
8. [references/01-inventario-de-packages.md](references/01-inventario-de-packages.md)
9. [references/02-historial-de-incrementos.md](references/02-historial-de-incrementos.md)
10. [references/03-riesgos-y-brechas-conocidas.md](references/03-riesgos-y-brechas-conocidas.md)

## Decisiones arquitectónicas vigentes

- `Result<T, E>` sin excepciones en todo el dominio (`@axiom/core`).
- Escritura atómica (tmp+rename) e idempotencia byte a byte en toda superficie generada.
- Aislamiento project-scoped obligatorio (`@axiom/isolation`): ninguna ruta/cache/binding cruza `projectId`.
- YAML declarativo como fuente de configuración (profiles, capabilities, providers, telemetry, policy), separado por responsabilidad.
- GATEs numeradas y verificables por `@axiom/doctor`, no solo documentadas.
- Ver detalle y enlaces a incrementos concretos en `references/02-historial-de-incrementos.md`.

## Convenciones y reglas operativas

- No promover como estado actual nada que solo esté en el roadmap de rediseño (`specs/increments/INC-20260702-axiom-redesign-roadmap/`) — ese roadmap es planificación pura, sin código.
- No confundir `Axiom.Spec/` (este repo) con `axiom.spec/` (carpeta interna que el producto espera dentro de un proyecto adoptante). Ver glosario en `specs/08_Glosario.md`.
- Cada documento de este contexto debe citar su fuente (path de código, README de package o doc oficial) para cada afirmación no trivial.
- Al añadir un documento nuevo bajo `context/`, actualizar el índice de esta página.

## Riesgos, excepciones y límites conocidos

Ver detalle completo en [references/03-riesgos-y-brechas-conocidas.md](references/03-riesgos-y-brechas-conocidas.md). Resumen:

1. `axiom.spec/`, `AGENTS.md` y `axiom.skills.lock` en la raíz de `Axiom/` están referenciados por scripts y docs propios pero no existen en este checkout.
2. 24 de los 36 comandos CLI reales no tienen documentación operativa dedicada.
3. Los packages sin `README.md` (mayoría) fueron caracterizados a partir de su estructura de `src/`, no de documentación propia — tratar como inferencia razonable, no como contrato firmado.
4. El roadmap de rediseño de topología de repos (INC-01 a INC-24) no tiene fecha de inicio ni stack de implementación decidido.

## Fuentes auditadas y última validación

- Fuentes: `Axiom/README.md`, `Axiom/package.json`, `Axiom/docs/**/*.md` (40+ ficheros), `Axiom/packages/**/README.md`, estructura de `Axiom/packages/**/src`, `Axiom/apps/cli/src/commands/*.ts` (listado), `Axiom/scripts/verify-first-project-readiness.mjs`, `Axiom.SDD/AGENTS.md`, `git log` de `Axiom/`.
- Última validación: 2026-07-02.
- Responsable humano: pendiente de asignar por el equipo (no declarado en el momento de esta redacción).

## Regla crítica

Este contexto técnico no debe convertirse en un resumen de código ni en documentación ficticia de módulos no implementados. Toda afirmación sobre "estado actual" debe ser verificable releyendo el código o los docs citados; toda afirmación sobre planes futuros debe marcarse explícitamente como tal.
