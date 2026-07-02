# Plantilla de getting started

Este documento se genera para explicar el camino mínimo de arranque de un proyecto Axiom sin obligar al lector a reconstruir manifests, profiles y providers a mano.

## Camino rápido

1. Confirmar producto, repo actual y perfil funcional activo.
2. Ejecutar `axiom init` o validar que el proyecto ya fue inicializado.
3. Revisar el target principal, el discovery/provider profile y la overlay operativa aplicada.
4. Ejecutar `axiom doctor` antes de cualquier mutación.

## Entradas mínimas

| Dato | Origen esperado | Ejemplo |
|------|-----------------|---------|
| productId | `product.manifest` | `axiom` |
| repoId | `repo.manifest` | `axiom-spec` |
| functionalProfile | `config/profiles.yaml` | `product-owner` |
| operationalOverlay | `config/profiles.yaml` | `standard` |
| discoveryProviderProfile | `config/providers.yaml` | `filesystem-first` |
| primaryTarget | `config/profiles.yaml` | `copilot-vscode` |

## Estado esperado tras init

- Existe `product.manifest` con topología global del producto.
- Existe `repo.manifest` compartido como contrato del repo.
- `.sdd/local/` queda reservado para configuración estrictamente local.
- La resolución de proyecto es explícita y auditable.

## Verificación

- `axiom doctor` pasa sin errores de contratos ni de aislamiento.
- El proyecto resuelve el repo actual sin ambigüedad.
- El profile funcional y el target principal no contradicen las políticas activas.

## Siguientes pasos

1. Si el repo es de spec, continuar con refinado, planificación o validación funcional.
2. Si el repo es técnico, confirmar plan aprobado antes de `axiom start`.
3. Si el target requiere gateway-first, validar disponibilidad del gateway antes de mutar.

## Referencias

- `config/onboarding.yaml`
- `config/providers.yaml`
- `config/profiles.yaml`
- `config/local-overlay-policy.yaml`