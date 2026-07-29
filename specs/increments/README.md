# Increments

Cada incremento de Axiom debe vivir en su propia carpeta.

## Baseline mínima

1. `README.md`
2. `metadata.yml`
3. `01_Requisitos.md`
4. `03_Criterios_Aceptacion.md`

## Archivos opcionales

1. `02_Cambios_Modelo.md`
2. `04_Interacciones_UI.md`
3. `context/`
4. planes asociados por rol o por fase cuando aplique.

## Archivado

Cuando un incremento se cierra, su carpeta se mueve físicamente a
`increments/_archive/<ID>/`. No se mantiene una copia activa ni se elimina
el registro histórico.