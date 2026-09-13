# 03 Criterios de Aceptación

## Criterios de aceptación

### AC-091-01: Sin enlaces muertos
Ningún enlace relativo de `Axiom/README.md` ni de `Axiom/docs/**` apunta a una ruta inexistente. En particular no queda ninguna referencia a `openspec/` en documentación activa, ni en enlace ni en prosa, en `README.md`, `docs/README.md`, `docs/cli/components.md` y `packages/installer/README.md`.

### AC-091-02: Separación de histórico y vigente
`Axiom/README.md` presenta el contenido histórico bajo un encabezado explícito y ninguna afirmación superada aparece fuera de esa zona. Las tablas de paquetes con métricas del cierre del MVP quedan dentro de ella.

### AC-091-03: Afirmaciones alineadas con el runtime
El README no afirma que los adapters versionen `dist/`, y su descripción del comportamiento de `TC-009` coincide con la vigente tras `ACC-084`. Cualquier recuento de comandos coincide con las familias registradas en `apps/cli/src/index.ts`.

### AC-092-01: Correspondencia manual-fichero
Para cada archivo de `docs/configuration/files/` existe el YAML correspondiente en `Axiom/axiom.config/`, o el manual está marcado como contrato histórico no materializado y el índice de la sección lo separa de los vigentes.

### AC-092-02: Cobertura de los ficheros reales
Existe manual para `agents-catalog.yaml`, `skills-catalog.yaml`, `toolchain-catalog.yaml`, `mcp-manifest.yaml` y `workflows.yaml`, con el mismo esquema de la sección y con indicación de quién valida cada fichero.

### AC-093-01: Índice completo y honesto
`docs/README.md` enlaza todo el contenido existente e identifica explícitamente lo histórico, incluido `docs/cli/tui.md`.

### AC-093-02: Sin vocabulario retirado en nombres activos
No existe ningún archivo de documentación activo cuyo nombre contenga vocabulario retirado en R-04. El renombrado de `profiles-overlays-targets.md` no deja referencias entrantes rotas.

### AC-093-03: Índice estructural del repositorio canónico
`Axiom.Spec/specs/README.md` lista las carpetas realmente presentes bajo `specs/`, incluidas `decisions/`, `adr/`, `plans/` y `archive/`.

### AC-D01-01: Decisión de estructura registrada
Existe un artefacto de decisión creado con `axiom axiom-decision create` que fija la organización del manual, el esquema mínimo de página, la convención de nombres, el tratamiento del histórico y la definición operativa de «familia documentada». El artefacto está enlazado a este incremento y su estado lo gobierna Core.

### AC-GEN-01: Integridad general
`npm run build`, `npm run doctor`, `npm run readiness:first-project` y `git diff --check` pasan limpiamente, y el barrido de vocabulario retirado no devuelve coincidencias fuera de secciones marcadas como históricas.
