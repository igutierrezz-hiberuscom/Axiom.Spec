# 07. Planes

Cómo se planifica el trabajo de un incremento o bug antes de tocar código, por rol.

## Qué es un plan

Un plan (`axiom-plan`) traduce un incremento/bug refinado en un conjunto de tareas por rol (back/front/qa-e2e, o roles custom), con un `allowedWriteScope` explícito por repo: qué paths, en qué repos, ese plan tiene permitido mutar. Es el contrato que luego valida el review de write-scope (ver [09_Revisiones.md](09_Revisiones.md)).

## Ciclo de vida (CLI)

```bash
axiom-plan create --title "..." --link-increment <INC-id>   # crea el plan y lo vincula al incremento/bug
axiom-plan approve                                             # aprueba el plan — desbloquea la implementación
```

Un plan (como incremento/bug) vive en `<specPath>/plans/<PLAN-id>/`, con su propio `metadata.yml` y estado dirigido por la misma máquina de estados de workflow. `axiom-role start` exige un plan APROBADO antes de arrancar el trabajo de ese rol (`checkPlanIsApproved`).

## Planes por rol

Un plan puede declarar tareas diferenciadas para cada rol (back/front/e2e u otros roles custom del proyecto), cada una con su propio `allowedWriteScope` de repos/paths. Esto es lo que permite que `axiom-role complete` valide el diff real del repo de ESE rol contra SOLO el scope que le corresponde a él — ver [08_Implementacion.md](08_Implementacion.md) y [09_Revisiones.md](09_Revisiones.md).

## La carpeta `artifacts/plans/`

Por convención, los artefactos de planificación más informales (notas de diseño, desgloses previos a formalizar un `axiom-plan`) pueden vivir en una carpeta `artifacts/plans/` dentro del repo correspondiente, sin forzar la disciplina completa de `metadata.yml` para borradores de trabajo. El plan FORMAL y canónico, sin embargo, siempre es el creado vía `axiom-plan create` en `<specPath>/plans/`.

## Repo-affinity

Igual que incrementos/bugs: `axiom-plan` debe ejecutarse desde el **repo de SPEC** en un workspace multi-repo con roles asignados.

## Desde el launcher

La familia `plan` está disponible en el flujo guiado de 3 pasos del launcher (Crear → familia `plan` → acción `plan-new`/`plan-approve`/`plan-execute`), con el mismo patrón preview → confirmar antes de mutar. Ver [11_Launcher_Visual.md](11_Launcher_Visual.md).

## Relacionado

- [05_Incrementos.md](05_Incrementos.md)
- [06_Bugs.md](06_Bugs.md)
- [08_Implementacion.md](08_Implementacion.md)
- [09_Revisiones.md](09_Revisiones.md)
- [11_Launcher_Visual.md](11_Launcher_Visual.md)
