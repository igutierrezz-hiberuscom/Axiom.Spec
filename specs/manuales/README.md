# Manuales de Axiom

Guías prácticas de operación para un equipo que acaba de tener Axiom instalado en sus repos. A diferencia de `specs/00_*.md`…`08_*.md` (la spec funcional/técnica completa del producto), estos manuales están escritos para responder "¿qué es esto y cómo lo uso hoy?" en un lenguaje directo, orientado a tareas.

Estos manuales están curados para ESTA instalación concreta (el workspace `Axiom.SDD` + `Axiom.Spec` + `Axiom`, con el plugin opcional de **Azure DevOps** ya configurado como tracker). Si tu instalación no tiene ese plugin, todo lo demás sigue aplicando igual; simplemente ignora [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md).

## Índice

### Conceptos y arranque

- [01. Qué es cada cosa](01_Que_Es_Cada_Cosa.md) — visión general: qué es Axiom, el ciclo SDD ligero, roles de repo, la spec canónica, perfiles, adapters, el launcher, el doctor.
- [02. Configuración](02_Configuracion.md) — `axiom.yaml`, el registro de usuario, el estado runtime, perfiles/overlays/targets, roles, tuning de agente por adapter.
- [03. Actualizar versiones](03_Actualizar_Versiones.md) — `axiom upgrade`, checkpoints y rollback, `axiom self-update`.
- [04. Generar spec y contexto técnico](04_Generar_Spec_y_Contexto_Tecnico.md) — adoptar código existente, la estructura 00–08, indexado de contexto técnico.

### Ciclo de vida de trabajo

- [05. Incrementos](05_Incrementos.md) — crear, refinar, planificar, verificar y archivar un incremento; también desde el launcher.
- [06. Bugs](06_Bugs.md) — el ciclo de vida de un bug, desde el expected-behavior hasta el archivado.
- [07. Planes](07_Planes.md) — planes por rol, aprobación, write-scope.
- [08. Implementación](08_Implementacion.md) — ejecución de rol (`start` → `apply` → `complete`).
- [09. Revisiones](09_Revisiones.md) — review de write-scope, `axiom-review`, el ledger de hallazgos.
- [10. Archivado](10_Archivado.md) — la transición terminal, reglas de cierre, integración a 00–08.

### Superficies instaladas

- [13. Skills, agentes y roles instalados](13_Skills_Agentes_y_Roles.md) — qué instala Axiom por rol de repo (`sdd`/`spec`/`code`), las disciplinas transversales reutilizables, el catálogo de agentes, y el canal de inyección por proyecto para añadir profundidad específica de tu stack sin tocar el producto.

### Peculiaridades de esta instalación

- [11. Launcher visual](11_Launcher_Visual.md) — `axiom app`: el front de navegador para operar Axiom sin terminal ni TUI (doctor gate, onboarding, adapters, registro).
- [12. Plugin de Azure DevOps](12_Plugin_Azure_DevOps.md) — el tracker instalado: qué completar al crear un incremento/bug y dónde va la API key/PAT.

## Relacionado

- [05_Interfaces_Operativas.md](../05_Interfaces_Operativas.md) — detalle exhaustivo de CLI/TUI/adapters/launcher.
- [08_Glosario.md](../08_Glosario.md) — definiciones canónicas de todos los términos usados aquí.
