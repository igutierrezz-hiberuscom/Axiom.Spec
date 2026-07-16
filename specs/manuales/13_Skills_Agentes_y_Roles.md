# 13. Skills, agentes y roles instalados

Qué instala Axiom exactamente en cada repo según su rol, y cómo tu proyecto
añade SU propia profundidad (patrones de arquitectura, convenciones de UI,
reglas de permisos/multi-tenant, comandos de build/test) sin que esa
profundidad viva hardcodeada en el producto Axiom.

## Qué instala Axiom por rol de repo

`surfaceIdsForRole(repoRole)` (`workspace-process-surfaces.ts`) decide qué
superficies de proceso se materializan en cada repo, según su rol declarado
(`axiom.yaml#role`):

| Rol de repo | Superficies materializadas | Propósito de cada una |
| ----------- | --------------------------- | ---------------------- |
| `sdd` (control) | `axiom-sdd-orchestrator` | Orquesta el ciclo spec-first completo: decide en qué fase está el trabajo y encadena analista → arquitecto → implementador sin saltarse pasos. |
| | `axiom-phase-reviewer` | Gate de calidad de solo lectura entre fases (spec/plan/código): barrido hasta seco, ledger de hallazgos, veredicto OK/KO. |
| | `axiom-qa-validator` | Plan de pruebas documental derivado de los criterios de aceptación, con trazabilidad 1:N criterio → caso de prueba. |
| `spec` | `axiom-spec-author` | Flujo de analista: crea/expande un incremento o bug con cobertura funcional (`CF-xx`) verificable. |
| | `axiom-role-planner` | Flujo de arquitecto: un plan por rol registrado, con `targetRepos` y `allowedWriteScope` mínimo. |
| | `axiom-spec-integrator` | Flujo de consolidación: integra el conocimiento durable de un incremento/bug ya implementado a la spec canónica y lo archiva (confirm-gated). |
| | `axiom-tech-context` | Autoría/mantenimiento del contexto técnico del proyecto (documento maestro + auxiliares) y detección de spec-drift contra el código real. |
| `code` | `axiom-role-implementer` | Construye el slice de ESE rol funcional en su repo, dentro del `allowedWriteScope`, validado y con git confirm-gated. |

En resumen: el repo de control (`sdd`) hospeda la orquestación más las gates
de revisión/QA; el repo de spec hospeda la autoría y la consolidación; cada
repo de código hospeda únicamente la implementación de su rol funcional.

Cada superficie se materializa en varias formas a la vez: la portable
(`.axiom/{agents,commands,skills}/<id>.md|SKILL.md`, siempre presente) y, si
el adapter correspondiente está seleccionado, la forma nativa
(`.claude/agents/<id>.md`, `.opencode/agents/<id>/{SKILL,AGENT}.md`, o un
archivo de instrucciones role-diferenciado para cursor/copilot). Todas
comparten el mismo cuerpo, con los placeholders `{role}`/`{repoPath}`/
`{specRepo}`/`{projectId}` ya rellenados para ese repo concreto.

## Disciplinas transversales (skills reutilizables)

Además de las superficies de flujo, el catálogo trae disciplinas
**transversales**: no son un flujo en sí mismas, sino reglas que varias
superficies referencian por id para no reimplementar la misma lógica cada
vez.

| Skill | Qué exige |
| ----- | --------- |
| `axiom-structured-doubts` | Parar-y-preguntar ante ambigüedad o conflicto entre artefactos: formato `DUDA [tipo]` / `CONFLICTO [ámbito]`, opciones cerradas a/b/c, un default justificado por evidencia, y confirmación explícita antes de avanzar. Nunca se autorresuelve por silencio. |
| `axiom-functional-checklist-coverage` | La checklist `CF-xx`: un id por comportamiento observable atómico, con rol afectado y documento fuente; repartida sin diluirse por el plan; reportada al cierre con un vocabulario fijo (`cubierto`/`parcial`/`no aplica`/`bloqueado`/`pendiente`) en una tabla `CF / Rol / Estado / Evidencia`. |
| `axiom-plan-drift-alignment` | Comparar `plan.specVersion` contra la `specVersion` viva y clasificar el impacto por rol (`none`/`review`/`replan`/`reopen`) cuando la spec cambia después de que ya existía un plan — nunca se implementa sobre un plan con drift sin clasificar. |
| `axiom-role-close-doc` | El cierre documental técnico que sigue al cierre técnico de un rol: distingue qué del diff real es transversal (va al contexto técnico) de qué es regla funcional local (ya vive en la spec), y produce siempre tres bloques — cambios al contexto técnico, recomendaciones para skills/agentes, alertas o decisiones pendientes. |
| `axiom-phase-reviewer` | (También superficie instalada en el repo `sdd`.) El gate de revisión por fase que consume `axiom-structured-doubts` (duda en vez de resolver por su cuenta) y `axiom-functional-checklist-coverage` (la tabla de cierre es parte de lo que verifica). |

Estas disciplinas están catalogadas en `axiom.config/skills-catalog.yaml`
junto al resto, pero no aparecen en la tabla de arriba como superficies
propias por rol: viven referenciadas, por id, dentro del cuerpo de
`axiom-spec-author`, `axiom-role-planner`, `axiom-role-implementer` y
`axiom-phase-reviewer` — es el mecanismo por el que Axiom evita duplicar la
misma regla en cuatro sitios distintos.

## Agentes del catálogo

`axiom.config/agents-catalog.yaml` (14 entradas) agrupa cada agente por
`role` (bucket funcional, no el rol de repo). Los más relevantes para el
ciclo diario:

- **`sdd-orchestration`**: `axiom-sdd-orchestrator` (encadena los 3 flujos),
  `axiom-role-implementer` (construye el slice del rol), `axiom-explorer` y
  `axiom-tester` (exploración read-only y ejecución/interpretación de
  validación).
- **`planning`**: `axiom-spec-author`, `axiom-role-planner`,
  `axiom-spec-integrator`, `axiom-tech-context`, y el histórico
  `axiom-spec-planner`.
- **`product-review`**: `axiom-phase-reviewer` (gate bloqueante spec/plan/
  código), `axiom-qa-validator` (plan de pruebas documental), y dos revisores
  opcionales — `axiom-reviewer` (calidad general) y `axiom-security-reviewer`
  (ahora con cuerpo real: checklist de 10 familias de riesgo, escala de
  severidad `CRITICAL`/`HIGH`/`MEDIUM`/`LOW`, formato de hallazgos `SEC-NNN`,
  y carácter explícitamente **opcional y no bloqueante** — a diferencia de
  `axiom-phase-reviewer`, nunca decide por sí solo el veredicto de cierre).

## Canal de inyección por proyecto (lo específico de tu stack va aquí, no en el producto)

Axiom se mantiene deliberadamente genérico y agnóstico de adapter/stack —el
mismo principio que sigue gentleai—: ningún cuerpo de skill o agente del
catálogo asume un lenguaje, framework, o convención de UI concretos. Un
sistema role-especializado como KVP25 consigue su riqueza hardcodeando reglas
de stack dentro de cada agente por rol (p. ej. "el backend usa .NET con este
patrón de repositorio concreto"); Axiom en cambio deja esas reglas fuera del
producto y las espera como **dato de proyecto**, para poder instalarse sobre
cualquier stack sin perder profundidad. Los tres puntos de inyección son:

1. **`axiom.config/skills-index/<role>.yaml`** — el índice de skills por rol
   que cada repo trae (esquema en `@axiom/skills`, `role-index.ts`): un
   array `mandatory` (skills que todo agente actuando en ese rol debe cargar
   siempre) y un array `available` (skills bajo demanda, seleccionables por
   `tags`). Cuando Axiom materializa las superficies de proceso, augmenta
   este índice con las suyas (`axiom-role-implementer`, etc.); tu proyecto
   añade AQUÍ sus propias skills de rol (convenciones de código, patrones de
   arquitectura del repo, reglas de permisos/multi-tenant específicas) como
   entradas adicionales de ese mismo índice. `sdd.skillIndexRead` y
   `implementationContextRead.mandatory.repoSkills` son quienes lo sirven en
   runtime a cada superficie.
2. **El contexto técnico del proyecto** (propiedad de `axiom-tech-context`,
   con `axiom-context-persistence` como skill de soporte para la
   persistencia entre sesiones) — el documento maestro + auxiliares que
   describe la arquitectura real, los contratos entre roles, y las
   restricciones generales del proyecto, siempre verificado contra código
   real (nunca inventado; lo no verificable se marca
   `[PENDIENTE DE VERIFICAR]`). Es donde vive el conocimiento transversal
   que `axiom-role-close-doc` decide consolidar tras cada cierre de rol.
3. **Skills de rol propias del proyecto** — cualquier skill que el equipo
   escriba y registre en el `skills-index` de su rol (más allá de las
   seed/proceso que Axiom ya instala), para capturar reglas de negocio,
   comandos de build/test, o convenciones de UI que solo aplican a ESE
   proyecto.

Contraste explícito: en un sistema role-especializado, cambiar de stack
implica reescribir el agente; en Axiom, cambiar de stack implica solo
actualizar el `skills-index` y el contexto técnico del proyecto — el cuerpo
de las superficies instaladas no cambia.

### Cómo extender (checklist corto)

1. Identifica si la regla es de proceso (va en una skill/agente del
   catálogo — raro, requiere increment propio) o de proyecto (el caso común:
   va en `skills-index`/contexto técnico).
2. Para una convención o patrón nuevo: añade o actualiza una entrada
   `mandatory`/`available` en `axiom.config/skills-index/<role>.yaml` del
   repo correspondiente, con su `reason`/`summary`.
3. Para una regla de arquitectura/permisos transversal: documéntala en el
   contexto técnico del proyecto (vía `axiom-tech-context`), no en la spec
   funcional del incremento de turno.
4. Nunca dupliques una regla funcional puntual en el contexto técnico —esa
   vive en la spec del incremento/bug (ver
   [04_Generar_Spec_y_Contexto_Tecnico.md](04_Generar_Spec_y_Contexto_Tecnico.md)).

## Ciclo con las gates

Recorrido end-to-end de un incremento, con las gates opcionales marcadas
explícitamente:

```
axiom-spec-author (analista, repo spec)
        │
        ▼
  [axiom-phase-reviewer: lente spec]   ← gate opcional
        │
        ▼
axiom-role-planner (arquitecto, repo spec)
  con análisis de alcance opcional por dimensión si la complejidad lo exige
        │
        ▼
  [axiom-phase-reviewer: lente plan]   ← gate opcional
        │
        ▼
axiom-role-implementer (por cada rol, repo de código)
        │
        ▼
  [axiom-phase-reviewer: lente código]      ← gate opcional (bloqueante si KO)
  [axiom-qa-validator: plan de pruebas]     ← gate opcional
  [axiom-security-reviewer: hallazgos SEC]  ← gate opcional, nunca bloquea por sí solo
        │
        ▼
axiom-role-close-doc (cierre documental técnico por rol)
  + axiom-tech-context (si hay drift o cambio transversal que consolidar)
        │
        ▼
axiom-spec-integrator (consolidar conocimiento durable + archivar, confirm-gated)
```

Las tres gates de revisión (`axiom-phase-reviewer`, `axiom-qa-validator`,
`axiom-security-reviewer`) son consultivas: solo `axiom-phase-reviewer`
puede emitir un `VEREDICTO: KO` que bloquea el avance de fase; las otras dos
son input para quien decide el cierre, nunca la última palabra por sí
solas. Ver también [09_Revisiones.md](09_Revisiones.md) para el detalle del
review de write-scope y el flujo `axiom-review`.

## Relacionado

- [05_Incrementos.md](05_Incrementos.md)
- [07_Planes.md](07_Planes.md)
- [08_Implementacion.md](08_Implementacion.md)
- [09_Revisiones.md](09_Revisiones.md)
- [10_Archivado.md](10_Archivado.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
- [02_Configuracion.md](02_Configuracion.md)
- [04_Generar_Spec_y_Contexto_Tecnico.md](04_Generar_Spec_y_Contexto_Tecnico.md)
- [../05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md)
