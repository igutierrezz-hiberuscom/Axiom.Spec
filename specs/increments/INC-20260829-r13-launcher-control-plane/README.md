# Control plane, transportes y confirmación del launcher R-13

> **Código**: INC-20260829-r13-launcher-control-plane
> **Estado documental**: implementación verificada; lifecycle gestionado por Axiom Core (`verifying`); cierre `pending` por provenance no aislable
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

## Resultado de verificación (2026-09-08)

- Freeze de candidate revalidado por Core: `92406c1068e037b2cca17b2e1047d6096134d65b595f98eb06ab30929a36b061`.
- Validación focal R-13: 8 suites, 192 tests PASS; typecheck de `packages/launcher` y `apps/cli` PASS; `npm run build` PASS; `git diff --check` focal PASS.
- `npm run doctor`: PASS, 48/61 OK, 0 fallos, 2 advertencias y 11 omitidos. `npm run readiness:first-project`: PASS.
- Review independiente: `ready_for_lifecycle`; CA-F1/ACC-070, CA-F2/ACC-071, CA-F3/ACC-072 y la evidencia ACC-076 conformes en el estado observado.
- Core ejecutó `plan-approved → verifying` mediante `axiom-increment verify`. Receipts: `increment-verify` hash `256618ad116836db81767b2372c5083107b314d209049d5418abe690be43275c` y fase `verify` hash `b53eed4dc56758c26d052427146019a79b06949b6926f8c5c7326f94c7d55697`.

## Provenance y decisión de cierre

El worktree de `Axiom` contiene cambios concurrentes de numerosos incrementos y no existe una commit focal aislada; `Axiom.Spec` también tiene cambios concurrentes. Por tanto, la evidencia certifica el estado observado del worktree, pero no permite atribuir todos los resultados exclusivamente a R-13. Conforme a la regla del lote, no se ejecuta `archive`, no se marca `closed` y el cierre queda `pending` hasta disponer de provenance focal o una matriz de atribución reproducible.

La observación no bloqueante de la review sobre comentarios históricos de `VSCodeLaunch` y el fallback SPA profundo queda registrada para una limpieza/documentación posterior; no cambia el alcance funcional aceptado de R-13.

## Integración estable

Consolidada al final de la verificación, sin editar metadata estructural ni índices: el contrato del control plane local, los límites HTTP/SSE, el transporte HTTP configurado/allowlisted, los estados honestos de clipboard, el rechazo de VSCode ficticio, el single-use grant y la separación entre Doctor diagnóstico y autorización fueron reconciliados en `Axiom.Spec/specs/02_Requisitos_No_Funcionales.md`, `05_Interfaces_Operativas.md`, `06_Integraciones_y_Capacidades.md`, `07_Gobierno_y_Seguridad.md` y el contexto técnico existente. No se archivó el incremento por la provenance no certificable.
