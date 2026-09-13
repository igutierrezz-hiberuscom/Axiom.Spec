# 04 Interacciones UI

## Objetivo del documento

Describir la salida en terminal y ergonomía de interacción para la selección de instancias en la CLI de Axiom.

## Superficie UI afectada

- Comandos CLI: `axiom-increment`, `axiom-bug`, `axiom-plan`.
- Inspector de estado: `axiom state`.

## Flujo de interacción

1. **Uso directo con flag `--id`:**
   ```bash
   axiom axiom-increment plan --id INC-20260911-r14-workflow-instance-selection
   ```
   Avanza el plan para la instancia y la selecciona como activa para comandos subsiguientes sin flag `--id`.

2. **Comando `select`:**
   ```bash
   axiom axiom-increment select --id INC-20260911-r14-workflow-instance-selection
   ```
   Salida:
   ```
   [axiom increment] Instancia activa seleccionada: INC-20260911-r14-workflow-instance-selection (estado: planned)
   ```

3. **Invocación sin `--id` cuando no hay activa o no coincide:**
   ```
   [axiom increment] No hay una instancia activa seleccionada.
   Instancias en vuelo disponibles:
     - INC-20260911-r14-workflow-instance-selection (specifying)
     - INC-20260911-r14-operator-decisions-baseline (specifying)
   Usá '--id <id>' o ejecutá 'axiom axiom-increment select --id <id>'.
   ```

4. **Inspección en `axiom state --workflow increment`:**
   ```
   [axiom state] workflow 'increment' (2 instancias en vuelo)
     * INC-20260911-r14-workflow-instance-selection (activo): specifying
       Transiciones: increment-specify, increment-refine
       Recomendado(s): increment-specify
     - INC-20260911-r14-operator-decisions-baseline: specifying
   ```
