# Requisitos: superficie MCP-only para `axiom.*`

1. El runtime debe distinguir los ids provider-routed del modelo genérico de
   los ids MCP-only del broker unificado.
2. `axiom.topologyRead`, `axiom.migrationManifestRead` y
   `axiom.adoptionStateRead` deben conservarse como ids MCP válidos.
3. Un provider registry sin esos tres ids no debe considerarse incompleto por
   esa ausencia específica.
4. Los perfiles y el routing genérico no deben presentar esos ids como una
   dependencia de provider tradicional.
5. La separación debe ser observable mediante tipos, constantes, validación o
   pruebas; no puede depender solo de comentarios.
