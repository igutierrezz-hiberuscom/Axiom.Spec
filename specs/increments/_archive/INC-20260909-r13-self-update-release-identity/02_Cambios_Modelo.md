# 02 Cambios de Modelo

## Objetivo del documento

Definir el modelo objetivo mínimo que impide confundir identidad de producto, publicación remota, checkout descargado, CLI instalado y runtime gestionado. Los nombres TypeScript son contractuales a nivel conceptual; sus rutas y nombres finales deben confirmarse contra el runtime durante la revalidación inicial del plan.

## Entidades o estructuras afectadas

### 1. Identidad de producto y build

```ts
type SemVerString = string; // validado estrictamente como SemVer 2.0.0
type GitObjectId = string;   // OID validado, todavía sin afirmar tipo de objeto
type GitCommit = GitObjectId; // OID cuyo tipo commit quedó demostrado

type IdentitySource =
  | "artifact"
  | "build"
  | "checkout"
  | "remote-live"
  | "remote-cache"
  | "managed-state"
  | "unavailable";

interface ProductIdentity {
  product: "axiom";
  distribution: "cli";
  version: SemVerString | null;
  releaseTag: string | null;
  commit: GitCommit | null;
  build: {
    id: string | null;
    builtAt: string | null; // ISO-8601 UTC
    dirty: boolean | null;
  };
  source: IdentitySource;
  provenanceId: string;
}
```

`ProductIdentity` vive en una capa del runtime/CLI que no depende de `@axiom/user-workspace`. El build debe poder generar el valor que el artefacto importa en runtime. Los builds de desarrollo pueden tener campos nulos; no pueden sustituirlos por datos de un proyecto abierto.

### 2. CLI global efectivo

```ts
interface InstalledCliIdentity {
  scope: "user-global";
  executable: {
    kind: "effective-entrypoint";
    displayPath: string | null; // saneado para salida pública
  };
  identity: ProductIdentity;
  provenanceId: string;
  candidates?: Array<{
    displayPath: string;
    reason: "path-shadow" | "project-local" | "other-global";
  }>;
}
```

`installedVersion` es `InstalledCliIdentity.identity.version`. La detección de candidatos adicionales sirve para diagnóstico, no para seleccionar silenciosamente otra autoridad. El CLI invocado desde una dependencia local no adquiere por ello `scope: user-global`; esa inconsistencia se representa mediante evidencia insuficiente o misalignment.

### 3. Checkout descargado

```ts
interface DownloadedBase {
  commit: GitCommit | null;
  dirty: boolean | null;
  provenanceId: string;
}

type DownloadedVersion =
  | (DownloadedBase & {
      kind: "tag";
      version: SemVerString;
      tag: string;
      ref: null;
      commit: GitCommit;
    })
  | (DownloadedBase & {
      kind: "ambiguous-tag";
      version: null;
      tag: null;
      ref: null;
      commit: GitCommit;
      candidates: Array<{ tag: string; version: SemVerString }>;
    })
  | (DownloadedBase & {
      kind: "ref";
      version: null;
      tag: null;
      ref: string;
      commit: GitCommit;
    })
  | (DownloadedBase & {
      kind: "commit";
      version: null;
      tag: null;
      ref: null;
      commit: GitCommit;
    })
  | (DownloadedBase & {
      kind: "unavailable";
      version: null;
      tag: null;
      ref: null;
      commit: null;
      dirty: null;
      reason: "not-a-repository" | "unborn-head" | "git-unavailable" | "invalid-output";
    });
```

La prioridad `tag` → `ref` → `commit` solo elige una representación; no altera HEAD. Si varios tags canónicos de distinta precedencia describen exactamente HEAD, se usa `ambiguous-tag`, se conservan candidatos ordenados por tag y se emite `checkout-tag-ambiguous`; no se escoge uno arbitrariamente. Varios aliases de igual precedencia y mismo commit siguen la regla determinista de aliases definida para la publicación.

### 4. Release publicada

```ts
interface PublishedVersion {
  version: SemVerString;
  tag: string;
  aliasTags: string[]; // incluye tag; orden byte-wise ascendente
  targetObjectId: GitObjectId;
  commit: GitCommit | null;
  targetType: "commit" | "unverified";
  remote: {
    repositoryId: string; // identidad normalizada, no URL con credenciales
    displayUrl: string;   // saneada
    matchedRemoteName: string | null;
  };
  refsPolicy: "canonical-v-semver-v1";
}
```

`publishedVersion` es `PublishedVersion.version`. `git ls-remote` demuestra el OID directo/peeled, pero no siempre el tipo del objeto; por ello el contrato conserva `targetObjectId` y solo rellena `commit`/`targetType: commit` cuando el objeto ya está disponible y Git demuestra su tipo sin traerlo. Un target demostrado como no-commit invalida ese tag para releases. Un target no disponible localmente puede identificar la release por tag/SemVer, pero no sirve para ancestry y mantiene `commit: null`, `targetType: unverified`.

Para candidatos de máxima precedencia SemVer:

1. si apuntan a distintos `targetObjectId`, no existe ganador y la publicación queda ambigua/`unknown`;
2. si apuntan al mismo target, son aliases de una única release: se prefiere como representación el único tag sin build metadata cuando exista; en otro caso se elige el tag byte-wise menor;
3. todos los nombres quedan en `aliasTags` ordenados. Esta regla elige representación, no inventa precedencia SemVer.

### 5. Runtime gestionado por proyecto

```ts
interface ManagedRuntimeIdentity {
  scope: "project";
  version: SemVerString | null;
  provenanceId: string;
}

interface ManagedRuntimeState {
  version: SemVerString | null;
  // Resto de campos existentes, sin redefinirlos en este incremento.
}

interface ManagedState {
  runtime: ManagedRuntimeState;
}
```

Este incremento fija semántica, no una migración general del schema: `ManagedState.runtime.version` pertenece al proyecto y expresa la versión de runtime declarada/materializada por ese proyecto según su contrato actual. No es alias de `installedVersion`, no identifica la release Git y no se escribe al ejecutar status.

### 6. Observación y procedencia

```ts
type EvidenceSubject =
  | "product"
  | "installed"
  | "downloaded"
  | "published"
  | "managed-runtime"
  | "cache";

type EvidenceFreshness = "current-process" | "live" | "cached" | "unavailable";

interface ReleaseProvenance {
  id: string;
  subject: EvidenceSubject;
  source: IdentitySource;
  freshness: EvidenceFreshness;
  method:
    | "embedded-artifact"
    | "git-local-read"
    | "git-ls-remote"
    | "offline-cache"
    | "managed-state-read"
    | "unavailable";
  observedAt: string | null; // ISO-8601 UTC
  ageMs: number | null;
  stale: boolean;
  repositoryId: string | null;
  refsPolicy: string | null;
}

type PublishedObservation =
  | {
      availability: "available";
      value: PublishedVersion;
      freshness: "live" | "cached";
      observedAt: string;
      ageMs: number;
      stale: boolean;
      provenanceId: string;
    }
  | {
      availability: "unavailable";
      value: null;
      freshness: "unavailable";
      observedAt: null;
      ageMs: null;
      stale: false;
      provenanceId: string;
      reason:
        | "canonical-remote-unresolved"
        | "remote-unreachable"
        | "remote-invalid-output"
        | "no-valid-release"
        | "ambiguous-release";
    };

interface ReleaseObservation {
  schemaVersion: 1;
  product: ProductIdentity;
  installed: InstalledCliIdentity | null;
  downloaded: DownloadedVersion;
  published: PublishedObservation;
  managedRuntime: ManagedRuntimeIdentity | null;
  provenance: ReleaseProvenance[];
  diagnostics: ReleaseDiagnostic[];
}
```

La frescura pertenece a cada evidencia, no a toda la observación: identidad del artefacto y checkout pueden haberse leído en el proceso actual mientras la publicación procede de caché o no está disponible. `offline` sin caché se representa con `PublishedObservation.availability: unavailable`, no con una frescura falsa. Cada identidad contiene un `provenanceId` que debe resolver a una entrada de `provenance` del mismo assessment.

`stale` no es un TTL ni un juicio configurable en este incremento: es `true` para toda evidencia `cached` y `false` para evidencia `live`, `current-process` o `unavailable`. `ageMs` aporta la antigüedad exacta para que políticas futuras decidan umbrales sin cambiar la verdad de esta base.

### 7. Caché offline

```ts
interface ReleaseCacheKey {
  schemaVersion: 1;
  repositoryId: string;
  refsPolicy: "canonical-v-semver-v1";
}

interface ReleaseCacheEntry {
  key: ReleaseCacheKey;
  observedAt: string;
  published: PublishedVersion;
  provenance: {
    source: "remote-live";
    method: "git-ls-remote";
    displayUrl: string;
  };
}
```

La caché se ubica en el almacenamiento user-level ya adoptado por el runtime, en un namespace propio y versionado. No forma parte de `ManagedState`, no se guarda dentro del checkout y no contiene errores transitorios ni credenciales. El lector debe validar schema, clave, timestamp, SemVer, tag, aliases y target OID antes de usarla. Al materializarla, su evidencia cambia a `freshness: cached`, `stale: true` y edad calculada; no reescribe el timestamp original.

### 8. Diagnósticos

```ts
type ReleaseDiagnosticCode =
  | "canonical-remote-unresolved"
  | "remote-unreachable"
  | "remote-auth-noninteractive"
  | "remote-timeout"
  | "remote-invalid-output"
  | "invalid-release-tag"
  | "release-target-not-commit"
  | "release-target-unverified"
  | "semver-precedence-ambiguous"
  | "checkout-unavailable"
  | "checkout-tag-ambiguous"
  | "ancestry-unavailable"
  | "installed-identity-unavailable"
  | "multiple-cli-candidates"
  | "cache-missing"
  | "cache-invalid"
  | "cache-clock-skew";

interface ReleaseDiagnostic {
  code: ReleaseDiagnosticCode;
  severity: "info" | "warning";
  subject: "published" | "downloaded" | "installed" | "managed-runtime" | "cache";
  message: string;
}
```

Los mensajes no son la API; `code` y `subject` permiten automatización y pruebas sin depender de texto localizado. Salida remota mal formada es un fallo esperado tipado, no `remote-unreachable` ni una excepción interna.

## Contratos o estados afectados

### Evidencia de comparación y unión de hechos

```ts
type IdentitySubject = "published" | "downloaded" | "installed" | "managed-runtime";
type ComparisonBasis = "semver" | "git-ancestry" | "identity" | "evidence-gap";

interface StateEvidence {
  subjects: [IdentitySubject, ...IdentitySubject[]];
  basis: ComparisonBasis;
  values: Array<{
    subject: IdentitySubject;
    value: string | null;
  }>;
  provenanceIds: string[];
}

type ReleaseState =
  | { kind: "updated"; subject: "installed"; evidence: StateEvidence & { basis: "semver" } }
  | { kind: "update-available"; subject: "installed"; evidence: StateEvidence & { basis: "semver" } }
  | { kind: "checkout-behind"; subject: "downloaded"; evidence: StateEvidence & { basis: "semver" | "git-ancestry" } }
  | { kind: "ahead"; subject: "downloaded" | "installed"; evidence: StateEvidence & { basis: "semver" | "git-ancestry" } }
  | { kind: "installed-misaligned"; subject: "installed"; evidence: StateEvidence & { basis: "semver" | "identity" } }
  | { kind: "unknown"; subject: IdentitySubject; evidence: StateEvidence & { basis: "evidence-gap" } }
  | { kind: "offline"; subject: "published"; evidence: StateEvidence & { basis: "evidence-gap" } };

type PrimaryReleaseState = ReleaseState["kind"];

interface ReleaseAssessment {
  schemaVersion: 1;
  observedAt: string; // instante UTC en que se compuso el assessment
  primaryState: PrimaryReleaseState;
  states: ReleaseState[];
  observation: ReleaseObservation;
}
```

`subjects` identifica todas las partes comparadas; `values` conserva sus representaciones saneadas y `provenanceIds` enlaza la evidencia exacta. Para `offline`, el enlace apunta a la evidencia del intento remoto fallido y, si se usó caché, también a la publicación histórica. Para `unknown`, el valor ausente es `null` y el diagnóstico explica el gap.

Los hechos pueden coexistir porque responden a preguntas distintas. `updated` afirma que el CLI global coincide con la release publicada; `checkout-behind` afirma que el checkout no coincide; `installed-misaligned` muestra que instalación y checkout son diferentes. Ninguno autoriza una mutación.

### Matriz de derivación mínima

| Evidencia demostrable | Hecho emitido |
|---|---|
| `installed == published` por precedencia e identidad compatible | `updated` |
| `installed < published` por SemVer | `update-available` |
| `installed > published` por SemVer | `ahead(subject=installed)` |
| `downloaded < published` por SemVer o checkout ancestor | `checkout-behind` |
| `downloaded > published` por SemVer o checkout descendant | `ahead(subject=downloaded)` |
| `installed != downloaded` y son comparables | `installed-misaligned` |
| falta/ambigüedad impide una comparación requerida | `unknown` para el sujeto |
| no se obtuvo observación live del remoto | `offline` |

Un checkout por ref o commit solo se clasifica ahead/behind por ancestry cuando los objetos necesarios ya existen y son commits demostrados. La simple desigualdad de SHA/OID no demuestra dirección.

### Orden y estado primario

`states` se ordena con la misma prioridad normativa usada para elegir el primario:

1. `offline`
2. `unknown`
3. `installed-misaligned`
4. `ahead`
5. `checkout-behind`
6. `update-available`
7. `updated`

Dentro de un mismo `kind`, el desempate es `subject` byte-wise ascendente y después la clave estable formada por `evidence.subjects` y `evidence.values`. Se eliminan únicamente duplicados estructuralmente idénticos. `primaryState` es el `kind` del primer elemento; `states` nunca está vacío, porque falta total de evidencia produce al menos `unknown`.

Un estado primario `offline` puede acompañar una comparación histórica derivada de caché; la UI/CLI debe mostrar la procedencia y no presentarla como frescura live.

### Contrato compartido de status

La capa de aplicación debe producir un único `ReleaseAssessment`, incluido su `observedAt`. La salida humana y `--json` son presentadores del mismo objeto. Futuras superficies pueden consumirlo sin importar tipos de UI ni acceder directamente a Git, caché, `ManagedState` o metadata de build.

## Notas de compatibilidad

- La separación es aditiva a nivel conceptual. La implementación debe adaptar los contratos existentes tras revalidarlos; no se presupone que los nombres o archivos aquí mostrados existan hoy.
- `ManagedState.runtime.version` conserva su almacenamiento project-scoped. Si el schema actual requiere aclaraciones o adaptadores, deben ser compatibles hacia atrás y no convertir esta tarea en una migración de manifest v2.
- La caché usa schema y namespace propios. Entradas desconocidas se ignoran; no se migran ni borran de forma destructiva.
- Los consumidores actuales de un literal de versión deben migrar al módulo de identidad del runtime. No se mantiene un fallback a `@axiom/user-workspace`, porque perpetuaría dos autoridades.
- La serialización JSON usa campos opcionales/nulos explícitos y una `schemaVersion`. Agregar campos futuros puede ser compatible; renombrar estados o cambiar su semántica requiere versión de contrato.
- La ausencia de Git, checkout, red, caché o proyecto es un caso soportado de observación parcial, no una razón para inventar valores por defecto.
- Un OID remoto no se serializa como commit hasta demostrar su tipo. Esta limitación preserva el descubrimiento read-only y se comunica mediante target/provenance/diagnóstico.
- No se cambia en este incremento el formato completo de artefactos de release, el mecanismo de instalación ni el pipeline que genera tags; esos consumidores deben adoptar el contrato en incrementos 2–5.
