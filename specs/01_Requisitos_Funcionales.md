# 01 Requisitos Funcionales

## Meta-requisitos del workspace de desarrollo (dogfooding)

### RF-AXM-001 Separación por ownership

Axiom debe distinguir claramente el repo de runtime (`Axiom/`), el repo de spec (`Axiom.Spec/`) y el repo SDD (`Axiom.SDD/`). Vigente hoy: `Axiom.SDD/AGENTS.md` fija esta separación como regla operativa activa para construir el propio producto.

### RF-AXM-002 Workflow SDD sobre Axiom

Axiom debe permitir que la especificación, planificación, implementación y validación del propio producto se ejecuten con sus propias reglas operativas (bootstrap lifecycle de `Axiom.SDD/AGENTS.md`: entender → localizar/crear spec → refinar → implementar → validar → revisar → cerrar → integrar conocimiento estable).

### RF-AXM-003 Modelo de topología de repos (multi-repo, dentro del producto)

El runtime debe conocer qué repos forman parte de un proyecto gestionado, cómo se llaman, qué alias usan y dónde se resuelven. Implementado hoy vía `@axiom/topology` (`topology.yaml`: `repoRefs` + `roleAssignments` + QA lanes) y expuesto por `axiom topology`/`axiom repo`.

### RF-AXM-004 Resolución multi-surface

La topología de repos y el proyecto activo deben poder resolverse tanto desde workspace local (`@axiom/filesystem-truth`, `@axiom/project-resolution`) como desde surfaces externas soportadas por adapters. No existe hoy una capa MCP genérica de resolución de proyecto (el roadmap futuro la introduce en `INC-13`/`INC-14`); los MCP actuales se gestionan como catálogo de herramientas vía `@axiom/toolchain` y comandos `axiom mcp`.

### RF-AXM-005 Operación con un único rol

La baseline actual debe ser usable por un único rol (`functionalProfile: builder`) sin impedir que el modelo evolucione a roles múltiples más adelante. Confirmado: `Axiom/docs/first-project-readiness.md` recomienda `builder` como perfil más cubierto en runtime hoy; `product-owner` existe como segundo profile declarado pero con menor cobertura runtime.

## Requisitos funcionales del producto (ciclo de vida real, verificado en código y docs)

### RF-AXM-006 Inicialización de proyecto (`axiom init`)

El CLI debe poder inicializar un proyecto validando el nombre (`^[a-z0-9][a-z0-9-]{0,62}$`), determinando el layout (`self-hosted` vs `installed-multi-repo`), generando `axiom.yaml` con el profile triple (`functionalProfile` + `operationalOverlay` + `adapterTarget`), creando `.sdd/local/` y `.sdd/<projectName>/`, y persistiendo `init.json`. Fuente: `Axiom/docs/cli/init.md`.

### RF-AXM-007 Registro de miembros (`axiom join`)

El CLI debe registrar miembros del proyecto por id (`user:alice`, `agent:sdd`, etc.) en `.sdd/<projectName>/members.yaml`, deduplicando por igualdad exacta. Fuente: `Axiom/docs/cli/join.md`.

### RF-AXM-008 Composición del install profile (`axiom configure`)

El CLI debe leer el profile triple, componer el `ResolvedInstallProfile` real (`@axiom/install-profiles` + `@axiom/installer`) y persistir `install-profile.json`, materializando además surfaces derivadas según el target activo (p. ej. `.github/copilot-instructions.md` vía `@axiom/document-bootstrap`). Fuente: `Axiom/docs/cli/configure.md`.

### RF-AXM-009 Sincronización de adapters (`axiom sync`)

El CLI debe reconciliar los outputs del adapter activo contra el filesystem, validando primero el gate de telemetría del overlay activo; si el gate falla, debe abortar antes de escribir cualquier marker. Fuente: `Axiom/docs/cli/sync.md`.

### RF-AXM-010 Arranque de runtime (`axiom start`)

El CLI debe resolver el modo de discovery (`filesystem` | `gateway`) según el overlay y las flags (`--gateway`/`--no-gateway`), ejecutar un primer ruteo sintético de capability/provider y persistir `last-start.json`. Fuente: `Axiom/docs/cli/start.md`.

### RF-AXM-011 Auditoría de integridad (`axiom audit`)

El CLI debe verificar en modo solo lectura el audit trail del proyecto (hash SHA-256, conteo de líneas, retención, detección de reescritura externa) y reportar uno de tres estados: `compliant`, `absent` o `violation`. Fuente: `Axiom/docs/cli/audit.md`.

### RF-AXM-012 Diagnóstico de salud (`axiom doctor`)

El CLI debe ejecutar checks de boundaries, policies, manifests, aislamiento, modelo de capacidades y gateway, devolviendo un resultado agregable (PASS/FAIL) y soportando salida `--json`. Fuente: `Axiom/docs/cli/doctor.md`, package `@axiom/doctor`.

### RF-AXM-013 Versionado y upgrade (`axiom upgrade`)

El CLI debe calcular migraciones aplicables entre versiones, crear un checkpoint pre-upgrade, aplicar migraciones en orden y hacer rollback automático si alguna falla, con soporte `--dry-run` y `--from-checkpoint`. Fuente: `Axiom/docs/cli/upgrade.md`, package `@axiom/versioning`.

### RF-AXM-014 Model routing por slot (`axiom model`)

El CLI debe permitir ver, fijar, quitar y resetear el modelo asignado a cada slot operativo (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`), con fallback declarado por `SupportLevel` del target (`multi-mode`, `single-mode`, `fallback-only`) y proyección opcional a `.opencode/model-routing.json`. Fuente: `Axiom/docs/cli/model.md`, package `@axiom/model-routing`.

### RF-AXM-015 Gestión de components y skills

El CLI debe exponer catálogo, instalación/desinstalación con checkpoint (`axiom components`) y registro/drift de skills materializadas contra `.opencode/skills-lock.yaml` (`axiom skills`). Fuente: `Axiom/docs/cli/components.md`, `skills.md`.

### RF-AXM-016 Generación de surfaces por adapter target

El runtime debe poder generar configuración específica para al menos 6 targets con adapter dedicado (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `litellm`) mediante una función `generate<Target>Config` de firma común (`Promise<Result<GeneratorResult, AdapterGeneratorError>>`), y declarar 3 targets adicionales sin adapter dedicado que caen a `fallback-only` (`copilot-vscode`, `antigravity`, `visual-studio-2026`).

### RF-AXM-017 Superficie de comandos operativos más amplia que la documentada

El código de `apps/cli/src/commands/` contiene 36 ficheros de comando (incluyendo `app`, `app-plugins`, `app-plugins-azure-devops`, `capability`, `context`, `gateway`, `intent`, `mcp`, `memory`, `projects`, `qa-archive-gate`, `repo`, `roles`, `self-update`, `toolchain`, `topology`, `axiom-bug`, `axiom-increment`, `axiom-plan`, `axiom-qa-e2e`, `axiom-role`), de los cuales `Axiom/docs/cli/` solo documenta 9 en profundidad (`init`, `join`, `configure`, `sync`, `start`, `audit`, `upgrade`, `doctor`, `tui`) más `model`, `components` y `skills`. Esta spec no debe asumir que los comandos no documentados están estabilizados solo por existir en código; requieren verificación puntual antes de citarlos como contrato estable.