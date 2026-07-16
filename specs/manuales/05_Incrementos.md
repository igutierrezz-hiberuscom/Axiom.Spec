# 05. Incrementos

El ciclo de vida completo de un incremento, desde la creación hasta el archivado.

## Dónde vive

Cada incremento es una carpeta propia: `<specPath>/increments/<INC-id>/`, con su `metadata.yml` (identidad de instancia). El `INC-id` se genera por sistema, no como texto libre.

## Ciclo de vida (CLI)

```bash
axiom-increment create --title "..."          # crea la carpeta + metadata.yml
axiom-increment refine   / specify             # actualiza metadata.yml (goal/scope/non-goals/acceptance criteria)
axiom-increment change                          # ajustes posteriores al refinado
axiom-plan create / approve                     # ver 07_Planes.md — un incremento no se implementa sin un plan aprobado
axiom-role start / apply / complete             # ver 08_Implementacion.md — ejecución por rol contra el plan
axiom-increment verify                          # descubre y CORRE la validación funcional del repo destino; bloquea ante fallo
axiom-increment archive                         # transición terminal — mueve la carpeta a _archive/ (ver 10_Archivado.md)
```

El estado (`status: WorkflowState`, vocabulario de 9 valores) es dirigido por una máquina de estados; `link-plan`/`link-increment` establecen relaciones entre artefactos.

### `verify` corre validación real

`axiom-increment verify` no es solo una transición de estado: descubre y ejecuta la validación propia del repo destino (test/build/lint/typecheck), siguiendo el mismo orden de descubrimiento que este propio documento describe en la sección de Validación de cada incremento (README → scripts de package → task runner → configs de test/build). Si no se descubre ningún comando, emite la sentencia best-effort estándar y trata el paso como no bloqueante. Bloquea la transición ante un fallo real; `--no-verify`/`--force` saltan el gate, `--preview`/`--dry-run` reporta lo descubierto sin ejecutar.

## Reglas de cierre

Un incremento solo puede marcarse `closed` si:

- el goal es claro;
- existen acceptance criteria;
- se implementó (o hay justificación explícita de no-code);
- se corrió la validación disponible;
- se revisó contra el intent original y los acceptance criteria (ver [09_Revisiones.md](09_Revisiones.md));
- se integró el conocimiento estable en `Axiom.Spec/specs/00_*.md`…`08_*.md` cuando aplica.

Si falta algo, el incremento queda `Status: pending` con el motivo explícito — nunca se fuerza a `closed`.

## Repo-affinity

En un workspace multi-repo con roles asignados, `axiom-increment` debe ejecutarse desde el **repo de SPEC** — ejecutarlo desde el repo de control o un repo de rol se rechaza con un mensaje que nombra el repo correcto. Esto es un NO-OP fuera de ese escenario (single-repo, o sin asignaciones de rol).

## Desde el launcher

La pestaña **Crear** del launcher (`axiom app`) ofrece un asistente guiado de 3 pasos: "¿qué querés hacer?" → familia (`increment`, entre otras) → acción concreta → formulario dinámico con solo los campos necesarios. Al completarlo se genera un **prompt pregenerado** para el adapter seleccionado (copiable con un clic), y opcionalmente se puede **ejecutar** (scaffold real de la estructura vía las mismas funciones de este manual), siempre bajo el patrón preview → confirmar. Si el plugin de Azure DevOps está configurado, tras una creación exitosa aparece una tarjeta que ofrece crear el User Story correspondiente con un clic — ver [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md). Detalle completo del front: [11_Launcher_Visual.md](11_Launcher_Visual.md).

## Relacionado

- [06_Bugs.md](06_Bugs.md)
- [07_Planes.md](07_Planes.md)
- [08_Implementacion.md](08_Implementacion.md)
- [09_Revisiones.md](09_Revisiones.md)
- [10_Archivado.md](10_Archivado.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
- [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md)
