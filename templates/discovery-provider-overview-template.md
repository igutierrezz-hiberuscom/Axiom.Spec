# Plantilla de overview de discovery y providers

Este documento se genera para explicar qué orden de discovery usa el proyecto, qué providers están activos y qué degradaciones quedan permitidas.

## Resumen ejecutivo

Indicar el profile funcional activo, el overlay operativo, el target principal y el discovery/provider profile aplicado.

## Discovery activo

| Orden | Modo | Uso esperado | Bloqueos |
|------|------|--------------|----------|
| 1 | filesystem / workspace | Camino principal cuando el workspace está disponible | N/A |
| 2 | axiom-gateway | Ruta homogénea entre entornos cuando aplica | Requiere gateway donde sea obligatorio |
| 3 | generated-snapshots | Degraded mode explícito | No habilita mutación incontrolada |

## Providers por capability

| Capability | Provider principal | Fallback | Notas |
|-----------|--------------------|----------|-------|
| `spec.read` | filesystem | axiom-gateway | Siempre project-scoped |
| `code.semanticNavigation` | serena | filesystem | Baseline MVP |
| `code.impactAnalysis` | codegraph | serena | Post-MVP opcional por defecto |
| `memory.contextRecall` | engram | generated-snapshots | Solo si está soportado por instalación |

## Restricciones activas

- `sdd` se usa en solo lectura en MVP.
- `spec` solo admite escritura brokerizada para control de implementación por rol.
- `backend-mcp` y `frontend-mcp` no forman parte del baseline del producto.

## Preguntas de verificación

1. ¿El target activo obliga a gateway-first?
2. ¿El degraded mode permitido bloquea la mutación prevista?
3. ¿El provider opcional aporta capacidad adicional sin redefinir el comportamiento funcional?

## Referencias

- `config/providers.yaml`
- `config/profiles.yaml`
- `capabilities/capability-provider-model.md`
- `integrations/project-scoped-integration-contract.md`