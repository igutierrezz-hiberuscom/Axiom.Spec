# 01 Requisitos

## Objetivo del documento

Fijar la fuente de evidencia correcta para cada repositorio del workspace durante una auditoría runtime.

## Requisitos del incremento

1. Las tres superficies declaran `Axiom/` como objeto de auditoría y fuente primaria de código, configuración, tests y documentación operativa del producto.
2. Declaran `Axiom.Spec/` como fuente de intención, contraste y consolidación; no como sustituto de evidencia ejecutable.
3. Declaran `Axiom.SDD/` como contrato operativo/ubicación del agente, no como prueba de integraciones runtime; excepciones solo por solicitud explícita.
4. Las tres copias mantienen los límites de escritura existentes.

## Reglas de negocio relevantes

La prioridad de evidencia ejecutable sobre documentación histórica no cambia. Una configuración MCP de Kiro/Copilot bajo Axiom.SDD no es una integración del producto Axiom.

## Fuera de alcance funcional

No se cambia ningún comportamiento de CLI, MCP, agentes distintos ni configuración del runtime.
