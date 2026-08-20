# 02 Cambios de Modelo

## Objetivo del documento

Registrar los límites de modelo necesarios para cerrar R-10 sin ampliar la superficie del producto.

## Entidades o estructuras afectadas

- **Workflow singleton**: una creación identificada usa el ID solicitado para desacoplarse de un record terminal de otra instancia; no reescribe ese record ni sus artefactos archivados.
- **Adapter target público**: el conjunto cerrado contiene ocho IDs canónicos. Se eliminan constantes/exportaciones que describan `copilot-vscode` como alias público.
- **Estado persistido legacy**: `init.json#profileTriple.adapterTarget` puede contener el alias histórico. Un normalizador interno de lectura lo convierte a `github-copilot` y persiste la forma canónica antes de invocar installer, generators o dispatches.
- **Decision Cavekit**: `DecisionStatus` no contiene `superseded`; el único vínculo de Core disponible es `links.incrementId`.

## Contratos o estados afectados

- `init` rechaza el alias en argumentos públicos.
- `configure` conserva compatibilidad únicamente para la lectura y migración de estado previamente escrito.
- La documentación de targets y archivos generados refleja los ocho IDs, sin LiteLLM ni `copilot-vscode`.
- Los estados `archived` de ACC-039..045 no cambian; su contenido humano se rectifica sin presentar «En especificación».

## Notas de compatibilidad

El alias no es compatible como contrato nuevo. Su única tolerancia es un paso de migración interno, acotado a `init.json` existente y observable porque deja el archivo persistido con `github-copilot`. Los consumidores posteriores reciben sólo el valor canónico.
