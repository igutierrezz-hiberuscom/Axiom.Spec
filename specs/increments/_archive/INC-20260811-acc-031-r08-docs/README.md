# Reconciliar documentación de la zona R-08 (agents/skills)

> **Código**: INC-20260811-acc-031-r08-docs
> **Estado**: closed
> **Fecha de creación**: 2026-08-11
> **Tipo de cambio**: documentar
> **Acción de auditoría**: ACC-031
> **Plan padre**: `PLAN-REVISION-INTEGRAL-AXIOM`

## Objetivo

Corregir la documentación de la zona R-08 que no concuerda con los hechos
verificados, sin cambiar comportamiento ni lógica.

## Contexto y motivación

La auditoría R-08 verificó que los README y comentarios de la zona agents/skills
quedaron atrás de la evolución del catálogo:

- `Axiom/packages/agents/README.md` describe "3 entries oficiales", pero el
  catálogo real (`axiom.config/agents-catalog.yaml`) tiene **14 agents**.
- El comentario de `DEFAULT_DESIRED_SKILLS` en
  `Axiom/packages/skills/src/apply.ts` afirma que el set se mantiene
  sincronizado con `toolchain.yaml` (subset `mvp: true`), pero esa
  sincronización ya no existe y `serena`/`context7` tienen `mvp: false` en el
  catálogo real.
- El README de `@axiom/agents` describe `materializeAgentSet` como si fuera la
  vía principal de materialización, cuando el materializado real de agents en
  workspace lo hace `workspace-process-surfaces.ts`.

## Alcance tipado

- `Axiom/packages/agents/README.md`.
- `Axiom/packages/skills/src/apply.ts` (solo el comentario de
  `DEFAULT_DESIRED_SKILLS`).
- Cualquier otra afirmación de la zona R-08 que contradiga los hechos
  verificados (p. ej. referencias a conteos de catálogo en README de skills si
  existiera).

## Decisiones cerradas

- No se modifica el parámetro `mvp` del toolchain (es correcto y deliberado:
  ninguna tool requerida por defecto).
- No se cambia comportamiento ni lógica; solo documentación.
- El conteo real del catálogo es la fuente de verdad: 14 agents y 20 skills.

## No incluido

- No cambiar el materializador ni la vía real de materialización.
- No alterar `DEFAULT_DESIRED_SKILLS` ni su uso.
- No tocar la spec canónica salvo que una afirmación de la zona deba reflejarse
  ahí (se evalúa durante la integración).

## Criterios de aceptación

- [x] `Axiom/packages/agents/README.md` refleja el conteo real del catálogo
      (14 agents) y describe correctamente la vía real de materialización.
- [x] El comentario de `DEFAULT_DESIRED_SKILLS` en `apply.ts` ya no afirma una
      sincronización inexistente con `toolchain.yaml`.
- [x] No hay cambios de comportamiento ni de lógica.
- [x] La validación disponible (build/tests) pasa sin regresiones nuevas.

## Resultado

Cambios realizados (solo documentación):

- `Axiom/packages/agents/README.md`: la sección "Catálogo runtime" ahora
  refleja los **14 agents** reales del catálogo (lista completa con sus
  roles), en lugar de "3 entries oficiales". La sección "Materialización"
  ahora describe que la vía real de materialización en workspace es
  `workspace-process-surfaces.ts`, y que `materializeAgentSet` del paquete
  es una utilidad directa del catálogo que no es la vía principal invocada
  por los comandos (se usa en tests y como ejemplo en el README).
- `Axiom/packages/skills/src/apply.ts`: se reescribió el comentario de
  `DEFAULT_DESIRED_SKILLS` para describirlo como un set de skills deseadas
  por defecto en el scaffolding de workspace, sin afirmar sincronización con
  `toolchain.yaml` ni con su subset `mvp`.

No se modificó comportamiento, lógica, catálogos, el parámetro `mvp` del
toolchain, ni `DEFAULT_DESIRED_SKILLS`/su uso.

## Validación

- `npm run build` (`tsc -b`) desde `Axiom/`: **OK**.
- `npx vitest run packages/agents packages/skills`: **9 test files, 117
  tests, todos pasan** (sin regresiones).
- `get_errors` sobre los archivos editados: sin errores.

No se detectaron fallos nuevos introducidos por este incremento.

## Revisión independiente

La revisión independiente (`axiom-review`) confirmó el conteo (14 agents) y la
vía de materialización, y validó el comentario de `DEFAULT_DESIRED_SKILLS`.
Encontró un hallazgo documental: 3 roles de agents en el README contradecían
el catálogo real (`axiom.config/agents-catalog.yaml`):

- `axiom-tester` → el README decía `product-review`; el catálogo dice
  `sdd-orchestration`.
- `axiom-role-implementer` → el README decía `planning`; el catálogo dice
  `sdd-orchestration`.
- `axiom-tech-context` → el README decía `sdd-orchestration`; el catálogo dice
  `planning`.

Se corrigieron los tres roles en `Axiom/packages/agents/README.md` para
alinearlos con el catálogo. No es un blocker funcional (no hay lógica), pero
era una desviación del objetivo del incremento (reconciliar documentación con
hechos verificados). Tras la corrección, el incremento queda OK.

## Integración de spec

No se requiere integración de conocimiento estable en la spec canónica: los
cambios son correcciones documentales de la zona R-08 que reflejan hechos ya
verificados del catálogo (14 agents, 20 skills). No hay afirmaciones nuevas
que consolidar en `Axiom.Spec/specs/`.