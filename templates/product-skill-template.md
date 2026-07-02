# Plantilla de skill de producto — Axiom

> Esta plantilla es la consumida por `init` para validar skills contra el
> contrato de 18 campos definido en
> `Axiom.Spec/artifacts/changes/0005-skill-model/specs/skill-model/spec.md` y aterrizado
> en `Axiom.Spec/config/skills-catalog.yaml`. La prosa final es narrativa: el
> bloque front-matter YAML es la fuente validable. Consulta
> `Axiom.Spec/context/shared/skill-model-contract.md` para la prosa normativa y
> la tabla de mapeo a superficies runtime.

## Bloque front-matter (18 campos, fuente validable)

```yaml
---
# 1 id ^[a-z][a-z0-9-]+$
id: <skill-id>
# 2 name (string no vacio)
name: <Nombre humano>
# 3 version semver
version: <X.Y.Z[-pre.N]>
# 4 summary 1..280 chars; no reemplaza SKILL.md (Regla 5)
summary: <Resumen corto>
# 5 type tag libre; NO perfil (Regla 4)
type: <tag-de-contexto>
# 6 applicableRepoTypes: subconjunto cerrado y no vacio de {product-owner, builder}
applicableRepoTypes: [<product-owner|builder>]
# 7 compatibleHarnesses: set cerrado, NO vacio (R-V5)
compatibleHarnesses: [<copilot-vscode|opencode|claude-code|antigravity|visual-studio-2026>]
# 8 requiredCapabilities: patron ^(sdd|spec|code|memory)\.[a-z][a-zA-Z0-9]+$
requiredCapabilities: [<sdd.workflow|spec.read|code.symbolSearch|memory.decisionRecall>]
# 9 optionalTools: exigen contraparte en `fallbacks` (Regla 2)
optionalTools: [<tool-id>]
# 10 fallbacks: declaracion explicita de degradacion (Regla 2)
fallbacks: [<fallback-id>]
# 11 inputs: paths, params, contextos (solo referencia)
inputs: [<input-id>]
# 12 outputs: salidas deterministas proyectadas al cuerpo de la skill
outputs: [<output-id>]
# 13 relatedCommands: comandos Axiom relacionados (solo referencia)
relatedCommands: [<command-id>]
# 14 relatedOrchestrators: orquestadores relacionados (solo referencia)
relatedOrchestrators: [<orchestrator-id>]
# 15 riskLevel: low | medium | high
riskLevel: <low|medium|high>
# 16 lifecycleState: top-level, NO se aplana al lockfile (AD-02)
lifecycleState: <draft|candidate|active|deprecated|archived>
# 17 portableEntry: resumen portable 1..280; derivado, no autoritativo (Regla 6)
portableEntry: <Resumen portable>
# 18 registryMeta: 4 requeridos. Si lifecycleState==deprecated, replacement obligatorio (R-V9)
registryMeta:
  reviewStatus: <draft|pending|approved|local-owner|blocked>
  securityCheck: { status: <ok|warning|blocked>, summary: <...>, findings: [] }
  bundleHash: sha256:<64 hex chars>  # patron ^sha256:[a-f0-9]{64}$
  source: <product-registry|repo-local>
  # Opcionales (viven solo en el skill model, no se aplanan al lockfile):
  lastReconciledAt: <ISO-8601>
  reconciliationHash: sha256:<64 hex chars>
  # Obligatorio si lifecycleState == deprecated (R-V9):
  replacement: <id-de-otra-skill>
---
```

## Ejemplo completamente cumplimentado (caso nominal)

```yaml
---
id: axiom-context-persistence
name: Persistencia de contexto de Axiom
version: 0.1.0
summary: Persiste el contexto tecnico curado de un repo Axiom en archivos versionados y snapshots.
type: persistence
applicableRepoTypes: [product-owner, builder]
compatibleHarnesses: [opencode]
requiredCapabilities: [sdd.workflow, memory.contextRecall]
optionalTools: [memory.decisionRecall]
fallbacks: [filesystem]
inputs: [repo.spec.dir]
outputs: [.sdd/local/context/snapshot.json]
relatedCommands: [axiom sync]
relatedOrchestrators: [axiom-sdd-orchestrator]
riskLevel: low
lifecycleState: active
portableEntry: Persiste el contexto tecnico curado de un repo Axiom en archivos versionados y snapshots.
registryMeta:
  reviewStatus: approved
  securityCheck: { status: ok, summary: Sin hallazgos bloqueantes., findings: [] }
  bundleHash: sha256:1c33dbefcf0d5081983260d6df1a234582eabebcdf7dad68dc70b9328b683df5
  source: product-registry
---
```