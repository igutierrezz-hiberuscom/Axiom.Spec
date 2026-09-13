# Rol de plan: builder

## Rol

- nombre del rol: builder
- repositorio(s) asignado(s): axiom-code-builder (`Axiom`), con operación sobre artefactos del repositorio canónico `Axiom.Spec` mediante comandos de Axiom en el incremento 4

## Objetivo del rol en este plan

Ejecutar los seis incrementos del lote R-15 en el orden fijado, dejando: documentación activa veraz, manual único completo y vigilado por prueba, una sola fuente de plantillas, un solo formato y raíz de decisiones, verificación documental ejecutable en el cierre, y el manual distribuido a los proyectos.

## Alcance incluido

- `Axiom/README.md`, `Axiom/docs/**` y los README de paquete afectados.
- `Axiom/axiom.spec/templates/` como fuente única, y el código y tests que la referencian: `packages/workflow/src/artifact-skeleton.ts`, `packages/workflow/tests/artifact-skeleton.test.ts`, `packages/cli-commands/src/commands/workspace-adapter-templates.ts`, `packages/adapters/{opencode,claude-code,codex,antigravity}/src/`.
- El ejecutor de transiciones gobernado de `@axiom/workflow` y los wrappers de cierre e integración, para la verificación documental.
- El inventario de siembra de `apps/cli/src/commands/workspace-setup.ts` y el camino de adopción y actualización, para la distribución.
- Nuevas pruebas en la suite del repositorio: cobertura documental, test dorado ampliado, verificación de cierre y distribución.
- En `Axiom.Spec`: artefactos de decisión, unificación de decisiones, retirada de raíces legacy y reconciliación de `README.md` y `specs/README.md`, siempre con comandos de Axiom para lo gestionado.

## Fuera de alcance

- Cambiar el comportamiento de cualquier comando existente.
- Redefinir semántica de archive, contrato de QA o resolución de workflows (`ACC-041`, `ACC-043`, `ACC-045`).
- Tocar el contenido de `Axiom.Spec/specs/manuales/**` más allá del índice estructural.
- Retirar plantillas sin consumidor conocido, renombrar `axiom.spec/` y corregir los residuos del archivado.
- Editar `metadata.yml`, índices o receipts a mano.

## Dependencias

- `D-01` (estructura del manual) cierra en el incremento 1 y condiciona 2, 5 y 6.
- El incremento 5 depende de que 1 y 2 estén cerrados; el 6 depende de 2 y 5.
- El incremento 4 se ejecuta después del 3 para tocar `Axiom.Spec/README.md` una sola vez.
- `ACC-035` y `ACC-037` de R-07 comparten superficie con el incremento 4: hay que declarar la frontera o coordinar la ejecución.

## Validaciones y evidencias esperadas

Por incremento: `npm run build`, suites dirigidas del área tocada, `npm run doctor`, `npm run readiness:first-project`, `git diff --check` y, cuando aplique, `axiom index validate`. Al cierre del lote: suite completa de Vitest.

Evidencias específicas que deben quedar registradas: barridos de enlaces y de vocabulario retirado, demostración de que la prueba de cobertura falla ante un comando sin documentar, comparación archivo por archivo de las plantillas antes de retirar la copia canónica, recuento explicable de `axiom index validate`, casos de aviso y bloqueo del gate, e idempotencia por contenido de la distribución.

## Bloqueos y observaciones

- Si `D-01` no se cierra con claridad, el incremento 2 no arranca: elegir formato en caliente obligaría a rehacer páginas.
- Si el volumen del incremento 2 resulta excesivo, la unidad de cobertura se mantiene en familia de comandos, no en sub-comando, y así se declara en `D-01`.
- Si al documentar aparece una promesa de comando sin implementación, se cruza con `ACC-039` en lugar de resolverse aquí.
- El gate del incremento 5 entra en modo aviso antes de bloquear, para no congelar artefactos en vuelo.
