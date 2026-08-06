# Context

## Propósito

Este contexto acompaña al incremento con hechos técnicos verificables sobre
la implementación del resolver y su frontera de compatibilidad.

## Qué puede vivir aquí

- La union pública `ProjectMode` y la función de normalización.
- La diferencia entre `rawConfig` y `ProjectResolution.mode`.
- Los consumers revisados y las pruebas que certifican v1/v2.

## Qué no debe vivir aquí

- Diseño futuro de gateway o enterprise.
- Reglas de topología o tool-routing que no dependan de `ProjectResolution`.
- Historia de implementaciones retiradas presentada como estado actual.

## Estructura sugerida

Fuente primaria: `Axiom/packages/project-resolution/src/resolver.ts`.
Consumidores de control: `Axiom/packages/doctor/src/checks.ts` y las pruebas
de `Axiom/packages/project-resolution/tests/resolver.test.ts`.
