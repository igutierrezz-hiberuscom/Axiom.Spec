# 08 Glosario

## Aviso de ambigüedad de nombres (leer primero)

- **`Axiom.Spec/`** (mayúsculas, este repo): fuente de verdad documental del WORKSPACE de desarrollo de Axiom (specs 00-08, incrementos, bugs, decisiones).
- **`axiom.spec/`** (minúsculas): carpeta que el PRODUCTO Axiom espera/genera dentro de cualquier proyecto que lo adopte (`config/*.yaml`, `templates/`, `target-axiom-skills/`, `target-axiom-agents/`), incluido potencialmente el propio repo `Axiom/` (dogfooding) — hoy ausente en ese checkout. Ver [context/references/03-riesgos-y-brechas-conocidas.md](../context/references/03-riesgos-y-brechas-conocidas.md).
- Nunca usar ambos términos como sinónimos en la documentación.

## Términos del producto

- **Builder tooling**: tooling usado para diseñar y construir el propio Axiom.
- **Product model**: representación interna de Axiom para workflows, capabilities y políticas.
- **Generated configuration**: archivos de entorno generados a partir del modelo de Axiom.
- **Capability**: función independiente de plataforma expuesta por Axiom, con `id`, `domain` (`sdd`, `spec`, `code`, `memory`), `requiredTools`, `optionalTools`, `fallbacks`.
- **Provider**: implementación concreta de una o más capabilities, resuelta según `discoveryOrder` y perfil de discovery (`filesystem-first`, `gateway-first`, `local-only`).
- **Adapter**: capa de traducción desde capacidades del producto hacia un runtime objetivo (IDE/CLI externo), expuesta como `generate<Target>Config(args)`.
- **Profile / profile triple**: combinación de `functionalProfile` (`builder` | `product-owner`) + `operationalOverlay` (`local-only` | `standard` | `enterprise`) + `adapterTarget`, persistida en `axiom.yaml` y resuelta a `ResolvedInstallProfile`.
- **Overlay operacional**: modo de riesgo/compliance del proyecto (`local-only`, `standard`, `enterprise`), que determina discovery order, expectativa de gateway y `minimumSignals` de telemetría.
- **Target (adapter target)**: IDE o CLI externo de destino (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `litellm`, `copilot-vscode`, `antigravity`, `visual-studio-2026`).
- **Support level**: nivel de soporte de routing por target: `multi-mode` (routing per-slot completo), `single-mode` (routing global, sin per-slot), `fallback-only` (sin routing).
- **Slot**: carril operativo de routing de modelo (`increment`, `bug`, `plan`, `implementation`, `qa-e2e`, `review`, `archive`).
- **ResolvedInstallProfile**: resultado materializado de componer el profile triple contra `profiles.yaml`; input de adapters e installer.
- **Managed state**: estado versionado de `@axiom/versioning` usado para calcular migraciones y checkpoints de `axiom upgrade`.
- **Checkpoint**: snapshot pre-mutación (upgrade, uninstall) restaurable; se conservan los últimos 5.
- **GATE**: verificación explícita y numerada (p. ej. GATE 0010, 0024, 0031) que `@axiom/doctor` puede comprobar en runtime, ligada a un incremento concreto.
- **Spec repository**: fuente compartida del comportamiento y las decisiones del producto (`Axiom.Spec/` para este workspace).
- **Product runtime**: implementación ejecutable ubicada bajo `Axiom/`.
- **Result\<T, E\>**: tipo de retorno sin excepciones usado en todo el dominio (`@axiom/core`) para modelar éxito/fallo explícito.
- **Dogfooding**: construir Axiom usando la propia disciplina SDD de Axiom (modelo de tres repos de este workspace).

## Términos añadidos por el roadmap de rediseño (cerrado)

- **`schemaVersion`**: campo entero presente en todo schema persistido del producto (`axiom.yaml`, registro de usuario, `TopologyManifest`, índices de skills/contexto técnico, `mcp.yml`) que permite evolución aditiva sin romper consumidores de una versión anterior — nunca se resuelve mal una versión como otra (ver NFR-AXM-011 en [02_Requisitos_No_Funcionales.md](02_Requisitos_No_Funcionales.md)).
- **`ArtifactKind`**: tipo cerrado de artefacto en el modelo de carpeta-por-artefacto (`increment | bug | plan | adr | decision`), cada uno con su propio `metadata.yml` y (salvo `adr`/`decision`) su propio vocabulario `WorkflowState`.
- **`TopologyManifest`**: schema versionado (`Axiom/packages/topology/src/types.ts`) que declara el modo de topología de un proyecto (`single-repo` | `multi-repo`), sus repos por rol (`sddRepo`, `specRepo`, `roleCodeRepositories`) y sus asignaciones. Ver [03_Modelo_Operativo_y_Datos.md](03_Modelo_Operativo_y_Datos.md).
- **`RepoRoleKey`**: clave de rol de repo dentro de un proyecto gestionado por Axiom — `sdd` (método/factory), `spec` (conocimiento canónico) o `code` (runtime instalable). No confundir con los roles funcionales del profile triple (`functionalProfile: builder | product-owner`).
- **`allowedWriteScope`**: campo de `PlanMetadata` (`{ repo: string; paths: string[] }[]`) que declara qué paths, en qué repos, un plan tiene permitido mutar; validado en runtime por `validateWriteScope` (ver RF-AXM-019 en [01_Requisitos_Funcionales.md](01_Requisitos_Funcionales.md)).
- **`capabilityId`**: identificador de una herramienta MCP registrada en `@axiom/mcp-tools` (dominios `sdd.*`/`spec.*`), despachada a través del `routeTool` existente de `@axiom/tool-routing` — no debe confundirse con el `id` de `Capability` del modelo de capabilities/providers preexistente (`@axiom/capability-model`), aunque ambos comparten el mismo mecanismo de despacho ortogonal (ver [06_Integraciones_y_Capacidades.md](06_Integraciones_y_Capacidades.md)).
- **`WorkflowId`**: identificador de tipo de workflow (`'increment' | 'bug' | 'plan' | 'role' | 'qa-e2e'`) usado como clave del registro singleton de `workflow-state.json` — un registro por tipo de workflow, no por instancia de artefacto.
- **`externalRefs`**: mecanismo agnóstico de proveedor, disponible en todo tipo de artefacto, para enlazar un artefacto de Axiom con un elemento de un sistema externo (p. ej. Azure DevOps). No debe confundirse con el flag de UI `field.externalRef?: boolean` del plugin de Azure DevOps.
- **`AdrStatus` / `DecisionStatus`**: vocabularios de estado cerrados y no solapados para artefactos `adr` (`proposed | accepted | superseded | rejected`) y `decision` (`proposed | accepted | rejected`, sin `superseded`).
- **`contextBudget`**: parámetro (`small`|`medium`|`large`) de la herramienta MCP `spec.implementationContextRead` que controla el nivel de inlining de contenido en la respuesta, nunca la presencia de campos.