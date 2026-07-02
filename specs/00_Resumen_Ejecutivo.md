# 00 Resumen Ejecutivo

## Visión

Axiom es una plataforma de Spec-Driven Development (SDD) con un runtime MVP+post-MVP ya operativo. No es un simple reempaquetado de una herramienta existente: es un CLI Node/TypeScript (`axiom`) que, para un proyecto concreto, coordina estructura, configuración, resolución de profiles, materialización de archivos para herramientas externas (IDEs/CLIs de IA), validaciones y checks de salud.

Modelo mental del producto (`Axiom/docs/overview.md`):
- la spec dice qué reglas existen y qué contrato debe cumplirse;
- la configuración (YAML por proyecto) define cómo se activa ese contrato en un proyecto concreto;
- el CLI materializa, valida y opera sobre ese contrato.

## Estado real (verificado en el código, 2026-07-02)

Axiom NO está en fase de diseño: tiene un monorepo npm workspaces (`Axiom/`) con `apps/cli` + 28 packages bajo `packages/*` (ver [context/references/01-inventario-de-packages.md](../context/references/01-inventario-de-packages.md)), 36 ficheros de comandos en `apps/cli/src/commands/` y un histórico de al menos 40 commits (2026-06-05 a 2026-07-02) e incrementos numerados (0008 a 0039+) que fueron cerrando MVP y post-MVP en oleadas.

Según `Axiom/README.md` (2026-06-25/30), todas las capas declaradas están "implementadas y testeadas": dominio, aplicación (CLI), adapters (6 reales), telemetría, orquestación y persistencia.

## Qué hace el producto hoy

Para un proyecto que adopta Axiom, el ciclo de vida real es:

```
axiom init → axiom join → axiom configure → axiom sync → axiom start → axiom audit → axiom doctor → axiom upgrade
```

Cada comando lee/escribe un conjunto concreto de artefactos (`axiom.yaml`, `.sdd/<project>/*.json`, `.sdd/local/*`) y, según el profile activo, genera surfaces para el IDE/CLI de destino (`.opencode/`, `.claude/`, `.github/copilot-instructions.md`, `.cursor/`, `.vscode/`, `litellm.config.json`). El detalle completo vive en [04_Flujos_SDD_y_Ciclo_de_Vida.md](04_Flujos_SDD_y_Ciclo_de_Vida.md) y en `context/architecture/`.

## Dos modelos distintos que conviven bajo el nombre "Axiom" (evitar confundirlos)

1. **El producto Axiom** (`Axiom/`): el CLI/runtime descrito arriba, que cualquier proyecto externo puede adoptar. Su unidad de configuración es `axiom.yaml` + `.sdd/<projectName>/` dentro del proyecto que lo instala, más un catálogo de ~20 YAML de política/capacidad que el propio Axiom espera encontrar en una carpeta `axiom.spec/` (minúsculas) sembrada por `init`/`configure`.
2. **El workspace de desarrollo de Axiom** (esta raíz: `Axiom/`, `Axiom.SDD/`, `Axiom.Spec/`): el modelo de tres repos desacoplados que `Axiom.SDD/AGENTS.md` establece para construir el propio producto con disciplina SDD ligera (spec en `Axiom.Spec/`, implementación en `Axiom.SDD/`, runtime opcional en `Axiom/`). Este es el "dogfooding" de Axiom sobre sí mismo, y es un concepto de nivel distinto al `axiom.spec/` interno de un proyecto cualquiera.

Ver el glosario ([08_Glosario.md](08_Glosario.md)) para no mezclar `Axiom.Spec/` (este repo) con `axiom.spec/` (carpeta que el producto espera dentro de cualquier proyecto que lo adopte, incluido potencialmente `Axiom/` mismo).

## Discrepancia real conocida (no resuelta a la fecha de esta spec)

`Axiom/README.md`, `Axiom/docs/first-project-readiness.md` y `Axiom/scripts/verify-first-project-readiness.mjs` asumen que el propio repo `Axiom/` tiene, en su raíz, `axiom.spec/config/`, `axiom.spec/templates/`, `axiom.spec/target-axiom-skills/`, `axiom.spec/target-axiom-agents/`, `AGENTS.md` y `axiom.skills.lock`. Ninguno de esos paths existe hoy en este checkout de `Axiom/` (verificado con listado directo de la raíz del repo). Por tanto, `npm run readiness:first-project` fallaría hoy con `ENOENT` al intentar sembrar la baseline canónica. Ver detalle en [context/references/03-riesgos-y-brechas-conocidas.md](../context/references/03-riesgos-y-brechas-conocidas.md).

## Roadmap de rediseño (futuro, no implementado)

Existe un incremento de planificación (`specs/increments/INC-20260702-axiom-redesign-roadmap/`) que describe un rediseño posterior: separación de repos por rol (`sdd`/`spec`/`code`), registro global `~/.axiom/projects.yml`, manifiestos `axiom.yml` por repo, índices derivados/curados, MCP único por proyecto con `get_implementation_context`, TUI contextual y sistema de versionado/migraciones. Ese roadmap es planificación pura (24 incrementos secuenciados, INC-01 a INC-24) y **no** describe el estado actual del código — describe una evolución futura sobre el modelo hoy vigente (`axiom.yaml` + `.sdd/` + profile triple). No confundir ambos modelos al leer el resto de esta spec.

## Principios del producto

1. Spec first, implementación second.
2. Separación clara entre builder tooling y product runtime.
3. Axiom model primero; generated config después.
4. La trazabilidad enterprise es obligatoria (audit trail, telemetry sinks).
5. Deben soportarse los modos `local-only`, `standard` y `enterprise` (overlays operacionales).
6. Las capabilities están guiadas por profiles (`functionalProfile` + `operationalOverlay` + `adapterTarget`).
7. Las integraciones de tooling son adapters opcionales, con niveles de soporte explícitos (`multi-mode`, `single-mode`, `fallback-only`).
8. Minimizar el gasto de tokens sin reducir la auditabilidad.
9. Mantener una arquitectura modular y reemplazable (`Result<T,E>` sin excepciones, escritura atómica, path-guard por proyecto).
10. Evitar vendor lock-in.

## Roadmap y estado operativo

No existe hoy `plans/PLAN-PRODUCT-ROADMAP.md` en `Axiom.Spec/plans/` (la carpeta solo tiene un `README.md` de propósito, sin contenido de roadmap). El estado operativo consolidado y verificable vive en:
- `Axiom/README.md` (tabla de capas y paquetes operativos);
- `Axiom/docs/0015-…` a `0030-…` (increments post-MVP cerrados);
- `specs/increments/INC-20260702-axiom-redesign-roadmap/README.md` (roadmap futuro, no implementado).