# 06. Bugs

El ciclo de vida de un bug, con la disciplina "expected-behavior-first".

## Dónde vive

Cada bug es una carpeta propia: `<specPath>/bugs/<BUG-id>/`, con su `metadata.yml`, análoga a la de un incremento.

## Disciplina previa: clarificar el comportamiento esperado

Antes de tocar código, un bug debe dejar explícito el **comportamiento esperado** (no solo "qué está fallando"). Esta es la diferencia principal frente a un incremento: un incremento describe una capacidad nueva; un bug describe una discrepancia entre comportamiento actual y esperado, y esa discrepancia debe quedar clara en la spec del bug antes de implementar el fix.

## Ciclo de vida (CLI)

```bash
axiom-bug create --title "..."         # crea la carpeta + metadata.yml
axiom-bug refine / specify              # documenta expected vs actual behavior, goal, acceptance criteria
axiom-plan create / approve             # cuando el fix requiere coordinar múltiples repos/roles (ver 07_Planes.md)
axiom-role start / apply / complete     # implementación del fix (ver 08_Implementacion.md)
axiom-bug verify                        # corre la validación funcional del repo destino; bloquea ante fallo
axiom-bug archive                       # transición terminal — mueve la carpeta a _archive/ (ver 10_Archivado.md)
```

## Reglas de cierre

Igual que un incremento: un bug solo se marca `closed` cuando el comportamiento esperado quedó claro, hay acceptance criteria, el fix está implementado (o hay justificación no-code explícita), se corrió la validación disponible, se revisó contra el comportamiento esperado, y se integró conocimiento estable en la spec cuando aplica. Si algo falta, queda `Status: pending` con el motivo.

## Repo-affinity

Igual regla que los incrementos: `axiom-bug` debe ejecutarse desde el **repo de SPEC** en un workspace multi-repo con roles asignados; ejecutarlo desde otro repo se rechaza nombrando el repo correcto.

## Desde el launcher

La misma pestaña **Crear** del launcher cubre la familia `bug`: formulario dinámico (con el campo de comportamiento esperado como parte del flujo guiado), prompt pregenerado por adapter, y ejecución opcional confirm-gated. Si el plugin de Azure DevOps está configurado, un bug creado con éxito ofrece crear un work item tipo **Bug** con un clic — ver [12_Plugin_Azure_DevOps.md](12_Plugin_Azure_DevOps.md).

## Relacionado

- [05_Incrementos.md](05_Incrementos.md)
- [07_Planes.md](07_Planes.md)
- [08_Implementacion.md](08_Implementacion.md)
- [09_Revisiones.md](09_Revisiones.md)
- [10_Archivado.md](10_Archivado.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
