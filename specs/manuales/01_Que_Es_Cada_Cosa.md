# 01. Qué es cada cosa

Mapa conceptual rápido de las piezas de Axiom, para orientarse antes de tocar nada.

## Qué es Axiom

Axiom es un producto que aplica una disciplina SDD (Spec-Driven Development) ligera: primero se especifica el comportamiento esperado (goal, scope, non-goals, acceptance criteria), luego se implementa, se valida y se revisa, y finalmente se cierra e integra el conocimiento estable en la spec canónica. No es un framework de ciclo de vida empresarial pesado — evita índices obligatorios, metadatos complejos y jerarquías de carpetas especulativas salvo que se pidan explícitamente.

## El ciclo SDD ligero

En su forma más simple: entender el pedido → localizar o crear la spec del incremento/bug → refinarla (goal/contexto/scope/non-goals/acceptance criteria) → implementar → validar → revisar contra el intent original → cerrar o dejar `pending` con motivo explícito → integrar conocimiento estable en la spec general. Ver el detalle completo en [05_Incrementos.md](05_Incrementos.md) y [06_Bugs.md](06_Bugs.md).

## Roles de repo (`sdd` / `spec` / `code`)

Un workspace Axiom multi-repo separa responsabilidades por repo:

- **`sdd`** (repo de control): orquesta el setup, aloja artefactos project-scoped no-por-repo (`topology.yaml`, bindings locales, config MCP del proyecto).
- **`spec`**: el conocimiento canónico — specs numeradas 00–08, incrementos, bugs, planes, decisiones. Es donde vive el ciclo de vida de artefactos.
- **`code`** (uno por rol funcional/de implementación: back, front, qa-e2e, o cualquier nombre custom): el runtime instalable de ese rol.

No confundir con el `functionalProfile` del profile triple (`builder`/`product-owner`), que es un eje distinto — ver [02_Configuracion.md](02_Configuracion.md).

## La spec canónica (00–08 + artefactos)

`Axiom.Spec/specs/` sigue una estructura numerada fija: `00_Resumen_Ejecutivo.md` (visión/alcance), `01_Requisitos_Funcionales.md`, `02_Requisitos_No_Funcionales.md`, `03_Modelo_Operativo_y_Datos.md`, `04_Flujos_SDD_y_Ciclo_de_Vida.md`, `05_Interfaces_Operativas.md`, `06_Integraciones_y_Capacidades.md`, `07_Gobierno_y_Seguridad.md`, `08_Glosario.md`. Junto a esos ficheros viven las carpetas de artefacto: `increments/`, `bugs/`, `archive/` (histórico integrado o cerrado). Ver [04_Generar_Spec_y_Contexto_Tecnico.md](04_Generar_Spec_y_Contexto_Tecnico.md) para cómo se scaffoldea/adopta esta estructura en un proyecto nuevo.

## Perfiles funcionales (`builder` / `product-owner`)

El "profile triple" combina `functionalProfile` (`builder` habilita `code.*` + roles de implementación; `product-owner` queda limitado a spec/SDD, con `code.*` bloqueado) + `operationalOverlay` (`local-only`/`standard`/`enterprise`, el modo de riesgo/compliance) + `adapterTarget`. Se persiste en `axiom.yaml` y se resuelve a un `ResolvedInstallProfile`. Ver [02_Configuracion.md](02_Configuracion.md).

## Adapters

Un adapter es la capa de traducción entre las capacidades de Axiom y un IDE/CLI externo concreto (`opencode`, `claude-code`, `github-copilot`, `vscode`, `cursor`, `litellm`, más `copilot-vscode`/`antigravity`/`visual-studio-2026` sin adapter dedicado). Cada adapter tiene un "support level" (`multi-mode`, `single-mode`, `fallback-only`) que determina cuánto control de ruteo por slot ofrece. Ver [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md).

## El launcher (`axiom app`)

El launcher es el front de navegador (PWA servida por `axiom app`) que permite operar Axiom sin abrir la terminal ni la TUI: seleccionar proyecto y adapter, ver el prompt pregenerado, ejecutar transiciones confirm-gated, instalar/unirse a un proyecto, registrar roles, y (si el tracker está configurado) crear el work item correspondiente en Azure DevOps. Ver el manual dedicado: [11_Launcher_Visual.md](11_Launcher_Visual.md).

## El doctor

`axiom doctor` ejecuta checks de salud estructural y operativa del proyecto (boundaries, policies, manifests, isolation, capability model, gateway) y reporta PASS/FAIL con exit code. El launcher usa el mismo motor como un gate visual antes de lanzar/ejecutar — ver [11_Launcher_Visual.md](11_Launcher_Visual.md).

## El tracker/plugin (concepto)

Un tracker es un sistema externo de seguimiento de trabajo (hoy: Azure DevOps) que Axiom puede sincronizar de forma opt-in con el ciclo de vida de incrementos/bugs. Configuración cero (`kind: 'none'`) es el default seguro; activarlo es una elección explícita del proyecto. Ver [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md).

## Relacionado

- [02_Configuracion.md](02_Configuracion.md)
- [05_Incrementos.md](05_Incrementos.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
- [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md)
- [08_Glosario.md](../08_Glosario.md)
