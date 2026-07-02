# Plantilla de spec general canónica

Esta plantilla define la forma objetivo de la spec general bajo `specs/`, manteniendo compatibilidad inicial con el estilo KVP.

## Documentos obligatorios de la spec general

| Documento | Obligatorio | Propósito |
|-----------|-------------|-----------|
| `specs/README.md` | Sí | Índice navegable de la spec general |
| `00_Resumen_Ejecutivo.md` | Sí | Visión, alcance y objetivos del producto |
| `00_Glosario.md` | Sí | Terminología canónica del dominio |
| `01_Requisitos_Funcionales.md` | Sí | RF trazables y auditables |
| `02_Requisitos_No_Funcionales.md` | Sí | NFR, restricciones y políticas transversales |

## Documentos condicionales de la spec general

| Documento | Se crea cuando... |
|-----------|-------------------|
| `03_Modelo_Dominio_o_Datos.md` | Hay modelo de dominio, datos, contratos o estados que deban fijarse |
| `04_Flujos_Negocio.md` | Existen flujos funcionales o procesos multi-paso relevantes |
| `05_Interfaces_Usuario.md` | El producto tiene superficie UI relevante |
| `06_Integraciones.md` | Existen integraciones internas o externas relevantes |
| `07_Seguridad.md` | Seguridad, permisos, aislamiento o auditoría requieren documento propio |

## Reglas de diseño

- `specs/` contiene la spec general canónica y las familias de artefactos.
- `specs/increments/` y `specs/bugs/` contienen work items activos y su `archive/`.
- Los planes cuelgan del artefacto que planifican, salvo planes excepcionales fuera de flujo normal.
- Los registros globales de incrementos o bugs no son fuente de verdad: se derivan desde `metadata.yml`.
- La fuente de verdad de la evolución documental vive en metadata y Git, no en `CHANGELOG.md` por artefacto.

## Relación con otras raíces del repo

- `technical-context/` conserva conocimiento técnico estable y obligatorio.
- `context/` conserva material añadido de apoyo a la spec general cuando haga falta.
- `artifacts/` solo aloja artefactos transversales o planes fuera de flujo normal.