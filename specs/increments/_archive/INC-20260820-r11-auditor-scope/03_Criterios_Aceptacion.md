# 03 Criterios de Aceptación

## Criterios de aceptación

### Happy path

1. Las tres copias contienen la misma regla sustantiva que prioriza `Axiom/` para evidencia runtime.
2. La regla delimita el uso de `Axiom.Spec/` y `Axiom.SDD/` y cita la excepción por petición explícita.

### Validaciones y errores

3. Un diff o comprobación determinista confirma que no hay divergencia del bloque distribuido.
4. La validación documental disponible y el build/test aplicable pasan o se documenta su ausencia.

### Permisos y visibilidad

5. Los límites de escritura y consolidación ya definidos por el auditor no se relajan.

### Estados y efectos observables

6. No cambia código de producto ni una integración MCP; solo se evita una conclusión de auditoría incorrecta.
7. El plan R-11 y el README del incremento registran la decisión, validación y consolidación no aplicable.
