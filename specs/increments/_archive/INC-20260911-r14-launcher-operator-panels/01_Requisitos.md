# 01 Requisitos

## Objetivo del documento

Definir los requisitos técnicos y de interfaz para incorporar al launcher web local (`axiom app`) las dos pantallas de operador que faltaban: Model Routing y Actualización de Axiom (`ACC-082`).

## Requisitos del incremento

- **REQ-082-01: Pantalla de Model Routing en el Launcher**
  Añadir vista y panel de Model Routing que muestre:
  - Política base y task classes configuradas.
  - Slots SDD, override efectivo por slot, y nivel de soporte real del target activo.
  - Acciones gobernadas: `set` (asignar override a slot), `unset` (remover override), `reset` (limpiar todos los overrides), y `validate` (ejecutar diagnósticos y proyección).
  - Mutaciones siguiendo el patrón preview $\rightarrow$ confirmación tokenizada.
  - Presentación honesta del soporte por target (sin prometer enrutado en destinos `fallback-only`).

- **REQ-082-02: Pantalla de Actualización en el Launcher**
  Añadir vista y panel de Actualización conectada a los endpoints de self-update:
  - Visualización de versión instalada, publicada, descargada, relación tipada (`updated`, `update-available`, etc.), frescura y procedencia.
  - Acciones gobernadas: `check`, `plan`, `apply`, `recover` y `cancel`.
  - Indicador de progreso acotado para operaciones en curso.
  - Selección de política de reinicio (`close-only` o `close-and-restart`).

- **REQ-082-03: Endpoints de Model Routing en `app-api.ts`**
  Exponer `/api/projects/:id/model` (`GET`), `/model/set` (`POST`), `/model/unset` (`POST`), `/model/reset` (`POST`) y `/model/validate` (`POST`), gobernados por sesión local, same-origin y grants de confirmación.

- **REQ-082-04: Reutilización de transportes y gates vigentes**
  Ambas pantallas consumen el bridge y transportes existentes en `transport.js` / `panels.js` sin crear capas paralelas ni duplicar lógica de negocio, SemVer o Git.

## Reglas de negocio relevantes

- Los mandos respetan estrictamente los gates del control plane (`ACC-070`..`ACC-076`).
- No se muta el estado sin confirmación explícita (tokens de un solo uso con TTL).
