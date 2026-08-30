# Telemetría y regresión final del launcher R-13

> **Código**: INC-20260829-r13-launcher-telemetry-regression
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-075 y cierre agregado de ACC-076
> **Dependencias**: F y G

## Objetivo

Exponer el audit trail solo mediante una API tail sancionada/acotada, presentar eventos recientes y métricas con scopes honestos y ejecutar la matriz integral de regresión ACC-070..ACC-075.

## Revalidación y parcial existente

El cambio local preexistente de R-12 ya retira `lessons` del panel y debe preservarse. Aun así, `app-launcher-telemetry.ts` importa `fs/auditLogPath`, lee el log completo, toma los primeros 20 de la ventana final y etiqueta counters singleton bajo un proyecto. ACC-075 sigue pendiente salvo esa retirada parcial.

## Alcance

- ACC-075: API pública en `@axiom/telemetry` para tail bounded/validated; CLI no lee audit.log directo; eventos realmente recientes con orden documentado y aislamiento por root de proyecto; métricas project-scoped separadas de process-wide.
- ACC-076: matriz final conjunta de sesión/origin/body/SSRF/headers/SSE/tokens/races/required/delivery/IDs/links/telemetría mediante servidor y wrappers efectivos.
- Preservar ausencia de lessons.

## No objetivos

No reintroducir learn/lessons, no crear backend de métricas por proyecto si no existe, no llamadas externas, no R-13.5.

## Decisiones cerradas

1. `@axiom/telemetry` expone `readAuditTrailTail({projectRoot,maxEvents,maxBytes}) -> Result<AuditTail, AuditReadError>`; solo esa API y `auditTrailVerify` pueden leer el log.
2. Implementación lee desde el final por chunks y nunca supera 1 MiB ni 500 eventos por defecto/máximo de panel; descarta la primera línea parcial, valida cada evento y reporta parse errors sin leer el archivo completo.
3. El scope de proyecto se garantiza por `projectRoot` resuelto y confinado; jamás se combina log de otro root. Dos proyectos producen snapshots independientes.
4. `recentEvents` contiene los 20 eventos válidos más recientes en orden `newest-first`. Agregados usan la misma ventana tail de hasta 500 eventos.
5. Audit verify y tail son llamadas separadas a APIs sancionadas, pero la CLI no usa `fs` ni conoce path físico.
6. Contadores de bus se devuelven bajo `processMetrics: {scope:'process-wide', ...}`. No aparecen como project metrics. `projectMetrics` contiene solo datos derivados del audit root del proyecto.
7. Corrupción devuelve parseErrors/diagnóstico tipado y eventos válidos restantes según contrato; I/O falla visiblemente sin filtrar contenido.
8. `lessons` y endpoints/copy asociados permanecen ausentes.
9. ACC-076 solo se considera satisfecha si la suite agregada cubre todos los escenarios de F/G/H y termina con conteos exactos; timeout no cuenta como PASS.

## Riesgos

Lectura reverse de líneas UTF-8/CRLF, archivos rotados/cambiantes, interferencia de counters singleton en tests y gran matriz. Se usan fixtures por proyecto, reset de bus y lotes si es necesario.

## Compatibilidad

El campo ambiguo `busCounters` puede retirarse o reemplazarse por el bloque scoped versionado; consumidores se actualizan juntos. No fallback a lectura directa.

## Validación prevista

Tests unitarios tail grande/corrupto/orden/límites/dos proyectos; endpoint/panel; búsqueda de acceso directo; matriz completa launcher real; build, full focused suites y diff-check.

## Integración estable

Diferida al cierre final junto con F/G; no editar specs 00..08/context durante apply.
