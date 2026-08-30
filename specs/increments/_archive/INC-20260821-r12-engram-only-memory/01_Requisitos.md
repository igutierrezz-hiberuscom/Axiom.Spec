# 01 Requisitos

## Objetivo del documento

Establecer Engram como dependencia operativa única de memoria y hacer observables sus errores.

## Requisitos del incremento

1. Toda lectura, consulta, escritura y session summary de memoria usa el backend Engram local vinculado al proyecto.
2. El runtime no expone ni selecciona un backend JSON local, `forceJson` ni otro fallback de persistencia.
3. La indisponibilidad de ejecutable, spawn, handshake o tool de Engram se devuelve al caller como un `MemoryError` tipado; el caller comunica error y no simula datos vacíos ni éxito.
4. Ninguna operación de memoria crea, lee o reescribe archivos JSON project-scoped.
5. Doctor incluye un check de memoria que falla con remediación clara cuando Engram no está disponible en PATH; el check pasa cuando el ejecutable responde a su comprobación local segura.
6. El aislamiento por `projectId` y el pin `--project=<projectId>` del backend Engram se preservan.

## Reglas de negocio relevantes

La memoria sigue siendo auxiliar frente a la spec canónica. Un JSON local histórico puede permanecer en disco, pero no participa en ninguna lectura o escritura del runtime.

## Fuera de alcance funcional

No hay instalación automática, red remota, migración/borrado de datos locales ni degradación a archivos JSON.
