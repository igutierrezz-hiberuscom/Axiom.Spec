# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-082-01: Pantalla de Model Routing operativa
El launcher incluye una vista accesible desde la barra superior que lista la política, target, nivel de soporte y los 7 slots con sus overrides y avisos.

### AC-082-02: Mutaciones de Model Routing gobernadas
Las operaciones `set`, `unset` y `reset` desde el launcher ejecutan preview y requieren confirmación con token antes de persistir cambios en disco.

### AC-082-03: Pantalla de Actualización operativa
El launcher incluye una vista de actualización que muestra las versiones instalada, descargada y publicada, la relación tipada y permite `check`, `plan`, `apply`, `recover` y `cancel`.

### AC-082-04: Gates de seguridad respetados
Todas las operaciones en ambas pantallas honran sesión local, same-origin y single-use grants.

### AC-082-05: Cobertura de pruebas hermética
La suite de pruebas del launcher (`app-launcher.test.ts`, `app-self-update.test.ts`) verifica las nuevas rutas y paneles sin alterar código ajeno.
