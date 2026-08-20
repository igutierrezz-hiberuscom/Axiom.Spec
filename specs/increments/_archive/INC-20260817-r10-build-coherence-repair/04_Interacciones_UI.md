# 04 Interacciones UI

## Objetivo del documento

Describir los efectos observables de la reparación de coherencia sobre CLI, configuración generada y proyección MCP. No se añadió una pantalla ni un flujo visual nuevo.

## Superficie UI afectada

- La CLI y sus APIs de inicialización/configuración aceptan sólo los ocho targets canónicos publicados.
- La documentación y los outputs generados presentan únicamente esos targets y sus archivos vigentes.
- Las configuraciones MCP nativas continúan siendo project-scoped; Visual Studio usa su generator vigente.

## Flujo de interacción

1. El usuario selecciona un target canónico.
2. La CLI compone e instala el perfil y genera únicamente sus archivos soportados.
3. Si se configura MCP, el filtro confirma proyecto, manifest y destino antes del writer.
4. Un identificador retirado no se anuncia ni se materializa como alternativa operativa.

## Estados visibles

Los ocho targets vigentes se comportan como las únicas opciones materializables. No se muestra LiteLLM ni `copilot-vscode`; una entrada no válida se rechaza por la validación pública correspondiente.

## Cascadas y comportamiento reactivo

No hay cascadas de UI. Las correcciones de registry, MCP y Visual Studio son internas y conservan la semántica existente de preview, no-clobber y aislamiento de proyecto.
