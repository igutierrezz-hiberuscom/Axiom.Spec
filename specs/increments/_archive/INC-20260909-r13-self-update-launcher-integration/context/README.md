# Context

## Propósito

Este directorio reserva contexto de trabajo **local al incremento 4/5** para implementar y revisar la integración del self-update en Launcher. Su contenido puede explicar cómo el adaptador HTTP/UI consume los contratos de los incrementos 1–3, pero no se convierte en fuente canónica ni demuestra por sí solo que la capacidad esté implementada.

## Qué puede vivir aquí

- Mapas fechados de rutas/símbolos reales del Launcher, control plane, static frontend y runner headless.
- Matrices de trazabilidad entre ACC-077/078, requisitos, criterios y pruebas focales.
- Inventarios de fixtures herméticos: remotos Git bare, repos, homes, prefixes y procesos helper temporales.
- Notas sanitizadas de decisiones de wiring, compatibilidad o riesgos que aún no sean conocimiento estable.
- Resultados resumidos de build, typecheck, tests HTTP/control-plane/frontend/helper y review independiente.
- Evidencia de no-mutación antes/después para bootstrap, status, check, plan y rechazos de seguridad.
- Handoffs consumibles por el incremento 5, siempre referenciando la evidencia primaria.

Todo archivo adicional debe indicar fecha, origen, alcance y si describe un hecho observado, una hipótesis o una ruta probable.

## Qué no debe vivir aquí

- `metadata.yml`, `plan.metadata.yml`, índices, caches generadas, receipts o cambios manuales de status.
- Copias de planes, ADR, decisiones o especificaciones canónicas.
- Un motor Git, script de instalación, helper ejecutable o código runtime.
- Secrets, cookies, confirmation tokens, credenciales, URLs con userinfo, PII, stacks o logs sin sanitizar.
- Dumps ilimitados de stdout/stderr, `node_modules`, repositorios Git, homes/prefixes de prueba o binarios.
- Evidencia que afirme que Launcher ya ofrece self-update antes de implementación y validación.
- Release CI o documentación final propiedad del incremento 5.

## Estructura sugerida

Si la ejecución necesita contexto adicional, se prefieren pocos archivos explícitos en lugar de una jerarquía generada:

- `runtime-map.md`: hechos y rutas/símbolos observados, distinguidos de candidatos.
- `traceability.md`: requisitos → criterios → tests/evidencia.
- `validation-summary.md`: comandos, entorno, resultados y fallos preexistentes/nuevos.
- `review-notes.md`: findings de seguridad, no-duplicación y UX.
- `handoff-to-5.md`: evidencia y limitaciones entregadas a certificación.

La creación de estos archivos no es obligatoria. Este README basta mientras no exista evidencia adicional; no se deben crear artifacts estructurales ni transiciones fuera de Axiom Core.

## Reglas de provenance

1. Diferenciar **existente**, **por implementar** y **probable**.
2. Enlazar a source/test concreto y commit o candidate freeze cuando el lifecycle lo aporte.
3. Registrar resultados exactos; no convertir un test no ejecutado en PASS.
4. Sanitizar comandos y outputs antes de conservarlos.
5. Mover conocimiento estable a su spec canónica solo durante la integración prevista y sin duplicar historia de implementación.
