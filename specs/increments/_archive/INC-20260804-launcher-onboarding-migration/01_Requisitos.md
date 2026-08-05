# Requisitos

Estado de implementación: cumplidos y validados en la API HTTP y el launcher
estático. La consolidación canónica y el freeze final del incremento están
completados por el orquestador.

## RQ-001. Onboarding completo

El launcher debe cubrir install, join, seleccion de proyecto, configuracion y
workspace setup con las opciones que el runtime realmente soporta.

## RQ-002. Adopcion visible

Debe existir una ruta visual para adoptar spec, SDD/control y contexto tecnico,
con preview, confirmacion y resultado basado en los comandos existentes.

## RQ-003. Seguridad de writes

Preview, dry-run y peticiones no confirmadas no escriben. Las operaciones
confirmadas respetan no-clobber, provenance, colisiones e idempotencia de los
primitives subyacentes.

## RQ-004. Aislamiento

Cada operacion debe resolver el repo destino y mantener el proyecto/registry
correctos en topologias single-repo y multi-repo.

## RQ-005. Honestidad

Las capacidades no ejecutables se muestran como pendientes o advertencias; no
se presentan selecciones de UI como aplicadas si no existe un camino real.

## Trazabilidad

- RQ-001: install/join existentes preservados; `workspace/setup` añade setup
	single-repo y multi-repo con selección de paths y registry real.
- RQ-002: `workspace/adopt` expone fuentes de spec, SDD/control y contexto y
	delega en `runWorkspaceAdopt`.
- RQ-003: todos los endpoints mutantes usan preview y confirmación explícita;
	las pruebas verifican cero writes en preview/declino.
- RQ-004: paths absolutos, fuentes legacy externas y `homeDir` del servidor se
	validan antes de delegar; el registry confirmado se lee desde ese mismo home.
- RQ-005: resultados incluyen pending, created, skipped, warnings, rutas,
	conformance y provenance; tools conserva la nota pendiente existente.
