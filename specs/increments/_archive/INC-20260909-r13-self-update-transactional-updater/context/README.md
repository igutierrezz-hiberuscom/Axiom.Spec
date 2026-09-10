# Context

## Propósito

Reservar un espacio local para evidencia técnica directamente necesaria para especificar o revisar el incremento 3/5 del actualizador global transaccional. La fuente canónica del comportamiento son los Markdown del incremento y, después de aceptación, las specs generales integradas mediante Axiom/Core. Este directorio no acredita implementación actual.

## Qué puede vivir aquí

Solo material acotado y trazable que ayude a revisar ACC-078 y las ramas `apply/recover` de ACC-079, por ejemplo:

- mapas de símbolos/rutas obtenidos durante el inventario del runtime;
- matrices de compatibilidad Windows/POSIX y primitivas de reemplazo atómico;
- extractos sanitizados de ejecuciones con repositorios Git locales;
- diagramas del journal, lock, activación y recovery que no sustituyan el contrato normativo;
- evidencia de fault injection, rollback y retry vinculada a un commit de implementación;
- notas de revisión independiente y decisiones técnicas ya confirmadas.

Todo archivo futuro debe identificar fuente, fecha, commit y criterio de aceptación al que aporta evidencia. No se almacenan secretos, tokens, URLs con credenciales ni dumps de entorno.

## Qué no debe vivir aquí

- Metadata, status, receipts, índices o enlaces estructurales gestionados por Axiom/Core.
- Copias de las specs generales, historial completo de implementación o documentación final de release.
- Código del runtime, fixtures ejecutables o dependencias vendorizadas.
- Artefactos del Launcher del incremento 4 o del pipeline/documentación final del incremento 5.
- Logs sin sanitizar, credenciales, datos personales o paths ajenos innecesarios.
- Instrucciones de `stash`, `reset`, force, clean o borrado manual de locks/staging.

## Estructura sugerida

Si durante implementación se necesita evidencia persistente y el flujo Axiom la autoriza, usar nombres descriptivos y pocos archivos:

- `runtime-symbol-map.md` para el mapeo confirmado entre responsabilidades del plan y código real;
- `platform-atomicity-matrix.md` para Windows/POSIX;
- `fault-injection-evidence.md` para fronteras, outcome y estado observado;
- `review-notes.md` para hallazgos y resolución.

No se crean esos archivos por anticipado. La ausencia de evidencia requerida mantiene el gate `STOP`; la evidencia final estable se consolida en su spec propietaria, no se convierte este directorio en un índice paralelo.
