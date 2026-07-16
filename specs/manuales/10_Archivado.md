# 10. Archivado

La transición terminal de un incremento o bug, y cómo integrar lo aprendido a la spec canónica.

## La transición terminal

`axiom-increment archive` / `axiom-bug archive` es la última transición del ciclo de vida. No solo escribe `status: archived` en `metadata.yml`: al completar la transición, **mueve físicamente** (rename atómico) la carpeta de la instancia de `<specPath>/{increments,bugs}/<ID>/` a `<specPath>/{increments,bugs}/_archive/<ID>/`. Nunca sobreescribe — si ya existe una carpeta archivada con el mismo ID, la operación falla con un mensaje claro en vez de clobberear.

Un artefacto archivado deja de aparecer en `axiom-increment list` / `axiom-bug list` por defecto (esos listados solo escanean el nivel directo del `<kindFolder>/`, no `_archive/`) — es el comportamiento esperado, consistente con la convención `_archive/` que ya usa este propio repo de spec (`specs/increments/_archive/`).

## Reglas de cierre (antes de archivar)

El archivado es el ÚLTIMO paso, después de que TODAS las reglas de cierre ya se cumplieron (ver [05_Incrementos.md](05_Incrementos.md) / [06_Bugs.md](06_Bugs.md)):

- goal o comportamiento esperado claro;
- acceptance criteria existentes;
- cambios implementados o justificación no-code explícita;
- validación disponible ejecutada;
- revisión contra el intent/acceptance criteria completada (ver [09_Revisiones.md](09_Revisiones.md));
- conocimiento estable integrado en la spec general, cuando aplica.

Si algo falta, el artefacto se deja `Status: pending` — nunca se archiva un artefacto que no está realmente cerrado.

## Integrar conocimiento estable en 00–08

Al cerrar, el paso final es actualizar `Axiom.Spec/specs/00_*.md`…`08_*.md` con el conocimiento estable y reutilizable que dejó el incremento/bug (decisiones de arquitectura, nuevos comandos, nuevos términos de glosario, cambios de contrato). Reglas concretas:

- actualizar primero el propio fichero del incremento/bug;
- integrar en `general-spec`/00–08 SOLO conocimiento estable y consolidado, nunca el historial completo de implementación;
- si no hace falta integrar nada, decirlo explícitamente en el propio artefacto (no dejarlo en silencio).

## Best-effort desde la CLI

El move físico a `_archive/` es best-effort desde la CLI: si el move falla, se reporta por stderr pero NO revierte la transición de estado ya persistida — el estado queda `archived` en `metadata.yml` aunque la carpeta no se haya movido; hay que resolver el move manualmente en ese caso excepcional.

## Relacionado

- [05_Incrementos.md](05_Incrementos.md)
- [06_Bugs.md](06_Bugs.md)
- [09_Revisiones.md](09_Revisiones.md)
- [08_Glosario.md](../08_Glosario.md)
