# Criterios de aceptación

- [x] La skill Copilot tiene frontmatter válido y el nombre coincide con su carpeta.
- [x] `axiom-autopilot` es un custom agent invocable y referencia la skill y los workers permitidos.
- [x] `axiom-increment` y `axiom-review` tienen límites de edición y delegación explícitos.
- [x] Existe el prompt `/axiom-autopilot` y delega en el agente principal.
- [x] El playbook exige reconciliar, modificar o eliminar claims activos obsoletos; `SUPERSEDE` aislado no es suficiente.
- [x] El playbook exige revisar `Axiom.Spec/context/**` y declarar el resultado cuando no haya cambios.
- [x] Las rutas del prompt de incremento apuntan a `Axiom.Spec/specs/increments/`.
- [x] Frontmatter, diagnósticos Markdown, rutas y `git diff --check` fueron validados.
- [x] La spec general se actualizó en `04_Flujos_SDD_y_Ciclo_de_Vida.md` y `08_Glosario.md`.
- [x] El contexto técnico no se modificó porque no cambió `Axiom/`.
- [x] La carpeta fue archivada sin ejecutar comandos Git mutantes.
