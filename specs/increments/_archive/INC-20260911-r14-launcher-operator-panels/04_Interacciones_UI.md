# 04 Interacciones UI

## Objetivo del documento

Describir la interacción en el frontend del launcher para Model Routing y Actualización.

## Flujo de Model Routing

1. El usuario navega a la pestaña **Model Routing**.
2. Observa la tabla de slots: slot, clase recomendada, override activo, estado efectivo y nivel de soporte.
3. Formulario para asignar override: selecciona slot y clase de modelo (`cheap`, `medium`, `strong`, `local`), pulsa "Previsualizar" y luego "Asignar modelo".
4. Botón "Validar configuración": ejecuta diagnósticos y muestra resultados en pantalla.

## Flujo de Actualización

1. El usuario navega a la pestaña **Actualización**.
2. Observa tarjeta de estado: versión actual, publicada, relación.
3. Botón "Buscar actualizaciones": dispara `check`.
4. Botón "Planificar actualización": dispara `plan` con política de reinicio.
5. Botón "Aplicar actualización": confirma y ejecuta el plan. Muestra barra/indicador de progreso y fases.
