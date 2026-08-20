# 03 Criterios de Aceptación

## Criterios de aceptación

1. Un test demuestra que `axiom-increment create --id <nuevo>` crea un artefacto desde `draft` cuando el singleton previo está `archived` para otro ID.
2. Pruebas de `init` demuestran que los ocho targets canónicos son aceptados y que `copilot-vscode` se rechaza tanto por CLI como por la función pública.
3. Pruebas de `configure` demuestran que un `init.json` legacy se normaliza/persiste como `github-copilot` antes de dispatch y que el resultado no contiene el alias.
4. Pruebas de registry/documentación fijan que LiteLLM y `copilot-vscode` no aparecen como target, import, dispatcher, ejemplo ni archivo activo, excepto el normalizador interno de persistencia y archivos históricos identificados.
5. `docs/installation.md` y `docs/generated-files.md` enumeran exactamente los ocho targets y no prometen archivos retirados.
6. ACC-044 clasifica la divergencia real separando contrato, evidencia, historia y resolución; los README archivados requeridos se describen como archivados y la reparación de coherencia incluye requisitos, interacción/documentación y evidencia concretas.
7. `DEC-20260818-134600-3jfjak` queda enlazada al correctivo mediante Core y no reclama supersesión formal; si Core no ofrece aceptación/cierre de Decision, la prosa declara ese límite.
8. Specs 01/04/05/07 y `context/**` no contradicen el código final en aliases, Cavekit ni R-10.

### Happy path

Un proyecto nuevo usa `github-copilot`; un proyecto con el estado persistido histórico se actualiza a ese ID canónico al ejecutar `configure`, sin que la CLI acepte el alias como entrada.

### Validaciones y errores

Los valores públicos retirados se rechazan con un error de opción que lista los targets válidos. El normalizador legacy no se ejecuta para argumentos nuevos ni abre dispatches alternativos.

### Permisos y visibilidad

No se añaden permisos, integraciones externas ni UI. La relación Decision–increment se crea exclusivamente con `axiom axiom-decision link-increment`.

### Estados y efectos observables

El correctivo sólo puede archivarse después de `verify`, receipts aplicables, validación obligatoria y re-review sin blockers. R-10 permanece pending hasta ese momento.
