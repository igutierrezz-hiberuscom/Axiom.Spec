# 01 Requisitos

## Objetivo del documento

Fijar qué debe comprobar la verificación documental de cierre y cómo debe comportarse.

## Requisitos del incremento

### RQ-088-01 Alcance documental completo
La verificación cubre especificación canónica numerada, manual único de producto en `Axiom/docs/**`, contexto técnico, decisiones, planes y README de runtime y de paquetes. Ningún tipo queda fuera por omisión.

### RQ-088-02 Alcance determinado, no total
La verificación determina qué documentación entra en el alcance de un cambio concreto según criterio declarado, para no exigir revisar todo en cada cierre ni dejar fuera lo afectado. El criterio queda documentado y es el mismo para CLI, launcher y MCP.

### RQ-088-03 Salida auditable
La verificación produce, por cierre, la lista de documentos en alcance con su resultado: actualizado, sin cambios con motivo, o no revisado. La salida es legible por una persona y consultable después.

### RQ-088-04 Comportamiento accionable
Ante documentación en alcance no revisada, el mensaje nombra el documento y la razón por la que entró en el alcance, e indica qué hacer. No se emiten avisos genéricos.

### RQ-088-05 Un solo camino de cierre
La verificación se integra en el ejecutor de transiciones gobernado que ya usan CLI, launcher y MCP. No se crea un comando de archivado alternativo ni una ruta que la esquive.

### RQ-088-06 Progresión declarada
Existe una progresión explícita entre avisar y bloquear, con la condición de cada modo declarada. La entrada en servicio no puede dejar bloqueado el trabajo en vuelo.

### RQ-088-07 Sin puertas traseras
La verificación no habilita edición manual de estado, no vuelve legal ninguna transición ilegal y no permite saltarse gates de aprobación existentes.

### RQ-088-08 Sin redacción automática
La verificación comprueba cobertura documental. No genera ni edita contenido de documentación.

## Reglas de negocio relevantes

- El contrato de QA y la semántica de archive los fijan `ACC-041`, `ACC-043` y `ACC-045`; este incremento los consume.
- La verificación no sustituye la revisión independiente ni la validación humana.
- Un cambio que legítimamente no afecta a documentación debe poder cerrarse sin fricción, dejando constancia de ello.

## Fuera de alcance funcional

- Escribir o corregir documentación.
- Cambiar la semántica de archive o el contrato de QA.
- Verificar calidad de redacción o estilo.
