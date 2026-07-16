# 11. Launcher visual

`axiom app`: el front de navegador que permite operar Axiom sin abrir la terminal ni la TUI.

## Abrirlo

```bash
axiom app
```

Levanta un servidor HTTP local que sirve la PWA del launcher (bajo `/launcher/`), sin ninguna API de VSCode ni dependencia de un IDE concreto (front cero-`acquireVsCodeApi`). Un miembro del equipo puede abrir esa URL en cualquier navegador y empezar a trabajar.

## Seleccionar un proyecto

El selector de proyecto se puebla desde el registro de usuario (`~/.axiom/projects.yml` / `GET /api/projects`). Si el proyecto todavía no está en ese registro, ver la sección **Onboarding** más abajo.

## Seleccionar un adapter

Cada adapter disponible (`claude-code`, `github-copilot`, `cli`, y otros configurados) aparece en un selector con una etiqueta compacta de su `agentTuning` (verbosidad/personalidad — ver [02_Configuracion.md](02_Configuracion.md)). El **prompt se pregenera automáticamente para el adapter seleccionado**: cambiar de adapter recalcula el prompt (header/mención + bloque de ajustes del agente + cuerpo neutral).

## El gate de doctor previo al lanzamiento

Al seleccionar un proyecto, el launcher pide automáticamente su reporte de `axiom doctor` (endpoint de solo lectura) y lo muestra en un panel compacto (línea de salud + lista de issues) antes del asistente de 3 pasos, con un botón "Revisar (doctor)" para refrescar manualmente. Cuando el doctor reporta al menos un `fail`, los dos botones que mutan algo (**Ejecutar** y **Lanzar**) exigen un **segundo clic de confirmación explícito** (arma → confirma), apilado antes de su propia confirmación normal — es una advertencia visible, no un bloqueo duro: el usuario puede igualmente decidir seguir adelante.

## El asistente guiado de 3 pasos (pestaña Crear)

1. "¿Qué querés hacer?" (familia: increment / bug / plan / implementación — back/front/e2e);
2. modo/acción concreta dentro de esa familia;
3. formulario dinámico (solo los campos relevantes según lo elegido, con condiciones `visibleWhen`).

Al completarlo se muestra el **prompt pregenerado** con un botón de copiar-a-un-clic, más un botón de **Ejecutar** (scaffold real de la estructura vía las mismas funciones que usa la CLI, siempre con preview → confirmar) y de **Lanzar** (abre el adapter con el prompt). Si el plugin de Azure DevOps está configurado y la acción fue crear un incremento/bug, aparece además una tarjeta de sugerencia para crear el work item correspondiente — ver [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md).

## Onboarding: instalar o unirse sin terminal

La pestaña **Instalar / Unirse** cubre, con tres tarjetas, exactamente lo que antes requería la CLI/TUI:

- **Instalar**: crea un proyecto Axiom NUEVO (equivalente a `axiom init`), con un formulario de nombre/perfil/overlay/layout/adapter.
- **Unirse**: registra un proyecto Axiom YA EXISTENTE en el registro de esta máquina (equivalente a `axiom projects join`).
- **Roles**: registra un rol de equipo/código y lo asocia a un repo (equivalente a `axiom roles register` + `axiom roles assign`).

Las tres comparten un **selector de carpeta** (folder-picker) que lista subdirectorios del servidor para elegir rutas sin escribirlas a mano. Toda mutación (instalar/unirse/registrar rol/asignar rol) sigue el mismo patrón preview → confirmar: sin confirmar solo se ve una previsualización, sin tocar el disco. Tras un instalar/unirse exitoso, el selector de proyecto se refresca automáticamente y el proyecto nuevo/unido aparece de inmediato, listo para usarse en el resto del launcher (Crear, ADO & Git, etc.). Bajo el capó, estos tres flujos llaman a las MISMAS funciones que usa la CLI (`runInit`, `runProjectsJoin`, `runRolesRegister`, `runRolesAssign`) — nada se reimplementa.

## Registro (pestaña "Registro")

Vista read-only sobre el registro de proyectos/roles conocidos por esta máquina, con actualización en vivo vía un canal de eventos (SSE) cuando cambia algo (p. ej. tras una mutación git confirmada).

## ADO & Git (si el plugin está configurado)

Panel con sugerencias de work items de Azure DevOps (lectura), y paneles de rama de rol / commit-sync (preview → confirmar, git local-only, sin push por defecto), más el formulario de workflow ADO completo. Ver [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md).

## Ejecutar/confirmar (patrón general)

Toda acción que muta algo en el launcher sigue el mismo patrón de dos pasos: primer clic = previsualización (sin tocar nada); segundo clic explícito = confirmar y ejecutar. Ningún botón muta en un solo clic.

## Relacionado

- [02_Configuracion.md](02_Configuracion.md)
- [05_Incrementos.md](05_Incrementos.md)
- [09_Revisiones.md](09_Revisiones.md)
- [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md)
- [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md)
