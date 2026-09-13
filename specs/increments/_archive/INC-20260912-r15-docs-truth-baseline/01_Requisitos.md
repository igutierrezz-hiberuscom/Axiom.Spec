# 01 Requisitos

## Objetivo del documento

Fijar qué debe cumplir cada corrección de veracidad documental de este incremento, sin describir todavía cómo se implementa.

## Requisitos del incremento

### RQ-091 README del runtime veraz

1. El `README.md` de `Axiom/` distingue de forma inequívoca, por encabezado, qué contenido es foto histórica y qué describe el estado vigente.
2. No contiene enlaces a rutas inexistentes. En particular desaparece el enlace a `openspec/changes/archive/...` y las tres menciones en prosa de `docs/README.md`, `docs/cli/components.md` y `packages/installer/README.md`.
3. Las afirmaciones sobre adapters reflejan el estado posterior a `ACC-084`: los artefactos compilados ya no se versionan y `TC-009` avisa cuando falta build.
4. El inventario de comandos no promete un número inferior al real ni presenta como completa una lista parcial. Si se conserva un recuento, coincide con las familias registradas en `apps/cli/src/index.ts`.
5. Las tablas de paquetes con métricas de cierre del MVP quedan dentro de la sección histórica, no en presente.

### RQ-092 Manuales de configuración coherentes con los ficheros reales

1. Para cada manual de `docs/configuration/files/` existe el fichero correspondiente en `Axiom/axiom.config/` o el manual está marcado como contrato histórico no materializado, con esa marca visible también en el índice de la sección.
2. Los cuatro casos conocidos (`onboarding.yaml`, `scaffolding-contract.yaml`, `command-protocol.yaml`, `local-overlay-policy.yaml`) reciben el mismo tratamiento, salvo justificación explícita por archivo.
3. Los cinco ficheros vigentes sin manual (`agents-catalog.yaml`, `skills-catalog.yaml`, `toolchain-catalog.yaml`, `mcp-manifest.yaml`, `workflows.yaml`) quedan documentados con el mismo esquema que el resto de la sección: para qué sirve, qué manda de verdad, cuándo tocarlo y qué superficies afecta.
4. El manual indica, para cada fichero, si su contenido lo valida `config-validation` o su propio cargador.

### RQ-093 Índices que reflejan la realidad

1. `docs/README.md` lista todo el contenido existente e identifica explícitamente lo histórico, incluido `docs/cli/tui.md`.
2. Ningún nombre de archivo activo conserva vocabulario retirado: `docs/configuration/profiles-overlays-targets.md` se renombra y todas sus referencias entrantes se actualizan.
3. `Axiom.Spec/specs/README.md` lista las carpetas que existen bajo `specs/`, incluidas `decisions/`, `adr/`, `plans/` y `archive/`.

### RQ-D01 Estructura del manual único declarada

1. Existe un artefacto de decisión, creado con `axiom axiom-decision create`, que fija: la organización del manual, qué constituye una página de familia de comandos, el esquema mínimo de cada página, la convención de nombres, dónde vive el contenido histórico y cómo se marca.
2. La decisión declara qué se considera «familia documentada» a efectos de medir cobertura, porque el incremento 2 y el gate del incremento 5 dependen de esa definición.
3. La decisión es consumible por un agente: la página debe permitir entender qué hace un comando, cuándo usarlo, qué escribe y con qué se conecta, sin leer el código.

## Reglas de negocio relevantes

- La documentación histórica se conserva marcada, no se borra, salvo que el usuario lo pida explícitamente.
- La estructura decidida en `D-01` es vinculante para los incrementos 2, 5 y 6 del lote.
- Ninguna corrección puede introducir una afirmación no verificada: si un dato no se puede comprobar en código, configuración o ejecución, no se escribe.

## Fuera de alcance funcional

- Cobertura de comandos no documentados.
- Cualquier cambio en `Axiom.Spec/specs/manuales/**` más allá del índice estructural `specs/README.md`.
- El gate de verificación documental y la distribución del manual a proyectos.
