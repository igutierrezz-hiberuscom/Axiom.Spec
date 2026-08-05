# Criterios de aceptación

- [x] `ACC-001` Registry enriquecido desde metadata, paths, workflow-state y
	topology reales; mantiene las claves basicas y omite relaciones no
	resolubles.
- [x] `ACC-002` Acciones de increment, bug, plan, role y QA conservadas, con
	validate transition, validate changes y archive/integrate donde existe
	wrapper canonico.
- [x] `ACC-003` Preview sin writes y confirmacion explicita para toda mutacion;
	validaciones read-only no mutan y reportan su `exitCode`.
- [x] `ACC-004` Cada ejecucion confirmada delega en el `run*` correspondiente y
	devuelve comando, mensaje, `exitCode`, estados y gates disponibles.
- [x] `ACC-005` Archive respeta transicion terminal, aprobacion/plan, QA,
	write-scope cuando se valida, `runIntegrate` y el resultado real del move.
- [x] `ACC-006` La UI presenta solo relaciones de spec/plan/role/repo/
	implementation respaldadas por una fuente identificable.
- [x] `ACC-007` Fixture create -> plan -> role -> validate -> archive o gate
	bloqueante documentado, sin estados falsos.
- [x] `ACC-008` Build, pruebas focalizadas y E2E disponible ejecutados; fallos
	preexistentes clasificados.
- [x] Freeze final, receipt `verify`, review e integracion canonica quedan
      documentados; la carpeta queda lista para archivado.
