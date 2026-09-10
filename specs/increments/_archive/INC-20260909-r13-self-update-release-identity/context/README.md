# Context

## Propósito

Esta carpeta reserva contexto auxiliar estrictamente vinculado a la identidad de release y al descubrimiento read-only del incremento `INC-20260909-r13-self-update-release-identity`. Su función es guardar evidencia técnica que facilite revalidación y revisión sin duplicar los requisitos canónicos de los documentos `01`–`04`.

En la fase documental actual no se necesita contexto adicional: los contratos y criterios están contenidos en el incremento y el procedimiento ejecutable en su plan asociado. La carpeta queda preparada, no obligatoriamente poblada.

## Qué puede vivir aquí

- Un mapa fechado de archivos y símbolos reales descubierto en la fase 0 del runtime, si resulta necesario para conservar trazabilidad.
- Notas breves sobre la convención existente de remote canónico, entrypoint global, metadata de build o almacenamiento user-level, respaldadas por rutas/commits.
- Matrices de compatibilidad SemVer/Git que complementen, sin redefinir, los criterios de aceptación.
- Evidencia estática pequeña y saneada que explique una decisión técnica no apropiada para receipts.
- Referencias a pruebas herméticas o fixtures del repositorio de código; no copias de su contenido.

Todo documento futuro debe indicar fecha, fuente, alcance y si describe estado observado o decisión aprobada.

## Qué no debe vivir aquí

- Metadata, status, enlaces estructurales, índices o transiciones de lifecycle gestionados por Core.
- Receipts, logs extensos, outputs completos de test, artefactos binarios, repos Git fixture o cachés de ejecución.
- Código del runtime, scripts de implementación o configuración del producto.
- Copias de `README.md`, requisitos, modelo, criterios o interacciones del incremento.
- Contratos del manifest v2 completo, diseño de Launcher, apply/rollback o CI final pertenecientes a incrementos 2–5.
- URLs con credenciales, tokens, rutas personales, información privada o cualquier secreto.
- Afirmaciones de que una capacidad está implementada sin evidencia de código y validación.

## Estructura sugerida

Solo si la implementación genera evidencia auxiliar con valor estable:

```text
context/
├── README.md
├── runtime-symbol-map.md       # rutas/símbolos confirmados y commit observado
├── git-readonly-notes.md       # comandos permitidos y evidencia de no mutación
└── compatibility-matrix.md     # casos SemVer/Git no obvios
```

No es un checklist obligatorio. Antes de crear un archivo debe comprobarse que la información no pertenece al plan, a un receipt o a la spec general. La estructura y cualquier incorporación futura deben respetar las operaciones estructurales de Axiom/Core.
