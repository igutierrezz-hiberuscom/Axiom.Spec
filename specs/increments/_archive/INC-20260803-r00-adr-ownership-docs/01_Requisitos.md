# Requisitos

## RQ-001. Hogar canónico de ADR

Los ocho ADR definidos por R-00 deben vivir en `Axiom.Spec/decisions/` y conservar su contenido byte-a-byte.

## RQ-002. Ownership documental

La documentación activa debe describir las raíces reales de `Axiom.Spec/` y eliminar la afirmación obsoleta sobre `general-spec.md`.

## RQ-003. Referencias activas

Las referencias actuales deben apuntar a `Axiom.Spec/decisions/`; las rutas antiguas solo pueden aparecer como historia explícita.

## RQ-004. Alcance runtime

La migración no debe cambiar comportamiento de runtime, artefactos generados ni semántica YAML.

## RQ-005. Evidencia y cierre

El resultado debe registrar validación, revisión independiente, integración documental y archivo sin mutaciones Git.
