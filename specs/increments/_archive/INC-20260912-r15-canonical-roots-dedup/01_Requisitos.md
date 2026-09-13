# 01 Requisitos

## Objetivo del documento

Fijar qué debe cumplirse para tener un formato único de decisiones y unas raíces limpias, sin perder historia ni romper referencias.

## Requisitos del incremento

### RQ-094-01 Formato único de decisiones
Existe un solo formato y una sola raíz de decisiones en el repositorio canónico. La opción elegida entre migrar los nueve documentos numerados a formato gestionado o congelarlos como históricos queda registrada con su motivo antes de ejecutarse.

### RQ-094-02 Numeración sin colisiones
No quedan dos artefactos de decisión distintos con el mismo número. La resolución del choque 0032 conserva la trazabilidad de ambos documentos y no invalida referencias activas.

### RQ-094-03 Raíces legacy retiradas
Las raíces `increments/` y `bugs/` de primer nivel dejan de existir, y con ellas la copia suelta `INC-20260817-r10-acc038-lifecycle-docs` y los dos READMEs legacy de bugs, previa comprobación de que su contenido no aporta información ausente en los artefactos vigentes o archivados.

### RQ-094-04 Carpeta vacía eliminada
`Axiom.Spec/axiom.spec/` no existe. Si alguna ejecución la recrea por caer en la proyección por defecto, se registra como hallazgo y no se convierte en excusa para conservarla.

### RQ-094-05 Raíces activas intactas
`specs/increments/`, `specs/bugs/` y `specs/archive/` siguen existiendo y operativas. Ninguna se retira por estar vacía.

### RQ-094-06 Gobierno respetado
Todo cambio de estado, identificador o ubicación de un artefacto gestionado se ejecuta con comandos de Axiom. No se editan `metadata.yml`, índices ni receipts a mano, y no se borra material con receipts sin decisión explícita registrada.

### RQ-094-07 Estructura declarada reconciliada
`Axiom.Spec/README.md` describe la estructura resultante, sin afirmar que una raíz está vacía cuando no lo está ni listar carpetas que ya no existen.

### RQ-094-08 Frontera declarada con R-07
El incremento declara explícitamente qué cubre y qué deja a `ACC-035` y `ACC-037`, que se ocupan de los residuos bajo `specs/increments/INC-20260809-*`.

## Reglas de negocio relevantes

- Ninguna limpieza puede perder historia verificable: si un artefacto tiene receipts, no se borra sin decisión explícita.
- Las carpetas que el producto usa como raíces activas no se tratan como residuo.
- Los residuos que el archivado deja en la raíz activa no se limpian aquí: están registrados como duda con evidencia y su causa no está establecida.

## Fuera de alcance funcional

- Corregir el comportamiento del archivado.
- Residuos bajo `specs/increments/INC-20260809-*`.
- Cualquier cambio en el runtime.
