# Plantilla de onboarding de miembro

Este documento se genera para incorporar a una persona nueva al proyecto sin tocar estado global fuera del contrato compartido y de la overlay local permitida.

## Objetivo

Dejar a la persona operativa en el repo correcto, con el profile funcional esperado, el target principal resuelto y el doctor en verde.

## Camino rápido

1. Abrir el repo correcto y verificar `product.manifest` y `repo.manifest`.
2. Ejecutar `axiom join` para registrar el binding local del miembro.
3. Revisar los defaults heredados de profile, overlay y discovery/provider profile.
4. Ejecutar `axiom doctor` y resolver cualquier bloqueo antes de trabajar.

## Qué puede escribir join

- `.sdd/local/member.yaml`
- `.sdd/local/workspace-binding.yaml`

## Qué no puede cambiar join

- Identidad del producto.
- Identidad del repo.
- Políticas globales del producto.
- Scope MCP del producto.

## Checklist de incorporación

- El repo actual pertenece al producto esperado.
- El functional profile activo coincide con el rol operativo previsto.
- El target principal está soportado para el profile actual.
- No hay ambigüedad de proyecto antes de mutar.
- `.sdd/local/` está protegida por gitignore.

## Problemas frecuentes

| Problema | Señal | Acción |
|----------|-------|--------|
| Proyecto ambiguo | `doctor` bloquea mutación | Seleccionar explícitamente el proyecto antes de continuar |
| Provider requerido no disponible | degraded mode no permitido | Cambiar target/profile o habilitar el provider soportado |
| Overlay local contradice shared manifest | `doctor` marca contaminación | Reconciliar `.sdd/local/` y repetir validación |

## Referencias

- `config/onboarding.yaml`
- `config/local-overlay-policy.yaml`
- `integrations/project-scoped-integration-contract.md`