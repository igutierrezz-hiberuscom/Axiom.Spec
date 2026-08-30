# r11-auditor-scope

> **Código**: INC-20260820-r11-auditor-scope
> **Estado**: Gestionado por Axiom/Core; consultar `metadata.yml` para el estado de lifecycle.
> **Fecha de creación**: 2026-08-20
> **Tipo de cambio**: documentar

## Resumen

Aclara el alcance probatorio del agente `axiom-runtime-auditor` en sus tres superficies distribuidas: audita el runtime `Axiom/`; consulta `Axiom.Spec/` para intención y consolidación; y usa `Axiom.SDD/` para reglas y hospedaje del agente, no como evidencia de comportamiento del producto.

## Contexto y motivación

ACC-047 registró que configuraciones locales de herramientas bajo `Axiom.SDD/` podían confundirse con integraciones del runtime. El contrato debe impedir esa inferencia sin restringir una petición explícita del usuario que sí incluya Axiom.SDD.

## Alcance

### Incluido

- Alinear las tres copias de `axiom-runtime-auditor` con una regla explícita de evidencia y ownership.
- Preservar límites de escritura, explicación en castellano y consolidación del auditor.
- Verificar que las tres superficies no divergen.

### Excluido

- Cambiar el runtime Axiom, su configuración MCP o las reglas de otro agente.
- Introducir generadores, índices o arquitectura nueva.
- Convertir Axiom.SDD en fuente de evidencia del producto.

## Documentos del incremento

Requisitos, modelo de alcance, criterios y ausencia de UI se detallan en los cuatro documentos acompañantes.

## Dudas abiertas

Ninguna. Las tres copias son el alcance solicitado; si existe un generador oculto, se detectará antes de editar para no generar deriva.

## Decisiones funcionales cerradas

1. El comportamiento se prueba principalmente en `Axiom/`.
2. `Axiom.Spec/` es intención, contraste y destino de consolidación estable.
3. `Axiom.SDD/` solo aporta reglas de trabajo y aloja el agente; su config local no demuestra una integración de Axiom salvo petición explícita.

## Consolidación en la spec general

No cambia requisito de producto ni contexto técnico del runtime. El plan R-11 registrará la acción completada; no se crearán claims nuevos en specs 00..08 o `context/**`.

## Estrategia E2E

Comparar las tres copias byte a byte en su bloque de alcance tras el cambio y ejecutar las pruebas o validación documental disponible.

## Trazabilidad y fuentes

ACC-047 del plan R-11 y las tres superficies: `.kiro/steering/axiom-runtime-auditor.md`, `.kiro/agents/axiom-runtime-auditor.md`, `.github/agents/axiom-runtime-auditor.agent.md`.

## Estado de validación humana

Implementación terminada. La comprobación determinista verificó el bloque normativo idéntico en las tres superficies; `git diff --check` no reportó errores (solo avisos LF/CRLF). No hay build, runner ni suite aplicable en Axiom.SDD para estas instrucciones Markdown. Review independiente: sin blockers; no aplica consolidación en specs 00..08 ni `context/**`.
