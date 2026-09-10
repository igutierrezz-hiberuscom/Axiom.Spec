# 04 Interacciones UI

## Objetivo del documento

Delimitar la superficie observable de este incremento sin introducir la UI del Launcher. El incremento 3 entrega un contrato headless para preview/apply/recover y una adaptación CLI mínima; el incremento 4 será responsable de presentar ese contrato visualmente. Este documento especifica comportamiento futuro, no una UI existente.

## Superficie UI afectada

No se crean ni modifican vistas, ventanas, botones, selectores, notificaciones, barras de progreso o lógica de relaunch del Launcher.

Las únicas superficies observables incluidas son:

1. **API de runner/helpers**: `previewGlobalUpdate`, `applyGlobalUpdate`, `recoverGlobalUpdate` y `runGlobalSelfUpdate` con tipos inmutables, cancelación y eventos.
2. **Adaptador CLI existente de self-update**: debe dejar de representar un apply simulado como éxito y traducir plan, eventos, outcome y diagnósticos sin duplicar lógica del core.
3. **Salida automatizable**: objeto/JSON tipado cuando el CLI ya ofrezca ese modo, códigos de salida coherentes y mensajes humanos derivados del mismo `UpdateResult`.

El Launcher del incremento 4 será un consumidor más. No puede enviar una versión arbitraria como target ni reimplementar validación Git, activación o recovery.

## Flujo de interacción

### Preview

1. El consumidor solicita preview para la instalación global identificada.
2. El core valida que no sea una instalación inicial y resuelve una release publicada mediante los contratos de los incrementos 1 y 2.
3. El core observa instalación/Git y devuelve un `UpdatePlan` sellado más diagnósticos no sensibles.
4. El consumidor puede mostrar versión observada actual, versión target, release id, commit abreviado, estrategia, operaciones y riesgos.
5. Aceptar el preview entrega exactamente ese plan a apply. Refrescar genera otro plan; nunca se modifica el anterior en memoria.

### Apply

1. El consumidor invoca apply con el plan recibido.
2. El core adquiere lock y emite eventos de fase; el consumidor no debe inferir porcentaje ficticio ni éxito antes de `completed`.
3. Si el mundo cambió, apply devuelve `failed` y exige nuevo preview; no autoacepta otro target.
4. Ante fallo preactivación, se muestra que la versión activa quedó intacta.
5. Ante rollback probado, se muestra `failed` y la versión observada restaurada.
6. Ante incertidumbre se muestra `recovery-required` con instrucción de ejecutar recovery; no se ofrece “continuar igualmente”.
7. `installed` se muestra solo con la versión observada post-swap; `unchanged` indica que no hubo instalación nueva.

### Recover

1. El consumidor solicita recovery de la instalación, sin elegir paths a borrar ni una release a forzar.
2. El core adquiere el mismo lock, inspecciona journal y realidad y emite eventos `recovering`/`rolling-back`.
3. El consumidor presenta el outcome exacto:
   - `installed`: ya había commit durable y el target se verificó;
   - `unchanged`: el estado previo quedó o fue restaurado y verificado;
   - `failed`: recovery no era necesario o falló sin ambigüedad y el activo sigue sano;
   - `recovery-required`: sigue siendo necesaria intervención, sin afirmar reparación.
4. Repetir recovery debe ser seguro e idempotente.

### Instalación inicial

Si el probe devuelve `initial-install-required`, el consumidor deriva a un flujo de instalación inicial separado. Apply no ofrece “crear ahora” como efecto lateral. El diseño visual de esa derivación no forma parte de este incremento.

## Estados visibles

| Evento/estado headless | Información observable | Prohibiciones |
|---|---|---|
| `planning` | instalación, release source y snapshot en evaluación | no prometer target antes de validarlo |
| `locked` | operación exclusiva adquirida | no exponer pid/host más allá de lo necesario |
| `preflight` | remoto/ref/commit/Git en validación | no sugerir stash/reset automático |
| `fetching` | target publicado y commit abreviado | no mostrar credenciales de remoto |
| `installing` | ejecución de `npm ci` | no mezclar output ilimitado de subprocess |
| `building` | build completo en candidato | no tocar entry activo |
| `verifying` | version/help/load/doctor por ruta absoluta | no usar versión deseada como observada |
| `activating` | swap atómico del único entry | no presentar éxito todavía |
| `persisting` | manifest observado y commit de journal | no declarar instalado antes de commit |
| `rolling-back` | restauración en orden inverso | no ocultar que apply falló |
| `recovering` | reconciliación de journal y realidad | no ofrecer borrado manual genérico |
| `completed` | outcome, versión observada y diagnóstico final | no traducir `failed`/`recovery-required` a éxito |

Los eventos son monotónicos para una ejecución y llevan `operationId`; pueden repetirse con subtareas, pero no retroceden salvo la transición explícita a `rolling-back`/`recovering`.

### Mapping de outcomes a CLI

- `installed`: salida de éxito y código 0 únicamente tras commit y verificación.
- `unchanged`: salida informativa y código 0, dejando claro que no se instaló nada nuevo.
- `failed`: código distinto de 0 con causa accionable; si el estado previo fue verificado se indica explícitamente.
- `recovery-required`: código distinto de 0 diferenciado, instrucción de recovery y ubicación segura del diagnóstico/journal; nunca sugiere borrar carpetas o locks a mano.

La asignación numérica exacta de códigos se fijará conforme al convenio existente del CLI durante el inventario de implementación, sin cambiar esta semántica.

## Cascadas y comportamiento reactivo

- Un cambio en release publicada, ref/commit, instalación, manifest, entry o Git entre preview y apply invalida el plan. El consumidor debe solicitar un preview nuevo; no actualiza campos del plan en sitio.
- Un lock ocupado deshabilita/rechaza otro apply/recover para la misma instalación, pero no bloquea indefinidamente la UI. El diagnóstico puede incluir edad y operation id sanitizados, no un botón de “borrar lock”.
- Una cancelación antes de mutaciones produce `failed`; durante fases mutables cambia a rollback. La UI futura debe esperar el outcome terminal y no matar el proceso como mecanismo normal.
- Timeout, signal, spawn y exit no cero se presentan con fase y resumen sanitizado. El consumidor no interpreta stderr para decidir outcome.
- `recovery-required` inhibe un nuevo apply hasta ejecutar recovery o resolver la inconsistencia de forma explícita y verificable.
- Un rollback completado permite retry solo después de un nuevo preview. El plan anterior se conserva como evidencia y no se reutiliza.
- Los paths mostrados proceden del adapter canónico. Windows y POSIX deben presentar la misma identidad lógica sin concatenación manual.
- El progreso no conoce detalles visuales. El Launcher 4 podrá traducir eventos, pero no controlar transiciones internas ni tocar journal/lock.
- El incremento 5 podrá consumir outcome/evidencia para documentación y release final; no se dispara publicación desde esta interacción.
