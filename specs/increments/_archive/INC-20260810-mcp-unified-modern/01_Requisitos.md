# 01 Requisitos

## Objetivo del documento

Definir el comportamiento funcional del único broker MCP gestionado de Axiom y
su migración al protocolo MCP `2026-07-28`.

## Requisitos del incremento

### RF-030-01. Broker único

Axiom debe materializar y lanzar una única entrada MCP gestionada por proyecto:
`axiom-mcp-broker`. El comando debe resolver el repo Axiom asociado al proyecto
y no depender de que el cliente conozca repos SDD y Spec separados.

### RF-030-02. Superficie completa

`tools/list` debe exponer la unión de todas las capabilities registradas por
`@axiom/mcp-tools` en los dominios `sdd`, `spec`, `axiom` y `memory`. Los dominios
siguen siendo identificadores internos; no deben producir procesos separados.

### RF-030-03. Lectura y escritura

El mismo broker debe permitir:

- leer incrementos, bugs, planes, ADR, decisiones, contexto técnico, skills,
  topología, migración y memoria;
- validar o reconstruir índices;
- previsualizar y confirmar transiciones de workflow;
- previsualizar y confirmar operaciones Git permitidas;
- rechazar transiciones ilegales, scopes ajenos y mutaciones sin confirmación
  según las reglas existentes.

### RF-030-04. Protocolo moderno

El servidor debe implementar únicamente MCP `2026-07-28`:

- `server/discover` es obligatorio;
- cada petición incluye en `_meta` la versión y las capacidades del cliente;
- una versión no soportada devuelve `-32022` con las versiones soportadas;
- las respuestas incluyen `resultType`;
- las respuestas cacheables incluyen `ttlMs` y `cacheScope`;
- el servidor incluye su identidad en `_meta`;
- `tools/list` devuelve las tools en orden determinista;
- no se implementan `initialize`, `notifications/initialized` ni `ping`.

### RF-030-05. Transporte y cancelación

El transporte stdio conserva un mensaje JSON-RPC por línea, sin ruido en stdout.
El servidor acepta `notifications/cancelled` y no debe enviar una respuesta
normal a la petición cancelada. Acepta `progressToken` conforme al protocolo y
solo emite progreso válido para operaciones activas si una operación lo necesita.

### RF-030-06. Aislamiento

El proceso se fija al proyecto al arrancar. El broker no puede leer ni mutar
otro `projectId`, `projectRoot`, repo o scope. Un binding ausente, ambiguo o
cruzado debe fallar cerrado.

## Reglas de negocio relevantes

- Las mutaciones mantienen preview/confirmación y nunca se vuelven implícitas.
- Los estados MCP, si aparecen en el futuro mediante Tasks, no sustituyen los
  estados de incrementos, bugs, planes ni workflow de Axiom.
- Engram continúa siendo un backend local opcional de memoria, no un segundo
  broker gestionado de Axiom.

## Fuera de alcance funcional

- Compatibilidad con clientes MCP legacy.
- HTTP/Streamable HTTP.
- Tasks y handles asíncronos durables.
- Nuevas capabilities no registradas actualmente.
