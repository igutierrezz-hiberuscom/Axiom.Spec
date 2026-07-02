# AGENTS.md

## Repository role

{role} — {description}

## Axiom workspace topology

Product: {product.id}
Spec root: {paths.specRoot}
SDD root: {paths.sddRoot}

## Canonical context order

1. Spec aprobada
2. Plan aprobado
3. Decisiones explícitas (ADR)
4. Código real
5. Manifests canónicos (`repo.manifest`, `product.manifest`)

## Manifests and discovery

- repo.manifest: `.sdd/repo.manifest.yaml`
- product.manifest: `{paths.specRoot}/product.manifest.yaml`
- Discovery order: {discovery.modeOrder}

## Repo-local surfaces

| Surface | Path | Versioned |
|---------|------|-----------|
| Skills | `.axiom/skills/` | Yes |
| Rules | `.axiom/rules/` | Yes |
| Commands | `.axiom/commands/` | Yes |
| Agents | `.axiom/agents/` | Yes |
| Local overlay | `.sdd/local/` | No |

## Adapter delegation

Adapters (Copilot, Claude Code, OpenCode, Cursor) may:
- Delegate to this file
- Emit minimal derived summaries
- Add only irreducibly runtime-specific config

Adapters must NOT:
- Duplicate full SDD workflow
- Redefine repo roles or product topology
- Replace canonical manifests

<!-- AXIOM:GENERATED:START -->
## Generated portable entries

{auto-generated skill/rule/command references}
<!-- AXIOM:GENERATED:END -->

<!-- TEAM:CUSTOM:START -->
## Team custom notes

{team-maintained content preserved byte-exact}
<!-- TEAM:CUSTOM:END -->