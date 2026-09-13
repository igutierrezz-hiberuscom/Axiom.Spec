# Estructura del manual único de Axiom

## Decisión

`Axiom/docs/**` es el único manual de producto del runtime. El contenido
específico del repositorio canónico no se fusiona con este manual y no se
considera una segunda fuente.

La organización del manual es:

- `docs/README.md`: entrada general y navegación por áreas.
- `docs/installation.md`, `docs/configuration/`, `docs/usage/`,
	`docs/generated-files.md` y `docs/troubleshooting.md`: operación general,
	instalación, configuración, uso, outputs y diagnóstico.
- `docs/cli/README.md`: índice de familias de comandos.
- `docs/cli/<command>.md`: página de cada familia de comandos de primer nivel.
- `docs/configuration/files/<file>.md`: página de cada YAML vigente, con los
	contratos históricos separados y marcados.

## Familia documentada

A efectos de cobertura, una familia documentada es cada comando de primer
nivel que el programa Commander registra en `apps/cli/src/index.ts`. La
familia se identifica por el nombre que expone la ayuda compilada; el comando
integrado `help` no cuenta. Una sola página cubre todos los subcomandos de
esa familia mediante secciones propias, salvo que una página adicional sea
necesaria por volumen y quede enlazada desde la página de la familia.

## Esquema mínimo de una página de comandos

Cada página activa debe declarar, contra la ayuda y el código vigentes:

1. qué hace el comando y para qué sirve;
2. cuándo usarlo y con qué comandos se conecta;
3. sintaxis, subcomandos y opciones reales;
4. archivos, estado y repositorios que lee o escribe;
5. validaciones, gates y condiciones de bloqueo;
6. resultado observable en éxito y en error, incluidos exit codes cuando sean
	 relevantes.

La redacción debe servir a una persona y a un agente: los ejemplos deben ser
ejecutables, los nombres de archivos deben ser rutas reales y las
afirmaciones no verificables no se presentan como contrato.

## Convenciones e histórico

Las páginas de comandos usan el nombre literal normalizado a
`docs/cli/<command>.md`; el índice se llama `docs/cli/README.md`. Los nombres
de familias con guion conservan el guion en el nombre del archivo. Los
manuales históricos permanecen dentro de `docs/`, pero empiezan con un
encabezado visible `Histórico` y declaran que no son instrucciones vigentes.
No entran en el recuento de cobertura ni pueden sustituir una página activa.
