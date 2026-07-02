# Plantilla de troubleshooting

Este documento se genera para guiar el diagnóstico de fallos de init, join, doctor, configure, start o sync sin inventar reparación fuera de los contratos vigentes.

## Señal observada

Describir el síntoma con una frase verificable.

## Diagnóstico rápido

1. Confirmar si el problema aparece en `init`, `join`, `doctor`, `configure`, `start` o `sync`.
2. Identificar si el bloqueo es de manifests, providers, overlay local, target o MCP scope.
3. Verificar si el modo degradado está permitido por el profile activo.

## Tabla de diagnóstico

| Área | Qué revisar | Evidencia esperada |
|------|-------------|--------------------|
| Manifests | `product.manifest`, `repo.manifest` | Identidad y paths coherentes |
| Providers | `config/providers.yaml` | Provider principal o fallback permitido |
| Overlay local | `.sdd/local/` | Sin overrides prohibidos |
| Scope MCP | contrato project-scoped | `sdd` read-only y `spec` brokerizado |
| Target | profile y target activos | Compatibilidad con el entorno |

## Acciones seguras

- Repetir `axiom doctor` tras cada corrección.
- Corregir primero ambigüedad de proyecto antes de cualquier mutación.
- Tratar cualquier degraded mode como explícito y auditable.

## Acciones prohibidas

- Mutar specs arbitrariamente por MCP.
- Saltar el doctor en comandos mutantes.
- Reescribir shared manifests desde `.sdd/local/`.

## Escalado

Escalar cuando el problema exija cambiar policy global, manifests compartidos o contrato de providers.

## Referencias

- `config/onboarding.yaml`
- `config/providers.yaml`
- `config/local-overlay-policy.yaml`
- `scripts/doctor-validate-contracts.mjs`