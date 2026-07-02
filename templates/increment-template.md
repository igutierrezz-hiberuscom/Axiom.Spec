# Plantilla de README de incremento

Esta plantilla corresponde al `README.md` raíz de un incremento.

## Cabecera mínima

> **Código**: INC-YYYYMMDD-HHMMSSZ-HASH4  
> **Estado**: En Borrador | Especificado | Planificado | En Desarrollo | En Validación | Implementado | Integrado | Archivado  
> **Fecha de creación**: YYYY-MM-DD  
> **Tipo de cambio**: Nueva capacidad | Modificación | Migración | Refactor funcional  
> **Referencias externas**: opcional  
> **RF afectados**: referencias a `01_Requisitos_Funcionales.md` cuando aplique

## Resumen

Explicar en pocas líneas qué cambia, por qué existe el incremento y qué resultado funcional aporta.

## Contexto y motivación

- necesidad de negocio;
- situación actual;
- límites conocidos;
- relación con decisiones o incrementos previos.

## Alcance

### Incluido

- comportamiento funcional que sí entra en este incremento;
- superficies afectadas;
- reglas observables que se añaden o modifican.

### Excluido

- lo que queda fuera;
- workarounds, migraciones o follow-ups no cubiertos;
- decisiones que requieren otro incremento.

## Documentos del incremento

| Documento | Obligatoriedad | Uso |
|-----------|----------------|-----|
| `metadata.yml` | Obligatorio | Fuente de verdad estructural del artefacto |
| `01_Requisitos.md` | Obligatorio | Requisitos funcionales del incremento |
| `03_Criterios_Aceptacion.md` | Obligatorio | Criterios observables y verificables |
| `02_Cambios_Modelo.md` | Condicional | Solo si hay cambio de modelo, contrato, estado o estructura |
| `04_Interacciones_UI.md` | Condicional | Solo si hay UI relevante o comportamiento reactivo observable |
| `context/` | Opcional | Material añadido del artefacto |
| `artifacts/plans/` | Condicional | Aparece cuando exista al menos un plan |

## Dudas abiertas

- preguntas aún no resueltas;
- si bloquean o no;
- qué falta para cerrarlas.

## Decisiones funcionales cerradas

| ID | Decisión | Resultado |
|----|----------|-----------|
| D-001 | | |

## Consolidación en la spec general

- documentos canónicos que este incremento deberá actualizar al integrarse;
- impacto esperado sobre RF, RNF o documentos base;
- criterio de cierre documental.

## Estrategia E2E

- `none` si no aplica;
- `inline` si el flujo E2E queda embebido;
- `parallel` si se materializa artefacto E2E con vida propia.

## Trazabilidad y fuentes

- documentos de spec general;
- decisiones relevantes;
- referencias externas opcionales;
- material adicional en `context/`.

## Estado de validación humana

- responsable;
- fecha de última revisión;
- veredicto actual.