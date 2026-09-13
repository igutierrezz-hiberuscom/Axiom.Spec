# 01 Requisitos

## Objetivo del documento

Fijar qué debe cumplirse para que exista una sola fuente de plantillas sin perder contenido ni romper la generación de superficies.

## Requisitos del incremento

### RQ-090-01 Fuente única declarada
`Axiom/axiom.spec/templates/` es la fuente única de plantillas. El contrato lo declara así en documentación activa y en los comentarios de código que hoy remiten a `Axiom.Spec/templates/*` como canónica.

### RQ-090-02 Rescate antes de retirar
Antes de eliminar la copia canónica se compara archivo por archivo. Cualquier contenido útil presente solo allí se incorpora a la fuente única. El resultado por archivo queda registrado con la elección tomada y su motivo.

### RQ-090-03 Vocabulario vigente
La fuente única no contiene vocabulario retirado en R-04 (perfiles y capas de política seleccionables, `axiom-gateway`, `generated-snapshots`, `product-owner`, `enterprise`) salvo dentro de notas marcadas explícitamente como históricas.

### RQ-090-04 Copia canónica retirada y estructura reconciliada
`Axiom.Spec/templates/` deja de existir y `Axiom.Spec/README.md` refleja la estructura resultante sin declarar plantillas que ya no aloja.

### RQ-090-05 Test dorado con alcance real
El test que protege las copias bundleadas compara la fuente única completa, no un subconjunto de títulos de sección. Una divergencia en cualquier plantilla cubierta hace fallar la prueba.

### RQ-090-06 Generación intacta
Los generadores de `opencode`, `claude-code`, `codex` y `antigravity` siguen resolviendo la plantilla con la precedencia present-file-wins sobre la copia bundleada, y su comportamiento observable no cambia.

### RQ-090-07 Sin retiradas por sospecha
Ninguna plantilla se elimina por no tener consumidor conocido en código. Las seis identificadas se conservan y su revisión queda aplazada.

## Reglas de negocio relevantes

- Se elige por actualidad de contenido, no por ubicación declarada: la copia del runtime gana porque es la vigente.
- Ninguna eliminación puede perder contenido no rescatado.
- La coherencia de nombres entre `axiom.spec/`, `<proyecto>.axiom` y `<proyecto>.spec` queda fuera: es una duda registrada, no un requisito de este incremento.

## Fuera de alcance funcional

- Retirada de plantillas sin consumidor.
- Renombrado de carpetas.
- Cambios en la materialización de skills y agentes.
