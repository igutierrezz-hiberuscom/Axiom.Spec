# 04 Interacciones UI

## Superficies

CLI `bindings` y `topology`, proyecciones launcher/MCP y mensajes de workspace/member install.

## Flujo

La superficie localiza autoridad, carga y valida topology, normaliza el path, valida repo ID, ejecuta writer bajo lock y proyecta el resultado. `show` informa estado físico sin mutar.

## Estados visibles

Errores de carga, schema, semántica, binding y persistencia permanecen diferenciados. En JSON no se imprime copy humana en stdout; en modo humano los diagnósticos van al canal correcto.

## Reactividad

Un cambio de authority invalida bindings desconocidos de forma visible; nunca los elimina automáticamente ni cae silenciosamente al ref.
