# 04 Interacciones UI

## Objetivo del documento

Describir la superficie y respuestas de `axiom self-update`.

## Superficie de comando

```
Usage: axiom self-update [options] [command]

Observa, planifica y ejecuta self-update transaccional; apply/recover requieren una instalación gestionada.

Commands:
  status [options]   Muestra el estado de la instalación y disponibilidad de updates
  check [options]    Comprueba si existe una versión publicada más reciente
  plan [options]     Genera un plan determinista de actualización
  apply [options]    Aplica un plan de actualización transaccional
  recover [options]  Recupera una transacción interrumpida o fallida
```

Ante invocación con flags legacy (ej: `axiom self-update --check`):
```
[axiom self-update] Use exactly one canonical self-update subcommand; legacy selector flags are not supported.
```
