# Cambios de modelo: política de cobertura de capabilities en Doctor

`CC-004` deja de ser una comprobación booleana sobre el conjunto observado y
pasa a evaluar una matriz pequeña:

- catálogo canónico provider-routed;
- declaración en `capabilities.yaml`;
- clase de cumplimiento (`required`, `optional`, `postMvpOptional`);
- estado (`enabled`, `experimental`, `disabled`, `unavailable`);
- providers que declaran la capability.

La salida se clasifica de forma visible: las capabilities requeridas activas
sin provider son fallos; las opcionales o post-MVP sin provider son warnings;
las MCP-only quedan fuera de esta obligación; las deshabilitadas o no
 disponibles no generan un fallo activo por sí solas.
