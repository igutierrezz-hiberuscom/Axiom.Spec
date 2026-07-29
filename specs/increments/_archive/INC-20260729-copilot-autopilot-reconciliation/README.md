# Incremento: Axiom Autopilot para Copilot con reconciliación normativa

Estado: closed
Fecha: 2026-07-29

## Objetivo

Portar el workflow `axiom-autopilot` de Claude Code a `Axiom.SDD` como personalización nativa de GitHub Copilot y reforzar su consolidación final para que reconcilie afirmaciones activas de la spec y del contexto técnico con el estado real resultante de cada incremento.

## Contexto

El workflow original vive en `.claude/commands/axiom-autopilot.md` y `.claude/skills/axiom-autopilot.md`. Axiom.SDD no tenía todavía custom agents ni skills Copilot equivalentes. Además, la integración documental del playbook debía expresar de forma inequívoca que una afirmación retirada no puede permanecer como contrato vigente solo porque exista una nota `SUPERSEDE`: debe actualizarse, eliminarse o reencuadrarse como histórica.

## Alcance

- Crear la skill `.github/skills/axiom-autopilot/SKILL.md`.
- Crear los custom agents `axiom-autopilot`, `axiom-increment` y `axiom-review` bajo `.github/agents/`.
- Hacer descubrible el workflow desde `.github/copilot-instructions.md`.
- Crear el prompt `/axiom-autopilot` como equivalente invocable del comando Claude.
- Actualizar las referencias de rutas stale del prompt Copilot de incremento.
- Incorporar en el playbook el ledger de claims, la eliminación o modificación de afirmaciones activas obsoletas, la conservación histórica explícita y la reconciliación verificable de `Axiom.Spec/context/**`.

## Fuera de alcance

- No modificar el runtime de `Axiom/`.
- No modificar directamente los archivos `.claude/` existentes.
- No crear un índice obligatorio ni una capa de orquestación enterprise.
- No ejecutar una tanda real de incrementos como parte de este incremento.
- No ejecutar comandos Git mutantes.

## Criterios de aceptación

- [x] La skill Copilot tiene frontmatter válido, coincide con el nombre de su carpeta y contiene el playbook completo de autopilot.
- [x] El agente `axiom-autopilot` es invocable y referencia la skill y los workers permitidos.
- [x] Los agentes `axiom-increment` y `axiom-review` son delegables y tienen límites de edición claros.
- [x] Existe un prompt `/axiom-autopilot` que delega en el agente principal.
- [x] El workflow exige modificar o eliminar claims activos obsoletos y no acepta `SUPERSEDE` como única reconciliación.
- [x] El workflow exige revisar y actualizar `Axiom.Spec/context/**` con fuentes verificables, o declarar que no aplica.
- [x] Las rutas del prompt Copilot de incremento usan `Axiom.Spec/specs/increments/`.
- [x] Las personalizaciones nuevas pasan comprobación estructural, diagnósticos y `git diff --check`.
- [x] El cambio queda documentado sin commit.

## Preguntas abiertas

Ninguna bloqueante.

## Supuestos y decisiones

1. Se usan custom agents además de una skill porque Copilot necesita una superficie delegable para el orquestador y workers separados para conservar el modelo del playbook original.
2. Se crea también un prompt `/axiom-autopilot` para conservar la ergonomía del comando Claude sin duplicar el procedimiento: el prompt remite al agente y la skill contiene la lógica.
3. La adaptación es tooling/documentación; no requiere actualizar `Axiom.Spec/context/**` porque no cambia el estado técnico del runtime de `Axiom/`.
4. La carpeta activa anterior `INC-20260729-autopilot-technical-context-step` no se reutiliza porque describe el cambio Claude ya cerrado; esta adaptación tiene criterios y superficies Copilot distintos.

## Notas de implementación

La skill separa la integración de requisitos (`specs/00..08`) de la descripción del estado construido (`context/**`). En ambos casos exige un inventario de claims afectados, corrección en el lugar de las afirmaciones normativas obsoletas y preservación histórica únicamente bajo una marca explícita. Los agentes delegados no pueden editar esas superficies finales; el orquestador conserva esa responsabilidad para evitar consolidaciones parciales.

## Validación

No se ejecutó build ni suite de runtime porque el incremento solo modifica Markdown de configuración de Copilot, instrucciones de workflow y documentación canónica. Se ejecutó:

- comprobación de frontmatter y routing de los seis archivos Copilot con frontmatter;
- comprobación de existencia de las nueve rutas citadas por la integración;
- `get_errors` sobre la skill, los agentes, los prompts, las instrucciones y los capítulos de spec modificados: sin errores;
- `git diff --check` en `Axiom.SDD` y `Axiom.Spec`: sin errores;
- búsqueda de rutas legacy en las superficies nuevas o modificadas: sin coincidencias;
- revisión de que los agentes delegados no puedan editar `specs/00..08`, `context/**` ni archivar incrementos.

## Resultado

Implementado y verificado. Axiom.SDD dispone ahora de una skill `axiom-autopilot`, un agente orquestador, workers delegables para incremento y revisión, y un prompt `/axiom-autopilot`. La integración final exige reconciliar claims activos en la spec y en el contexto técnico, retirar afirmaciones obsoletas y conservar historia solo bajo una marca explícita. También se corrigieron las rutas del prompt de incremento y se actualizó el mapa de repositorios de las instrucciones Copilot.

## Integración en la spec general

- `Axiom.Spec/specs/04_Flujos_SDD_y_Ciclo_de_Vida.md`: se actualizó la superficie `/axiom-autopilot` para incluir Claude Code y GitHub Copilot, y se documentó la reconciliación activa de claims y contexto técnico.
- `Axiom.Spec/specs/08_Glosario.md`: se actualizó la definición de `/axiom-autopilot` con sus superficies Copilot, su carácter de tooling de desarrollo y la regla de retirar claims obsoletos del contrato activo.
- `Axiom.Spec/context/**`: ninguno; el cambio no altera el estado técnico del runtime de `Axiom/`.
