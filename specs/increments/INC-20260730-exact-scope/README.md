# INC-20260730-exact-scope: Type Check Recovery de Entradas

## Goal
Limitar el surface area de los parsers del CLI y del orquestador instalando un validador de esquemas (como Zod) para validar todo input.

## Scope
- Instalar `zod` en los packages necesarios (e.g. `@axiom/orchestrator`, `apps/cli`).
- Crear esquemas exactos para las configuraciones y parseos clave.
- Implementar validación fail-closed: si el input falla la validación, el proceso se interrumpe y se reporta el error, sin intentar "recuperar" estados inválidos.

## Non-goals
- No se migrarán todos los parsers existentes al instante, solo las entradas críticas (e.g. argumentos CLI, config yaml).

## Acceptance Criteria
- La validación utiliza un esquema exacto (Zod).
- Fallas en el parseo resultan en un error claro y salida temprana (fail-closed).
