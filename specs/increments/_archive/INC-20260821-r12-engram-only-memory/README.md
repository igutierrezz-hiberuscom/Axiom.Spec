# r12-engram-only-memory

> **Código**: INC-20260821-r12-engram-only-memory
> **Estado**: Archivado
> **Fecha de creación**: 2026-08-21
> **Tipo de cambio**: persistencia de memoria obligatoria en Engram

## Resumen

Convertir Engram en el único backend de memoria. Axiom no debe leer ni escribir memoria JSON local ni degradar silenciosamente si Engram no está instalado o no responde: cada operación devuelve un error tipado y Doctor hace visible la indisponibilidad.

## Contexto y motivación

El fallback JSON introduce dos fuentes de comportamiento y permite creer que la memoria está integrada con Engram cuando realmente queda aislada en un archivo local. Para los primeros proyectos, el contrato debe ser simple: memoria disponible mediante Engram local o error explícito; nunca una persistencia alternativa oculta.

Se verificó que el entorno actual tiene `engram 1.17.0` instalado. El subcomando MCP es un servidor stdio de larga ejecución, por lo que las pruebas deben usar stubs herméticos y no depender de la instancia local.

## Alcance

### Incluido

- Eliminar el backend JSON local, `createInMemoryBackend`, `memoryFilePath`, `forceJson` y toda selección/fallback JSON.
- Conservar únicamente las operaciones genéricas sobre `MemoryBackend` que necesite el backend Engram.
- Resolver siempre Engram para lecturas y escrituras de memoria; si su proceso, handshake o tool falla, devolver un `MemoryError` tipado y visible al caller.
- Migrar todos los callers runtime de memoria para que no construyan ni lean el backend JSON directamente.
- Añadir un check estándar de Doctor que marque como `fail` la indisponibilidad del ejecutable local de Engram, con remediación clara.
- Adaptar pruebas a stubs de `MemoryBackend`/Engram, sin crear o consultar archivos JSON como mecanismo de memoria.

### Excluido

- No se borra ni migra automáticamente ningún JSON de memoria existente; queda sin ser consumido por el runtime para evitar destrucción de datos.
- No se instala, descarga, actualiza ni configura Engram automáticamente.
- No se cambia el aislamiento project-scoped, la metadata, Knowledge Sync ni la disciplina de agentes.
- No se convierte MCP ni el toolchain en una arquitectura obligatoria general; la obligatoriedad se limita al backend de memoria solicitado.

## Documentos del incremento

- `01_Requisitos.md`: comportamiento, errores y Doctor.
- `02_Cambios_Modelo.md`: contratos y APIs retiradas.
- `03_Criterios_Aceptacion.md`: validación dirigida.
- `04_Interacciones_UI.md`: mensajes visibles y remediación.

## Dudas abiertas

Ninguna. La ausencia o fallo de Engram se trata como error de memoria, no como condición para usar almacenamiento alternativo.

## Decisiones funcionales cerradas

- Engram es el único origen de lectura y escritura de memoria.
- No existe fallback JSON ni un flag para forzarlo.
- Los errores de Engram se propagan por el contrato `Result` de memoria y se muestran por cada caller; no se convierten en listas vacías o guardados aparentes.
- Doctor falla de forma visible cuando el ejecutable Engram no está disponible localmente.
- Los JSON locales históricos no se eliminan ni se leen.

## Consolidación en la spec general

Al cierre del lote, actualizar los claims activos de memoria de `specs/00..08`, manuales y `context/**`: Engram obligatorio, sin backend JSON, sin fallback y con Doctor visible. Mantener la historia solo en artifacts archivados.

## Estrategia E2E

Con stubs de Engram, comprobar save/load/query correctos y el error tipado ante proceso no disponible. Verificar que la CLI devuelve error sin crear JSON local y que Doctor marca la ausencia de Engram. Validar la ausencia de APIs y archivos de backend JSON activos.

## Trazabilidad y fuentes

- `Axiom/packages/memory/src/{store,resolve-backend,engram-backend,index,types}.ts`
- `Axiom/packages/memory/tests/{memory,engram-backend}.test.ts`
- `Axiom/apps/cli/src/commands/{memory,context,knowledge-sync,freeze}.ts`
- `Axiom/packages/doctor/src/checks.ts`
- `Axiom/packages/doctor/tests/memory.test.ts`

## Estado de validación humana

Implementación terminada y evidencia de cierre reunida para el archive gobernado por Axiom CLI.

- La resolución de memoria solo devuelve Engram: no existen selección ni fallback JSON, `createInMemoryBackend`, `memoryFilePath`, `forceJson` ni `kind: 'json'` en las fuentes runtime activas. Los JSON históricos no se migran, borran ni consumen.
- `resolveMemoryBackend()` comprueba la disponibilidad, inicia `engram mcp --project=<projectId> --tools=agent` y realiza el handshake estándar `initialize()`. Un fallo de proceso, handshake o tool se propaga como `MemoryError` visible, sin listas vacías ni guardados aparentes.
- El backend conserva el scope por proceso, no envía un `project` redundante a las tools, decodifica el envelope de `mem_get_observation.result` y mantiene compatibilidad con respuestas raw. El límite por defecto de consulta CLI es `10`.
- TC-024 comprueba `engram --version` con timeout y remediación visible; en este entorno se observó `engram 1.17.0`. No se ejecutó `engram mcp --help`, porque inicia un servidor stdio persistente.
- La validación dirigida compartida del lote pasó: `npx vitest run packages/providers/tests/stdio-mcp-client.test.ts packages/memory/tests/memory.test.ts packages/memory/tests/evidence.test.ts packages/memory/tests/engram-backend.test.ts apps/cli/tests/memory.test.ts apps/cli/tests/knowledge-sync.test.ts apps/cli/tests/freeze.test.ts packages/doctor/tests/memory.test.ts packages/mcp-tools/tests/registry.test.ts packages/mcp-server/tests/server.test.ts apps/cli/tests/mcp-serve.test.ts --reporter=dot` ejecutó 94 pruebas correctas; `npm run build` pasó; el smoke `node apps/cli/dist/index.js memory query --query "axiom" --limit 1 --path "C:\\repos\\Axiom Workspace\\Axiom"` pasó; `npm run doctor` terminó con `46/61 OK, 0 fail, 4 warnings, 11 skipped`; y `git diff --check` terminó con código 0 (solo avisos CRLF).

## Revisión contra criterios de aceptación

La revisión independiente de las zonas de riesgo confirmó todos los criterios: Engram es la única ruta de persistencia, el pin de proyecto se conserva al iniciar el proceso, los errores son observables, Doctor falla de forma explícita si falta el binario y no quedan APIs activas de persistencia JSON. No se detectaron regresiones nuevas ni se aplicaron mutaciones Git.

## Integración documental y contexto técnico

La reconciliación final ya actualizó los claims activos de `specs/00_Resumen_Ejecutivo.md`, `01_Requisitos_Funcionales.md`, `03_Modelo_Operativo_y_Datos.md`, `05_Interfaces_Operativas.md`, `06_Integraciones_y_Capacidades.md`, `07_Gobierno_y_Seguridad.md`, `08_Glosario.md`, `specs/manuales/04_Generar_Spec_y_Contexto_Tecnico.md` y `specs/manuales/13_Skills_Agentes_y_Roles.md`. También se actualizaron `context/architecture/03-ciclo-de-vida-cli-y-orquestacion.md`, `context/integrations/01-capabilities-providers-y-toolchain.md` y `context/operations/02-doctor-troubleshooting-y-telemetria.md` con conocimiento estable sobre Engram obligatorio, errores visibles y TC-024. No se creó documentación técnica ni índices nuevos; cualquier índice derivado se valida o regenera exclusivamente mediante Axiom CLI.
