# 03 Criterios de Aceptación

## Happy path

- **CA-A1 / ACC-050**: un home vacío y uno con `projects.yml` válido funcionan sin consultar `registry.json`; el barrido activo no encuentra contratos v1.
- **CA-A2 / ACC-051**: `use` actualiza recencia, `list` ordena por fecha/id y `--print-path` devuelve solo el path correcto con espacios/comillas/caracteres especiales.
- **CA-A3 / ACC-052**: fixtures válidas llegan idénticas al dominio tras validación estricta.
- **CA-A4 / ACC-053**: la reejecución de la misma identidad/path es no-op legítimo.
- **CA-A5 / ACC-054**: altas/upserts desde procesos distintos conservan todas las entradas y no dejan locks/temporales.
- **CA-A6 / ACC-055**: repos existentes, ausentes, fichero, symlink y no-Axiom se proyectan sin confundir disponibilidad/resolución.
- **CA-A7 / ACC-056**: los cuatro subcomandos emiten el envelope v1 en éxito.

## Validaciones y errores

- YAML corrupto, nested shape inválido, schema futuro, unknown key, timestamp inválido, mapa vacío y key/ID mismatch devuelven error tipado sin throw ni overwrite.
- Slugs colisionantes, paths canónicamente equivalentes entre proyectos y merge sin evidencia fallan sin mutación.
- Lock timeout/fallo write/rename preserva el archivo previo y produce exit no cero.
- Home no inicializable, duplicate-id y not-found con `--json` producen un solo JSON válido en stdout y diagnóstico humano solo en stderr.

## Permisos y visibilidad

El catálogo continúa siendo user-level local. Ningún error expone contenido YAML completo, secretos o paths distintos de los necesarios para remediar el conflicto.

## Evidencia mínima

Tests unitarios y E2E Commander, test multiproceso, build del monorepo/paquetes afectados y `git diff --check`. Cero tests de rechazo observan mutaciones.
