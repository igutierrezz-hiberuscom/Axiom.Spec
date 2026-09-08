# 04 Interacciones UI

## Superficies

CLI workspace setup/adopt/repo/role y endpoints launcher que las delegan.

## Flujo

Preview expone plan y conflictos sin escribir. Apply vuelve a validar, toma locks, prepara journal, comitea estructura y después ejecuta derivados. Recovery requerida bloquea nuevas mutaciones y ofrece diagnóstico accionable.

## Estados visibles

`created`, `updated`, `unchanged`, `skipped`, warnings derivados, rejected, failed y recovery-required. Un booleano `confirmed` no sustituye preflight ni autorización.

## Reactividad

Launcher reutiliza el mismo runner/envelope; no interpreta warnings estructurales como éxito. I reutilizará el catálogo y outcomes para comandos granulares.
