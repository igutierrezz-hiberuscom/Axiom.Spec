# Plantilla de README de plan

Esta plantilla corresponde al `README.md` raíz de un plan.

## Cabecera mínima

> **Código**: PLAN-YYYYMMDD-HHMMSSZ-HASH4  
> **Estado**: draft | approved | superseded | archived  
> **Artefacto origen**: incremento o bug  
> **Versión de spec**: vN  
> **Versión de plan**: pN

## Resumen ejecutivo

Explicar qué se va a ejecutar, sobre qué artefacto, con qué objetivo técnico y qué alcance tiene el plan.

## Objetivo técnico

- resultado técnico esperado;
- restricciones de arquitectura o delivery;
- decisiones que este plan debe respetar.

## Alcance incluido

- trabajo que sí entra en el plan;
- bounded contexts o superficies afectadas;
- dependencias relevantes.

## Alcance excluido

- trabajo que no entra;
- follow-ups previstos;
- incertidumbres todavía fuera del plan.

## Roles impactados

| Rol | Archivo | Estado | Notas |
|-----|---------|--------|-------|
| `role-slug` | `role-role-slug.md` | queued | |

Regla: cada rol impactado debe tener su propio archivo separado. El `README.md` no sustituye el detalle por rol.

## Estrategia E2E

- `none` si no aplica;
- `inline` si se cubre en el mismo flujo;
- `parallel` si el proyecto materializa un carril E2E independiente.

## Riesgos y dependencias

| ID | Severidad | Descripción | Mitigación | Rol responsable |
|----|-----------|-------------|------------|-----------------|
| RISK-001 | medium | | | |

## Cambios en contexto técnico

- documentos de `technical-context/` que deban actualizarse;
- razones para actualizar contexto y no solo el artefacto local.

## Consolidación y archivado

- cómo se integrarán los cambios en la spec general;
- cuándo el artefacto origen podrá pasar a `archive/`;
- qué evidencias o verificaciones faltan para cerrar.

## Validaciones y gates

- gates previos al arranque;
- gates previos al cierre;
- manifest, pruebas o evidencias requeridas.

## Fuentes y supuestos

- decisiones relevantes;
- documentación de origen;
- referencias externas opcionales;
- supuestos aún vigentes.