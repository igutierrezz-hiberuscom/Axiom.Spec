# 02 Cambios de Modelo

## Objetivo del documento

Describir el cambio como precisión de contrato distribuido, sin modelo runtime nuevo.

## Entidades o estructuras afectadas

Tres archivos de definición del mismo agente en `Axiom.SDD`: steering Kiro, agente Kiro y agente GitHub.

## Contratos o estados afectados

Se añade una regla explícita de alcance/evidencia. No cambian inputs, outputs, tools, estados lifecycle ni permisos del auditor.

## Notas de compatibilidad

Una petición explícita del usuario puede ampliar la evidencia a Axiom.SDD; el default deja de inferir integraciones del producto desde ese repositorio.
