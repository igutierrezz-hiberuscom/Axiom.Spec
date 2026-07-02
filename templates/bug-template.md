# Plantilla de README de bug

Esta plantilla corresponde al `README.md` raíz de un bug.

## Cabecera mínima

> **Código**: BUG-YYYYMMDD-HHMMSSZ-HASH4  
> **Estado**: Reportado | Especificado | Planificado | En Desarrollo | En Validación | Implementado | Integrado | Archivado  
> **Fecha de reporte**: YYYY-MM-DD  
> **Severidad**: low | medium | high | critical  
> **Prioridad**: low | medium | high | critical  
> **Referencias externas**: opcional  
> **Incremento relacionado**: opcional

## Resumen del defecto

Describir el problema observable, qué rompe y por qué merece quedar trazado como bug independiente.

## Contexto conocido

- módulo o capacidad afectada;
- actor o rol afectado;
- superficie visible afectada;
- frecuencia conocida;
- workaround si existe.

## Clasificación funcional

- validación;
- regresión;
- cálculo;
- permisos;
- datos;
- integración;
- UI.

## Comportamiento actual

Describir qué ocurre hoy y por qué es incorrecto.

## Comportamiento esperado

Describir el resultado funcional correcto.

## Reproducción

### Precondiciones

- condición inicial del escenario.

### Pasos

1. Paso 1.
2. Paso 2.
3. Paso 3.

### Resultado observado

- resultado real;
- mensajes, bloqueos o incoherencias visibles.

## Superficie de regresión

- zonas que no deben romperse al corregir el bug;
- escenarios vecinos que deben revisarse.

## Estructura mínima del bug

| Documento | Obligatoriedad | Uso |
|-----------|----------------|-----|
| `metadata.yml` | Obligatorio | Fuente de verdad estructural del artefacto |
| `01_Requisitos.md` | Condicional | Solo si el bug necesita separar reglas o requisitos afectados |
| `03_Criterios_Aceptacion.md` | Condicional | Solo si los criterios no caben de forma clara en el README |
| `02_Cambios_Modelo.md` | Condicional | Solo si hay cambio real de modelo, contrato, estado o estructura |
| `04_Interacciones_UI.md` | Condicional | Solo si hay UI relevante o comportamiento reactivo observable |
| `context/` | Opcional | Material añadido del artefacto |
| `artifacts/plans/` | Condicional | Aparece cuando exista al menos un plan |

## Material adicional

Referenciar aquí lo que se haya colocado en `context/`: pantallazos, exports, transcripciones, notas, documentos adjuntos o resúmenes.

## Trazabilidad y fuentes

- documentos canónicos afectados;
- incrementos o bugs relacionados;
- referencias externas opcionales;
- decisiones relevantes.

## Estado de validación humana

- responsable;
- fecha de última revisión;
- veredicto actual.