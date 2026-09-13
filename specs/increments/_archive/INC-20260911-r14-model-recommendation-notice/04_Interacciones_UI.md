# 04 Interacciones UI

## Objetivo del documento

Describir los textos exactos de los avisos y la salida en CLI y Launcher.

## Avisos en prompts copiados (cabecera)

- **Para target `single-mode` (`claude-code`):**
  ```
  [Axiom Model Notice] El adapter 'claude-code' utiliza un único modelo para toda la sesión. Clase recomendada para este paso: strong.
  ```

- **Para target `fallback-only` (ej: `github-copilot`):**
  ```
  [Axiom Model Notice] El adapter 'github-copilot' no soporta enrutado automático por slot. Seleccioná manualmente un modelo de clase 'strong' en tu herramienta.
  ```

- **Para target `multi-mode` (`opencode`):**
  *(Sin aviso)*

## Salida en `axiom model show`

```
Proyecto: Axiom
Target:   github-copilot  (support: fallback-only)
Overrides: 0 slot(s)

  increment       →  strong    (recomendado: strong | fallback al entorno)
  bug             →  strong    (recomendado: strong | fallback al entorno)
  plan            →  medium    (recomendado: medium | fallback al entorno)
  implementation  →  strong    (recomendado: strong | fallback al entorno)
  qa-e2e          →  strong    (recomendado: strong | fallback al entorno)
  review          →  strong    (recomendado: strong | fallback al entorno)
  archive         →  cheap     (recomendado: cheap | fallback al entorno)
```
