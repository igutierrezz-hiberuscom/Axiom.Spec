# Context

## Propósito

Esta carpeta puede contener contexto acotado y verificable para implementar/revisar ACC-080. La spec canónica permanece en el directorio padre; metadata, status, enlaces, receipts e índices siguen gobernados por Axiom Core.

La redacción no aporta evidencia de ejecución. Cualquier material posterior debe distinguir observación, diseño y resultado reproducible, e indicar criterio, origen y commit/ref de fixture o candidato.

## Contratos que el contexto no puede reabrir

- Secuencia: 1 identity → 2 state/CLI → 3 updater/helper → 4 Launcher → 5 certification.
- La autoridad única de versión es creada por 1/5; 5/5 no migra consumidores ni crea otra identidad.
- Solo tags anotados `refs/tags/v<SemVer>` certifican; lightweight se rechazan.
- `engines.node` es `>=20.14.0 <23` y `engines.npm` es `>=10.7.0 <11`.
- Las cuatro celdas obligatorias son Windows/Linux × {Node 20.14.0/npm 10.7.0, Node 22.14.0/npm 10.9.2}.
- Una celda fallida exige STOP y cambio explícito de spec.
- La documentación se reconcilia solo después de GO técnico.

Las notas pueden registrar evidencia sobre estos contratos, no proponer valores alternativos ni degradarlos silenciosamente.

## Material autoral permitido

- Matriz de handoff 1→5 con owner, ruta, interfaz y blocker.
- Inventario read-only que diferencie comandos existentes de scripts/tests/workflow nuevos que 5/5 debe entregar.
- Evidencia de la baseline completa inicial, clasificación de fallos y vínculo a su resolución separada.
- Topología de fixtures Git locales con tags A/B anotados y caso lightweight negativo.
- Matriz de las cuatro celdas Node/npm con versiones efectivas.
- Mapa de fault points y expectativas de rollback/recovery.
- Ledger legible que referencie evidencia gestionada sin copiarla masivamente.
- Notas de review independiente y checklist de integración posterior a GO.

La ausencia inicial de validator, clean-clone harness, E2E o workflow no es un blocker de entrada: son entregables de 5/5. Su ausencia al pedir GO técnico sí lo es.

## Baseline roja

Si la baseline existente está roja, el contexto conserva:

1. comando y fallo original;
2. clasificación como predecesor o bug ajeno;
3. owner y cambio separado del diff self-update;
4. rerun completo verde.

No se documentan skips, exclusiones, cambios de expectativas o reintentos sin causa como resolución. La nota de clasificación no autoriza reparar bugs ajenos dentro de este incremento.

## Documentación e integración

Después de GO técnico, el ledger puede registrar la reconciliación de `README.md`, `docs/README.md`, `docs/installation.md`, `docs/overview.md`, `docs/cli/README.md`, `docs/cli/self-update.md`, claims CLI relacionados y `specs/manuales/03_Actualizar_Versiones.md`. También puede marcar owners de `specs/00..08` y `context/` como actualizados o revisados sin cambio.

Antes de GO técnico no se registran esos claims como capacidades demostradas. Los outputs de fixture no se presentan como releases upstream.

## Material prohibido

- Copias o ediciones de metadata, plan metadata, status, receipts, enlaces o índices.
- Clones, `node_modules`, builds, caches, homes, prefixes, shims o backups E2E.
- Logs masivos, secretos, tokens, credenciales, homes reales o variables no redactadas.
- Una segunda spec o implementación de identity, manifest, updater/helper o Launcher.
- Migraciones de consumidores de versión bajo responsabilidad de 1/5.
- Alternativas lightweight o rangos/celdas Node/npm distintos sin cambio previo de spec.
- Prosa canónica duplicada de `specs/00..08` o manuales.
- Instrucciones de publicación npm, retirada de `private: true` o restauración de TUI.
- Marcadores lifecycle o órdenes de cierre/archivo fuera de Core.

Los resultados ejecutables permanecen en temp roots o en el sistema de evidencia aprobado. Esta carpeta solo conserva contexto autoral breve y trazable.