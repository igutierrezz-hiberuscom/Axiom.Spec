# 04 Interacciones UI

## Objetivo del documento

Describir la salida de `axiom doctor` para `TC-009`.

## Salida en `axiom doctor`

- **Caso construido:**
  ```
  ✓ [TC-009] Cobertura runtime de adapters multi-target (GATE 0031)
  ```
- **Caso sin construir (clone limpio):**
  ```
  ⚠ [TC-009] Cobertura runtime de adapters multi-target (GATE 0031)
       → 8/8 adapters presentes, pero dist/index.js no está construido en: [opencode, claude-code, ...]. Ejecute 'npm run build'.
  ```
