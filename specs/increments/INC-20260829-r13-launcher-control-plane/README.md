# Control plane, transportes y confirmación del launcher R-13

> **Código**: INC-20260829-r13-launcher-control-plane
> **Estado documental**: especificación refinada; lifecycle gestionado por Axiom Core
> **Fecha**: 2026-08-29
> **Acciones**: ACC-070..ACC-072 y matriz correspondiente de ACC-076
> **Dependencias**: independiente de A-E en comportamiento; se ejecuta después de I para minimizar colisiones de CLI

## Objetivo

Cerrar el servidor launcher como control plane local autenticado, asegurar sus transportes y ligar preview/confirmación con tokens de un solo uso sobre el payload exacto.

## Revalidación

El server usa loopback por defecto pero acepta CORS `*`, carece de sesión/auth, límites y schemas uniformes; SSE está abierto; browse asciende desde home. HttpLaunch recibe endpoint del body. VS Code declara delivery sin transporte. Craft/execute no tienen binding y el frontend conserva confirmación ante edición/carreras.

## Alcance

- ACC-070: bind loopback obligatorio, sesión por proceso, auth API/SSE, same-origin, headers, schemas, límites, errores genéricos, browse confinado y SSE acotado.
- ACC-071: HttpLaunch configurado/allowlisted y resistente a SSRF; estados honestos para clipboard/VS Code.
- ACC-072: token preview de un solo uso, TTL y binding de proyecto/acción/payload; frontend anti-race/replay/edit y required validation; onboarding/plugins/Git/ADO incluidos.
- Pruebas efectivas de ACC-076 para estos contratos, sin red externa.

## No objetivos

- No implementar modo remoto.
- Doctor es diagnóstico server-side y nunca autorización.
- Catálogo/IDs/ADO copy son G; telemetría agregada es H.
- No self-update/R-13.5.

## Decisiones cerradas

1. El servidor solo acepta bind literal `127.0.0.1` o `::1`; cualquier host no-loopback falla al arrancar. No hay modo remoto implícito.
2. Al servir el bootstrap launcher se crea una sesión aleatoria de 256 bits por proceso, en cookie HttpOnly, SameSite=Strict, Path=/; API y SSE exigen cookie válida. Reinicio invalida sesiones. Tests pueden inyectar RNG.
3. Requests API validan `Host` contra listener y `Origin` exactamente contra el origin local en métodos mutantes; no se emite CORS ni se acepta preflight cross-origin. Token/cookie no sustituye same-origin.
4. JSON mutante exige `application/json`, body máximo 256 KiB, timeout de lectura 5 s y schemas runtime cerrados. Método incorrecto devuelve 405. Mensajes 5xx son genéricos y detalles quedan solo en diagnóstico local seguro.
5. Headers mínimos: CSP sin fuentes remotas/inline no autorizadas, `default-src 'self'`, frame-ancestors none, nosniff, Referrer-Policy no-referrer, Permissions-Policy restrictiva, no-store para API.
6. Browse se limita a raíces canonicalizadas del axiomRepo y code repos del proyecto resuelto; no expone home general ni permite traversal/symlink escape.
7. SSE usa sesión y same-origin, heartbeat 15 s, máximo 8 suscriptores por proceso, cola 64 KiB por cliente, cleanup en close y desconexión de cliente lento. El frontend usa transporte que autentica sin token en URL.
8. HttpLaunch no toma URL del request. Selecciona `endpointId` de configuración validada; protocolos solo http/https, allowlist exacta, resolución DNS/IP gobernada, redirects desactivados, timeout 5 s y respuesta máxima 64 KiB. Loopback solo si endpoint configurado explícitamente; private/link-local/metadata y DNS rebinding se rechazan de otro modo.
9. Se retira `VSCodeLaunch` como transporte de entrega hasta existir bridge cliente con ack. Targets VS Code pueden usar clipboard; estados son `generated`, `client-instructed`, `acknowledged`, `delivered` y solo evidence/ack permite delivered.
10. Craft normaliza y valida payload, calcula digest y almacena token aleatorio single-use ligado a session, projectId, actionId, normalized payload, server-side doctor diagnostic y expiry de 120 s.
11. Execute exige token + payload; comparación constante, mismo session/proyecto/acción/digest. El token se consume atómicamente antes del side effect y replay/race/expiry/edit falla cerrado.
12. Frontend usa requestId monotónico, ignora crafts stale, desarma confirmación ante cualquier edición, ejecuta snapshot inmutable previsualizado y valida required fields con helper común.

## Riesgos

Compatibilidad de cookies/SSE local, CSP con assets existentes, DNS hermético y gran fan-out de endpoints. Se requiere refactor central del middleware y fixtures locales, sin fetch externo.

## Compatibilidad

No se mantiene API launcher sin sesión, CORS, `confirmed:true` como autorización, endpoint arbitrario ni falso VSCode delivered.

## Validación prevista

Servidor real: auth/origin/content-type/body/timeouts/headers/browse/SSE; SSRF/redirect/DNS fixtures locales; delivery ack; token mismatch/replay/race/expiry/edit/required; build y diff-check.

## Integración estable

Diferida al final; no editar specs 00..08/context durante apply.
