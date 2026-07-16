# 08. Implementación

Cómo se ejecuta el trabajo real de un rol contra un plan aprobado.

## Ciclo de vida (CLI)

```bash
axiom-role start   [--create-branch] [--commit] [--push] --confirm   # arranca el trabajo de ESTE rol
# ... implementación real en el repo de código del rol ...
axiom-role apply                                                       # aplica cambios intermedios / checkpoints de progreso
axiom-role complete [--no-review] [--force]                            # cierra el trabajo de este rol
```

`axiom-role` se ejecuta SIEMPRE desde el repo de código del rol correspondiente (nunca desde el repo de spec ni desde otro repo de rol — repo-affinity, ver más abajo).

## `start`: precondición de plan aprobado

`axiom-role start` exige que exista un plan APROBADO para este incremento/bug antes de arrancar (`checkPlanIsApproved`). Sin plan aprobado, el comando se rechaza.

### Efectos git opt-in

`axiom-role start` puede, opcionalmente, crear la rama de trabajo y commitear localmente: `--create-branch` crea la rama; `--commit` commitea LOCALMENTE; el push solo ocurre con `--push` explícito Y `--confirm`. Sin ninguno de estos flags, el comportamiento por defecto no incluye NINGÚN efecto git — el rol se ejecuta sobre el estado actual del repo, sin tocar ramas ni commits.

## `complete`: review de write-scope + verificación funcional

`axiom-role complete` corre dos gates antes de cerrar el rol:

1. **Review de write-scope**: valida el `git diff` real del repo de rol contra el `allowedWriteScope` que el plan aprobado declaró para ESE repo (ver [07_Planes.md](07_Planes.md)), y **bloquea la completion** ante cualquier violación (el estado sigue `in-progress`). `--no-review`/`--force` saltan este gate explícitamente. Detalle del contrato: [09_Revisiones.md](09_Revisiones.md).
2. **Verificación funcional**: descubre y ejecuta la validación propia del repo (test/build/lint/typecheck) y bloquea ante fallo, con el mismo orden de descubrimiento que usa este workspace (README → scripts de package → task runner → configs de build/test).

## Repo-affinity

`axiom-role` para el rol **X** debe ejecutarse SOLO desde el repo asignado a ese rol X — abrir el repo de OTRO rol y correr `axiom-role` se rechaza, nombrando el repo correcto. Es un NO-OP fuera de un workspace multi-repo con roles asignados (single-repo, o sin asignaciones, no se ve afectado).

## Desde el launcher

El launcher no reemplaza la implementación en sí (que sigue siendo trabajo real de código en el repo del rol), pero sí ofrece: el prompt pregenerado para el agente/adapter que va a implementar, el panel de doctor previo (ver [11_Launcher_Visual.md](11_Launcher_Visual.md)), y los paneles operador de rama de rol / commit-sync (preview → confirmar, git local-only, sin push por defecto).

## Relacionado

- [07_Planes.md](07_Planes.md)
- [09_Revisiones.md](09_Revisiones.md)
- [10_Archivado.md](10_Archivado.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
