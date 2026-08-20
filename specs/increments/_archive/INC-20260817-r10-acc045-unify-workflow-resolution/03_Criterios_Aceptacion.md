# 03 Criterios de Aceptación

## Criterios de aceptación

- [x] No existe una segunda copia editable a mano del default de workflows.
- [x] CLI, launcher, MCP e integrate usan el mismo resolvedor.
- [x] Ausencia, YAML válido, YAML inválido y schema no soportado tienen resultados inequívocos y probados.
- [x] Ningún caller aplica fallback silencioso para un YAML presente inválido.
- [x] Las pruebas verifican paridad de subcomandos alcanzables con el grafo declarado, incluido el override efectivo del launcher.
- [x] Build focal de `@axiom/workflow`, baterías focalizadas y build global pasan; no queda bloqueo de compilación fuera de alcance.

### Happy path

Un proyecto sin YAML resuelve el default distribuido; uno con YAML válido usa su grafo; uno inválido recibe un error accionable.

### Validaciones y errores

Los errores de parseo y schema se propagan a todos los canales sin cambiar de grafo.

### Permisos y visibilidad

Sin cambios.

### Estados y efectos observables

Resolver no ejecuta transiciones ni efectos; devuelve configuración o error.
