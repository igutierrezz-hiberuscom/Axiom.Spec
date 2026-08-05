# Requisitos

## RQ-001. Entry point convencional

El paquete debe publicar su JavaScript y declaraciones desde `dist/index.js`
y `dist/index.d.ts`, y sus campos `main` y `types` deben apuntar a esas rutas.

## RQ-002. Ownership unico

Cada fuente compartida debe tener un unico owner de compilacion. La solucion no
puede depender de compilar el mismo modulo desde `apps/cli` y desde el paquete.

## RQ-003. Runtime distribuido

La CLI, TUI y MCP deben resolver los comandos desde los entrypoints construidos,
no desde aliases que solo entiende TypeScript.

## RQ-004. Compatibilidad de comportamiento

Las funciones publicas y resultados existentes se mantienen; solo se normaliza
la organizacion de fuentes y artefactos.

## RQ-005. Evidencia

Debe existir validacion ejecutable del build y de una carga runtime de la CLI
compilada, ademas de pruebas dirigidas para los consumidores afectados.
