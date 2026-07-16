# 12. Plugin de Azure DevOps

El tracker instalado en este workspace: qué completar al crear incrementos/bugs, y dónde va la API key.

## El fichero de configuración: `tracker.json`

El plugin de Azure DevOps se activa por proyecto mediante un fichero de estado runtime (no versionado):

```
.axiom-state/<project>/tracker.json
```

Forma exacta:

```jsonc
{
  "kind": "ado",
  "enabled": true,
  "organization": "<org-de-azure-devops>",
  "project": "<proyecto-de-azure-devops>",
  "auth": {
    "patEnvVar": "<NOMBRE_DE_LA_VARIABLE_DE_ENTORNO>"
  }
}
```

- **`kind`**: `"none"` (default seguro, sin tracker) o `"ado"`.
- **`enabled`**: gate explícito — aunque `kind` sea `"ado"`, con `enabled: false` el proyecto se comporta igual que si no hubiera tracker (resuelve a un tracker nulo, sin red).
- **`organization`** / **`project`**: identifican la organización y el proyecto de Azure DevOps. El tracker solo se considera "configurado de verdad" cuando `kind:"ado"` + `enabled:true` + ambos campos presentes — cualquier otra combinación degrada silenciosamente a "no configurado".
- **`auth.patEnvVar`** (opcional): el NOMBRE de una variable de entorno donde vive el PAT (Personal Access Token). Alternativamente, `auth.secretKey` puede indicar una clave custom del almacén de secretos local.

## Dónde va la API key (el PAT) — NUNCA en el repo

El PAT nunca se escribe en `tracker.json` ni en ningún fichero versionado. Se resuelve en tiempo de ejecución, en este orden:

1. **Variable de entorno del proceso**: `process.env[<patEnvVar>]`, si `patEnvVar` está declarado y la variable tiene valor.
2. **Variable de entorno de usuario de Windows** (USER-scoped, vía `[System.Environment]::GetEnvironmentVariable(..., 'User')`) con el mismo nombre — cubre el caso de una variable seteada a nivel de usuario de Windows que aún no está en el `process.env` de la sesión actual.
3. **Almacén de secretos local (`SecretStore`)**, bajo la clave `auth.secretKey` si se declaró, o si no bajo la clave default derivada: `axiom.ado.<organization>.<project>.pat`.
4. **Prompt interactivo** (solo en contextos interactivos): si ninguna de las anteriores resolvió un valor, se pide el PAT por prompt (enmascarado) y, una vez introducido, se persiste en el almacén de secretos local bajo esa misma clave para no volver a pedirlo.

En resumen: para configurar el PAT, lo más simple es exportar una variable de entorno con el nombre que pusiste en `patEnvVar` (p. ej. `AZURE_DEVOPS_PAT`) antes de correr Axiom — nunca hace falta escribir el token en ningún fichero del repo.

## Qué completar al crear un incremento o un bug

Cuando el tracker está configurado (`enabled:true` + org/project presentes), el launcher ofrece — tras una creación exitosa de incremento o bug — una tarjeta de sugerencia de work item pre-rellenada:

- **incremento → `User Story`**
- **bug → `Bug`**

El título se toma del formulario de creación (título introducido → si no, el id del artefacto → si no, una etiqueta por defecto), editable antes de confirmar. Es un paso de un clic **confirm-gated**: primer clic previsualiza, segundo clic confirma y crea el work item real en Azure DevOps. Si el tracker NO está configurado, la tarjeta se muestra igual pero solo informativa ("ADO no configurado"), sin botón ni tráfico de red. Esto NO reemplaza ni acopla el ciclo de vida — crear el incremento/bug en Axiom funciona exactamente igual con o sin el plugin; el plugin solo AÑADE la opción de reflejarlo en Azure DevOps.

## El panel manual de workflow ADO

Además de la sugerencia automática en el flujo de creación, el launcher (pestaña "ADO & Git") expone un panel de workflow manual, cada acción con su propio formulario preview → confirmar:

- **Crear work item** (tipo/título/campos libres);
- **Cambiar estado** (transición de work item existente);
- **Estimar** (story points / esfuerzo);
- **Marcar horas** (registro de tiempo);
- **Enlazar rama/PR/commit** (git-link, `ArtifactLink`);
- **Enlazar con el artefacto de Axiom** (vincula el work item con el `externalRef` del incremento/bug/plan).

Todas estas acciones están config-gated: si `tracker.kind` no es `ado`, el panel muestra "ADO no configurado" sin intentar red.

## Sincronización manual (`axiom external-sync`)

```bash
axiom external-sync azure-devops ...
```

Es la ruta de sincronización explícita por CLI hacia Azure DevOps, respaldada por el mismo cliente real (`@axiom/tracker-ado`) que usan el panel del launcher y la sugerencia automática — ninguna ruta duplica la lógica de comunicación con ADO.

## Relacionado

- [11_Launcher_Visual.md](11_Launcher_Visual.md)
- [05_Incrementos.md](05_Incrementos.md)
- [06_Bugs.md](06_Bugs.md)
- [02_Configuracion.md](02_Configuracion.md)
- [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md)
