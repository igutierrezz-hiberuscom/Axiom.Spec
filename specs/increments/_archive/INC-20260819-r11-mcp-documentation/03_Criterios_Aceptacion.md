# 03 Criterios de Aceptación

## Criterios de aceptación

### Happy path

1. Las superficies activas afectadas describen exclusivamente `axiom-mcp-broker` como broker gestionado y `--kind axiom` como forma de arranque.
2. El flujo `mcp-manifest.yaml`/integraciones → `@axiom/mcp-server` → `@axiom/mcp-tools` se explica sin atribuir el despacho al router genérico de providers.
3. Engram queda caracterizado sin ambigüedad como tercero/backend opcional de memoria.

### Validaciones y errores

4. Las pruebas focalizadas MCP existentes pasan y el build raíz completa sin errores.
5. Un barrido dirigido no deja claims activos que anuncien los brokers/kinds históricos como procesos actuales.

### Permisos y visibilidad

6. La documentación preserva el carácter project-scoped del broker; no recomienda configuración MCP global no verificada.

### Estados y efectos observables

7. No se modifica el registro de tools ni la salida observada de `axiom mcp serve`; los cambios son de documentación, comentarios y contexto.
8. El plan R-11 y las specs/contextos propietarios registran el estado estable sin reescribir historia archivada.
