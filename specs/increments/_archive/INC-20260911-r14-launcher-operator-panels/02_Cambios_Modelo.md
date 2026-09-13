# 02 Cambios de Modelo

## Objetivo del documento

Detallar las estructuras y rutas expuestas en el servidor del launcher y los elementos de interfaz.

## Endpoints añadidos

- `GET /api/projects/:id/model`: Devuelve política base, assignments, target, supportLevel y slots recomendados.
- `POST /api/projects/:id/model/set`: Asigna override con preview/confirmación.
- `POST /api/projects/:id/model/unset`: Remueve override con preview/confirmación.
- `POST /api/projects/:id/model/reset`: Limpia overrides con preview/confirmación.
- `POST /api/projects/:id/model/validate`: Ejecuta validación y proyecciones de adapters.

## Superficies UI añadidas

- Pestañas en la barra superior (`view-switch`):
  - `tab-models` (`data-view="models"`): Model Routing
  - `tab-self-update` (`data-view="self-update"`): Actualización de Axiom
- Vistas correspondientes:
  - `<section class="view" id="view-models">`
  - `<section class="view" id="view-self-update">`
