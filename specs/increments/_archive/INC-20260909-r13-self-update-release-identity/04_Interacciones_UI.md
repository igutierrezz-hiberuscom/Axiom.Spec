# 04 Interacciones UI

## Objetivo del documento

Definir únicamente el contrato de interacción compartido y la superficie CLI de consulta del incremento. Aquí “UI” significa presentación de estado por línea de comandos y un payload reutilizable por consumidores futuros. La consulta es read-only respecto de Git, instalación y proyecto; su única escritura permitida es la caché user-level tras una observación remota live válida. **No se diseña ni implementa Launcher UI** en este incremento.

## Superficie UI afectada

### Comando planificado

```text
axiom self-update status
axiom self-update status --json
```

El nombre debe revalidarse contra la convención del CLI antes de implementarlo, pero su semántica es cerrada: observar identidades, evaluar hechos y mostrarlos. No descarga, instala, aplica, revierte, edita el proyecto ni solicita confirmación para hacerlo.

La forma humana y `--json` reciben un único `ReleaseAssessment` de la capa de aplicación. Ningún renderer vuelve a consultar Git, metadata, caché o `ManagedState`, y ninguno reinterpreta las reglas de SemVer.

### Contrato JSON compartido

El payload mínimo sigue directamente el modelo de `ReleaseAssessment`:

```json
{
  "schemaVersion": 1,
  "observedAt": "2026-09-09T12:00:00.000Z",
  "primaryState": "updated",
  "states": [
    {
      "kind": "updated",
      "subject": "installed",
      "evidence": {
        "subjects": ["installed", "published"],
        "basis": "semver",
        "values": [
          { "subject": "installed", "value": "1.5.0" },
          { "subject": "published", "value": "1.5.0" }
        ],
        "provenanceIds": ["installed:artifact", "published:live"]
      }
    }
  ],
  "observation": {
    "schemaVersion": 1,
    "product": {},
    "installed": {},
    "downloaded": {},
    "published": {
      "availability": "available",
      "value": {},
      "freshness": "live",
      "observedAt": "2026-09-09T12:00:00.000Z",
      "ageMs": 0,
      "stale": false,
      "provenanceId": "published:live"
    },
    "managedRuntime": null,
    "provenance": [
      {
        "id": "installed:artifact",
        "subject": "installed",
        "source": "artifact",
        "freshness": "current-process",
        "method": "embedded-artifact",
        "observedAt": "2026-09-09T12:00:00.000Z",
        "ageMs": 0,
        "stale": false,
        "repositoryId": null,
        "refsPolicy": null
      },
      {
        "id": "published:live",
        "subject": "published",
        "source": "remote-live",
        "freshness": "live",
        "method": "git-ls-remote",
        "observedAt": "2026-09-09T12:00:00.000Z",
        "ageMs": 0,
        "stale": false,
        "repositoryId": "axiom-canonical",
        "refsPolicy": "canonical-v-semver-v1"
      }
    ],
    "diagnostics": []
  }
}
```

Los objetos concretos siguen [02_Cambios_Modelo.md](./02_Cambios_Modelo.md). El ejemplo muestra forma, no valores por defecto. Un dato desconocido se representa con `null`, `availability: unavailable` o una variante `unavailable`, acompañado de diagnóstico; nunca se copia desde otra identidad. Todo `provenanceId` debe resolver dentro de `observation.provenance` en una respuesta real.

El `observedAt` superior es el instante en que se compuso el assessment. No sustituye el timestamp de una publicación cacheada: esa publicación conserva su `observedAt`, edad, `freshness: cached` y `stale: true`.

### Salida humana

El orden de bloques es estable para facilitar lectura y snapshots:

1. estado primario y hechos adicionales;
2. `published`: SemVer/tag/aliases/target y origen live/cache/unavailable;
3. `downloaded`: tag, tags ambiguos, ref o commit y atributo dirty;
4. `installed`: versión y procedencia del CLI global efectivo;
5. `managed runtime`: versión project-scoped o “not in project/unavailable”;
6. frescura, timestamps y advertencias accionables sin ofrecer apply.

La salida no depende solo de color, no imprime secretos ni rutas de usuario completas y usa lenguaje que distingue certeza live, evidencia cacheada y desconocimiento. Si un OID remoto no está demostrado como commit, se muestra como target no verificado y no como commit.

## Flujo de interacción

1. El usuario invoca status, opcionalmente con `--json`.
2. El CLI valida argumentos. No abre prompt ni solicita credenciales.
3. La capa de aplicación lee la identidad embebida del entrypoint global efectivo.
4. Si existe contexto de proyecto, lee `ManagedState.runtime.version` sin mutarlo; si no existe, continúa.
5. Observa el checkout local con comandos Git de lectura y optional locks/refrescos deshabilitados. No exige checkout para consultar la publicación.
6. Resuelve la identidad del remoto canónico y realiza una consulta remota no interactiva sin actualizar refs u objetos.
7. Si la consulta live es válida, la usa y actualiza atómicamente la caché user-level. Si falla por indisponibilidad, intenta una entrada cacheada compatible y conserva su antigüedad/procedencia. Sin caché, la publicación queda `unavailable`.
8. El evaluador produce todos los hechos aplicables, su evidencia y un `primaryState`; ordena los hechos con la prioridad normativa.
9. El renderer text o JSON emite el mismo assessment. Los estados de dominio retornan éxito si el contrato fue generado; solo uso inválido o fallo interno no clasificado devuelven error conforme a la convención CLI existente.
10. El proceso termina sin modificar checkout, instalación, PATH, `ManagedState` ni configuración Git. La única posible escritura ya realizada es la caché válida del paso 7.

No hay refresco reactivo, watcher ni segunda fase de confirmación. Una nueva observación requiere otra invocación explícita.

## Estados visibles

| Estado | Mensaje conceptual | Evidencia que debe acompañarlo | Acción en este incremento |
|---|---|---|---|
| `updated` | El CLI global coincide con la release publicada observada. | Installed/published, valores, base SemVer y procedencias. | Ninguna. |
| `update-available` | Existe una release publicada con mayor precedencia que la instalación. | Installed/published SemVer y procedencias. | Informar; no descargar ni aplicar. |
| `checkout-behind` | El checkout es demostrablemente anterior a la publicación. | Downloaded/published, tag/ref/commit y base SemVer/ancestry. | Informar; no hacer pull/checkout. |
| `ahead` | El checkout o CLI es demostrablemente posterior/no publicado. | Sujeto, contraparte, valores y base SemVer/ancestry. | Informar; no publicar ni resetear. |
| `installed-misaligned` | Instalación global y checkout describen identidades distintas. | Ambas identidades, valores, base y procedencias. | Informar; no reemplazar ninguna. |
| `unknown` | Falta evidencia o existe ambigüedad. | Sujeto, valor nulo/ambiguo, procedencia y diagnóstico tipado. | Informar el límite; no adivinar. |
| `offline` | No se obtuvo observación remota live. | Evidencia del intento fallido y cache timestamp/edad si existe. | Usar cache compatible o mostrar published unavailable. |

Los estados pueden coexistir. La forma humana muestra el primario y resume los demás; JSON conserva el conjunto completo. `offline` nunca se presenta como “sin actualizaciones”: significa que la actualidad no pudo comprobarse. Toda publicación cacheada es histórica (`stale: true`), sin depender de un umbral oculto.

## Cascadas y comportamiento reactivo

- **Cambio de publicación live:** afecta `publishedVersion`, comparaciones y caché; no cambia checkout, instalación ni runtime gestionado.
- **Cambio de checkout:** afecta `downloadedVersion`, `checkout-behind`, `ahead` e `installed-misaligned`; no redefine la versión instalada.
- **Cambio del CLI global:** afecta `installedVersion`, `updated`, `update-available`, `ahead(subject=installed)` e `installed-misaligned`; no escribe el proyecto.
- **Cambio de proyecto:** solo puede cambiar `managedRuntime` y observación del checkout de ese directorio. La autoridad global permanece estable.
- **Fallo remoto con caché:** activa `offline`; permite comparaciones históricas y cambia la publicación a `freshness: cached`, `stale: true`.
- **Fallo remoto sin caché:** activa `offline` y `unknown`; la publicación queda `availability: unavailable`.
- **Caché corrupta o ajena:** añade diagnóstico y se ignora; no bloquea otras identidades.
- **Datos no comparables:** añade `unknown` para el sujeto sin eliminar hechos independientes demostrables.

No existe cascada automática hacia download/apply, manifest v2, Launcher o CI. El contrato se diseña para que esos incrementos posteriores lo consuman, no para anticipar su lógica visual o transaccional.
