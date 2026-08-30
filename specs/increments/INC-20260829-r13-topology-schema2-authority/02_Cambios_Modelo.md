# 02 Cambios de Modelo

## Manifest único

`TopologyManifestV2` contiene: `schemaVersion: 2`, `mode`, `axiomRepo`, `codeRepos`, `legacyRepos`, `roles`, `assignments` y `qaLane`. No expone campos espejo.

`RepoRef` usa discriminantes obligatorios. Para `legacy`, `legacyFunction` es obligatorio y cerrado a `sdd|spec`; `mode` es siempre `read-only-source`. Refs locales se comparan canónicamente; URIs se validan sintácticamente y no se confunden con paths materializados.

## Invariantes

- Un axiom repo exacto.
- IDs globales únicos y refs físicas no colisionantes.
- 0..N code repos, cada uno con ownership y una primary exacta.
- 0..2 legacy repos, funciones no repetidas.
- Roles únicos y referenciados; assignments solo a code repos.
- `mode` coherente con cantidad/distribución.
- Shape cerrada y QA lane válida.

## Autoridad y proyección

El locator parte del `axiom.yaml` local, resuelve `axiomRepo` y carga allí el manifest. Una proyección opcional incluye `generatedFrom`, hash/version y marker de derivación; no puede ser mutada ni usada si no coincide con la autoridad.

## Dogfood

Autoridad: `Axiom.Spec`. Code repo: `Axiom` (`builder`, primary). Legacy SDD read-only: `Axiom.SDD`. Esto no cambia la separación de repos ni autoriza escrituras estructurales en la fuente legacy.

## Compatibilidad

No hay migrador. Schema 1 falla de forma visible.
