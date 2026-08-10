# E2E 2026-07-27 — Adopción KVP25 desde cero (install → configure → join → ADO → MCP)

> **Documento histórico:** esta ejecución se realizó antes de ACC-030 y conserva
> la evidencia de ese estado del producto. Las referencias a `mcp-manifest`
> separado, `initialize` y `sdd-mcp-server`/`spec-mcp-broker` describen solo la
> captura del 2026-07-27; no son el contrato vigente. Desde ACC-030, Axiom usa
> únicamente `axiom-mcp-broker` (`--kind axiom`) con MCP `2026-07-28`,
> `server/discover` y sin `initialize`/`ping`.

Ejecución end-to-end real en sandbox aislado (`C:\repos\_axiom-e2e\`), sin tocar los repos
reales de KVP25 (adopción read-only) ni el working tree de Axiom. CLI usado:
`node apps/cli/dist/index.js`. ADO real: org **KVPHiberus** / proyecto **KVP25**, PAT vía
`ADO_PAT_KVP25`.

## Resultado por criterio

| # | Criterio | Resultado | Evidencia |
|---|----------|-----------|-----------|
| 1 | Instalar Axiom en un KVP25 nuevo | ✅ | `workspace setup` adopción → `kvp25-e2e.axiom` creado (axiom.yaml v2, AGENTS.md) |
| 2 | Configurar roles / adapters / MCPs / providers | ✅ | Captura histórica: `axiom.config/` con topology, mcp-manifest (sdd/spec/axiom), toolchain-catalog, skills-index (backend/frontend/qa/sdd/spec), profiles, providers=cmm |
| 3 | Autoskills en repos de código | ✅ | roles/{backend,frontend,qa} con `.axiom/agents/`, `.claude/{agents,commands}`, `.mcp.json`, `AGENTS.md` |
| 4 | Migración de spec | ✅ | 271 artefactos migrados (`specCreatedCount:271`); `index validate: OK, 271 metadata.yml`, 0 inválidos (201 increments + 70 bugs) |
| 5 | Migración de contexto técnico | ⚠️ | 0 items (la fuente legacy no exponía contexto técnico ingestable; requiere `--ingest-context <ARCHITECTURE.md/docs/adr>`) |
| 6 | Configs propias NO viajan / necesarias SÍ | ✅ | copia "descarga de repo" excluyendo `.gitignore`: ausentes `.axiom-state/`, `.axiom/`; presentes axiom.yaml, axiom.config/*, skills-index, increments, AGENTS.md |
| 7 | Unión al proyecto como otra persona | ✅ | `member install --member alice`: registrado, repos bindeados, regenerados locales `.axiom-state/local/topology-bindings.yaml` + `.mcp.json` |
| 8 | ADO configurado y validado por doctor | ✅ | `doctor --deep`: 56 checks, 50 pass, 0 fail; `ado-connectivity: pass` (PAT/org/proyecto válidos contra KVP25 real) |
| 9 | MCPs disponibles y responden | ✅ | Captura histórica: doctor deep verificó liveness legacy mediante `initialize` en sdd-mcp-server + spec-mcp-broker; el broker `axiom` expuso 15 tools. Ese protocolo fue sustituido por ACC-030 y no representa la validación vigente. |
| 10 | Crear incremento con plugin ADO | ✅ (preview) | peripherals reales: 18 sprints (Iteration 16 activa), 80 features, 31 tags; create-work-item `configured:true`, gated (`executed:false` sin `confirmed:true`). Create real NO ejecutado para no contaminar el board KVP25 |
| 11 | Planificar / implementar cada rol | ✅ (gate) | `axiom-role start` desde el repo spec → bloqueado por repo-affinity ("se opera desde su repo de código asignado"); gate de plan-aprobado cubierto por la suite (axiom-role-worktree/worktree-execution) |

## Hallazgos

- **BUG-20260727-adopt-telemetry-sinks-warn** (pending): WARN ruidoso en cada comando por
  `axiom.config/telemetry-sinks.yaml` ausente tras adopción. No fatal.
- **Contexto técnico**: la migración sólo trae contexto si se pasa `--ingest-context`; conviene
  documentarlo en el manual de adopción (no es bug, es uso).
- **Adapter outputs viajan** (`.mcp.json`, `.claude/`, `.opencode/`): son regenerables
  (`member install` recrea `.mcp.json`); valorar añadirlos a `.gitignore` (menor).
- **Create real de work item**: retenido en preview a propósito (acción externa sobre board de
  producción KVP25). Para un WI real, enviar `confirmed:true` desde el launcher.

## Conclusión

El ciclo completo de adopción multi-repo, migración de spec, separación config-personal /
config-de-equipo, onboarding de un segundo miembro, plugin ADO (conectividad + peripherals reales
+ create gated) y disponibilidad/respuesta de los MCPs quedó validado end-to-end contra ADO real.
Los dos únicos gaps son de uso (contexto técnico) y de ruido (telemetry-sinks WARN, ya fichado).
