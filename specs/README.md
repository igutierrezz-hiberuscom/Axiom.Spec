# Specs de Axiom

Esta carpeta contiene la spec general del producto y los artefactos funcionales específicos.

## Navegación

1. `00_Resumen_Ejecutivo.md`: visión general y alcance del producto.
2. `01_Requisitos_Funcionales.md`: capacidades y comportamiento esperado.
3. `02_Requisitos_No_Funcionales.md`: requisitos de calidad y restricciones.
4. `03_Modelo_Operativo_y_Datos.md`: topología de repos, manifests y datos operativos.
5. `04_Flujos_SDD_y_Ciclo_de_Vida.md`: ciclo de trabajo sobre Axiom.
6. `05_Interfaces_Operativas.md`: CLI, launcher web, adapters y surfaces de operación.
7. `06_Integraciones_y_Capacidades.md`: herramientas, providers y capacidades externas.
8. `07_Gobierno_y_Seguridad.md`: reglas de ownership, trazabilidad y seguridad operativa.
9. `08_Glosario.md`: términos y definiciones del dominio del producto.

## Artefactos específicos

1. `increments/`: incrementos en formato carpeta.
2. `bugs/`: bugs en formato carpeta.
3. `increments/_archive/`: histórico de incrementos integrado o cerrado.
4. `manuales/`: material específico de esta instalación canónica; no es el manual runtime distribuible ni una segunda fuente de `Axiom/docs/**` — ver [manuales/README.md](manuales/README.md).
5. `decisions/`: decisiones gestionadas del producto.
6. `adr/`: decisiones arquitectónicas heredadas o complementarias.
7. `plans/`: planes de ejecución y sus artefactos.
8. `archive/`: material archivado de la spec.

## Regla

La spec general de Axiom debe escribirse directamente en este formato. No se debe arrastrar la estructura legacy como fuente canónica solo por continuidad.