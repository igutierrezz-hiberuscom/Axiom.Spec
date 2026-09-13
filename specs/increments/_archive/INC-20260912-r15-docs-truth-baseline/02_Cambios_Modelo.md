# 02 Cambios de Modelo

## Objetivo del documento

Detallar qué superficies documentales se corrigen, renombran, retiran o crean, y qué contratos quedan afectados.

## Archivos afectados

### Corrección de contenido

- `Axiom/README.md`: reorganización en dos zonas (vigente e histórica), retirada del enlace muerto, corrección de las afirmaciones sobre `dist/` de adapters y del inventario de comandos y paquetes.
- `Axiom/docs/README.md`: índice completo, incluida la identificación de contenido histórico.
- `Axiom/docs/cli/components.md` y `Axiom/packages/installer/README.md`: retirada de las referencias en prosa a `openspec/`.
- `Axiom/docs/configuration/files/README.md`: separación explícita entre manuales de ficheros existentes y contratos históricos no materializados.

### Renombrado

- `Axiom/docs/configuration/profiles-overlays-targets.md` pasa a un nombre alineado con el vocabulario vigente (configuración funcional única y política local). Se actualizan todas las referencias entrantes, incluidas las de `docs/README.md`, `docs/configuration/README.md` y cualquier manual que lo enlace.

### Retirada o marcado como histórico

- `Axiom/docs/configuration/files/onboarding.md`
- `Axiom/docs/configuration/files/scaffolding-contract.md`
- `Axiom/docs/configuration/files/command-protocol.md`
- `Axiom/docs/configuration/files/local-overlay-policy.md`

El tratamiento elegido debe ser homogéneo y quedar registrado con su motivo en el propio incremento.

### Documentación nueva

- Un manual por cada fichero vigente hoy sin cobertura: `agents-catalog.yaml`, `skills-catalog.yaml`, `toolchain-catalog.yaml`, `mcp-manifest.yaml`, `workflows.yaml`.

### Índice del repositorio canónico

- `Axiom.Spec/specs/README.md`: se completa el listado de artefactos bajo `specs/`.

## Artefactos gestionados creados

- Un artefacto de decisión para `D-01`, creado con `axiom axiom-decision create`. No se escribe a mano ni se le asigna estado manualmente.

## Contratos o estados afectados

- No hay cambios en contratos de runtime, esquemas, estado persistido ni comportamiento de comandos.
- Cambia el contrato documental implícito: a partir de este incremento, `Axiom/docs/**` es el manual único de producto y toda referencia debe apuntar ahí.
- El renombrado de un archivo de documentación puede romper enlaces externos al repositorio; se acepta porque la referencia vigente vive dentro del propio repositorio.

## Estructura resultante esperada

- `Axiom/docs/` mantiene su árbol actual (entrada general, configuración, comandos, uso, superficies generadas, troubleshooting), con la organización que fije `D-01` y con lo histórico identificado.
- `Axiom/axiom.config/` no cambia: este incremento documenta lo que existe, no altera configuración.
