# 01 Requisitos

- **G-073.1**: toda acción launcher declarada tiene reconciliación exacta a workflow/command/target; lifecycle se deriva/valida contra fuente R-10 y no usa fallback.
- **G-073.2**: previews contienen comandos invocables mediante `axiom` y opciones existentes.
- **G-073.3**: se eliminan fields no consumidos; mutaciones de todas las lanes exigen ID explícito y mismatch falla antes de escribir/receipt.
- **G-074.1**: artefacto local se crea/valida antes de ADO; resultado local/remoto se representa separado.
- **G-074.2**: ADO es opcional, sus fallos no destruyen local y la UI no promete validación inexistente.
- **G-074.3**: solo HTTP(S) llega a href.
- **G-076.G**: tests cubren paridad/no-fallback/command/IDs/local-first/links en superficies efectivas.

## Restricción R-10

No se añade una fuente de transiciones ni se amplía el alcance de ACC-041/045; se consume el boundary vigente.
