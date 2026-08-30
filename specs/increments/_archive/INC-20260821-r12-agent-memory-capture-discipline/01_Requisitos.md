# 01 Requisitos

## Objetivo del documento

Hacer explícita y verificable la responsabilidad de los agentes de capturar decisiones y bugs, sin automatismos de runtime.

## Requisitos del incremento

1. El runtime no expone una política que afirme que `decision` o `bug` se guardan automáticamente.
2. `axiom memory add` acepta opcionalmente `--visibility project-shared|private`, `--rationale <text>` y `--source <text>` y persiste esos valores en la `MemoryEntry`.
3. Una visibilidad distinta de `project-shared` o `private` falla con un error accionable; omitirla conserva la entrada local no exportable por Knowledge Sync.
4. `private` no restringe la lectura local; solo excluye la entrada del intercambio Git. `project-shared` es la única visibilidad exportable, siempre que pase los controles existentes de schema y secretos.
5. Las skills y agentes Kiro de incremento/bug exigen una llamada explícita a `axiom memory add` al confirmar una decisión o un bug, y comunican un fallo de guardado sin inventar fallback.
6. Las instrucciones distinguen contenido `project-shared` (confirmado, reutilizable, relevante al equipo) de `private` (contexto local, hipótesis o notas temporales), y prohíben secretos en ambos casos.

## Reglas de negocio relevantes

La memoria complementa la spec y no la sustituye. Capturar una decisión o bug no cambia lifecycle, permisos ni Git por sí mismo; Knowledge Sync sigue siendo el único mecanismo que puede compartir una entrada `project-shared`.

## Fuera de alcance funcional

No hay captura implícita desde audit/telemetría, no hay sincronización automática y no se cambian los backends de memoria.
