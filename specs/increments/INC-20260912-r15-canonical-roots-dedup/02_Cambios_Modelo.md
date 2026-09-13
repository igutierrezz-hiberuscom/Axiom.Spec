# 02 Cambios de Modelo

## Objetivo del documento

Detallar qué artefactos, carpetas y contratos cambian en el repositorio canónico.

## Artefactos gestionados afectados

- Los nueve documentos de `Axiom.Spec/decisions/` (`0015`, `0019`, `0026`..`0032`): migrados a `specs/decisions/` como decisiones gestionadas `DEC-*` mediante comandos Core, con banner de procedencia y contenido preservado.
- `specs/adr/ADR-0032-toolchain-versioning/`: implicado en la resolución del choque de numeración.
- `specs/decisions/DEC-*`: los siete artefactos preexistentes permanecen sin sobrescritura y los nueve nuevos representan el contenido legacy migrado.
- `increments/INC-20260817-r10-acc038-lifecycle-docs`: copia suelta retirada; su artefacto real permanece archivado bajo `specs/increments/_archive/`.

## Carpetas afectadas

- `Axiom.Spec/increments/`: retirada.
- `Axiom.Spec/bugs/`: retirada, con sus dos READMEs legacy.
- `Axiom.Spec/axiom.spec/`: eliminada tras comprobar que solo contenía un directorio vacío sin archivos.
- `Axiom.Spec/specs/{increments,bugs,archive}/`: sin cambios, se conservan operativas.

## Documentación afectada

- `Axiom.Spec/README.md`: estructura reconciliada.
- Referencias activas a los documentos de decisión afectados: actualizadas al destino resultante.

## Contratos o estados afectados

- Cambia el contrato documental del repositorio canónico: una sola raíz y un solo formato de decisiones.
- No cambia la resolución de artefactos del runtime: la raíz resuelta ya es `specs` y este incremento no la altera.
- No cambia ningún esquema, estado persistido del runtime ni comportamiento de comando.

## Riesgo estructural

Migrar identificadores de decisión rompe referencias si no se actualizan todas. El barrido de referencias es parte del criterio de aceptación, no un paso opcional.
