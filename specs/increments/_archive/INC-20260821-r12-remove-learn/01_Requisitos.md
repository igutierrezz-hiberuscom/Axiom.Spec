# 01 Requisitos

## Objetivo del documento

Definir la retirada completa y segura de la funcionalidad de aprendizaje continuo.

## Requisitos del incremento

1. La CLI no registra ni documenta el comando público `learn` ni sus subcomandos.
2. No queda código ejecutable que convierta texto explícito o eventos de auditoría en `LessonCandidate` o en entradas con etiqueta `lesson`.
3. `axiom context status` conserva su contrato principal, pero no consulta ni presenta lecciones recientes.
4. La memoria general y el audit trail continúan disponibles por sus rutas propias.
5. La documentación operativa activa no recomienda invocar `axiom learn`.

## Reglas de negocio relevantes

- El audit trail es trazabilidad, no un origen de aprendizaje automático.
- La ausencia de candidatos o de `audit.log` deja de tener una semántica específica de lecciones porque la superficie se retira.
- Los datos históricos no se borran automáticamente ni se reinterpretan.

## Fuera de alcance funcional

- Migrar o eliminar entradas históricas con etiqueta `lesson`.
- Crear un mecanismo alternativo de aprendizaje, resúmenes de sesión o captura automática.
- Cambiar la política de retención del audit trail.
