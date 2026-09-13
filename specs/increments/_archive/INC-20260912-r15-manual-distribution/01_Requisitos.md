# 01 Requisitos

## Objetivo del documento

Fijar qué debe cumplir la distribución del manual para ser útil, predecible y segura.

## Requisitos del incremento

### RQ-095-01 Destino declarado
El manual se materializa en un destino explícito dentro del proyecto, documentado y distinguible de la documentación propia del equipo.

### RQ-095-02 Mismo contrato en los tres momentos
Instalación, adopción y actualización aplican el mismo contrato de escritura. No hay un camino que deje el manual y otro que lo olvide.

### RQ-095-03 Idempotencia
Ejecutar la operación dos veces seguidas no produce cambios en la segunda. La comparación se hace por contenido, no por fecha.

### RQ-095-04 Política de sobrescritura explícita
El comportamiento ante un manual editado localmente está declarado y no pierde contenido en silencio. Si se elige un modelo de bloque generado con zona propia del equipo, la zona del equipo se preserva byte a byte, como ya ocurre con las superficies generadas existentes.

### RQ-095-05 Solo material distribuible
Se distribuye el manual genérico de producto. No se distribuye material específico de una instalación concreta.

### RQ-095-06 Mecanismo reutilizado
La materialización usa el patrón ya probado de catálogo con fuente declarada y huella verificable, y no una copia de carpeta sin control de integridad.

### RQ-095-07 Correspondencia verificable
Existe forma de comprobar que el manual presente en un proyecto corresponde a la versión de producto instalada, y de detectar que quedó atrás.

### RQ-095-08 Coste acotado
La operación no penaliza de forma perceptible los comandos frecuentes. Si el coste es relevante, la distribución se limita a instalación, adopción y actualización explícita.

## Reglas de negocio relevantes

- El manual distribuido es de solo lectura desde el punto de vista del contrato: el proyecto no es su fuente de verdad.
- Ninguna escritura puede pisar documentación propia del equipo sin declararlo.
- El manual debe servir también a un agente que opere en ese repositorio.

## Fuera de alcance funcional

- Redactar el manual.
- Distribuir el conjunto interno de manuales del repositorio canónico.
- Publicar el manual fuera del proyecto.
