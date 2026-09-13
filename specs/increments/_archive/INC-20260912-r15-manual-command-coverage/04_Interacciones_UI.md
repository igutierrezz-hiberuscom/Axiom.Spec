# 04 Interacciones UI

## Objetivo del documento

Describir cómo se consume el manual resultante. No hay cambios en CLI ni en el launcher.

## Recorrido de una persona que no conoce Axiom

1. Entra por `Axiom/docs/README.md`, elige la sección según lo que quiere hacer.
2. Abre `docs/cli/README.md` y encuentra los comandos agrupados por propósito: ciclo básico de proyecto, ciclo SDD, operación de operador, integraciones y contexto.
3. Abre la página del comando y obtiene, en ese orden, para qué sirve, cuándo usarlo, cómo se escribe, qué toca en disco, qué puede bloquearlo, qué verá al terminar y a qué comando ir después.

## Recorrido de un agente

Un agente que necesite operar Axiom sin leer el código encuentra en la misma página el contrato completo del comando: opciones reales, ficheros afectados, gates y salidas. La página no asume conocimiento previo del repositorio ni remite al código fuente para entender el contrato.

## Señal cuando falta documentación

Si alguien añade un comando y no lo documenta, la suite falla con un mensaje que nombra el comando descubierto. La ausencia de documentación deja de ser invisible.

## Sin interacciones de CLI ni de launcher

Este incremento no registra, modifica ni retira comandos y no altera pantallas del launcher.
