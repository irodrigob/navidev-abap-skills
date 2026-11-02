# navidev-abap-skills

Skills para agentes de IA (Claude Code y otros compatibles) orientadas al desarrollo ABAP, tanto en **ABAP Cloud** (BTP ABAP Environment / Steampunk) como en **ABAP On premise**.

## Skills incluidas

### `navidev-abap-dev-standards`

Estándar de desarrollo ABAP para que el agente cree y revise objetos de forma coherente:

- **Nomenclatura** de objetos de repositorio: clases, interfaces, tablas, elementos de datos, dominios, CDS (`R_`, `I_`, `C_`, `_SH`), behavior definitions, metadata extensions, servicios, communication scenarios, IAM, clases de mensajes y maintenance objects.
- **Estilo de código**: prefijos de variables, tipos, constantes y parámetros (`IV_`, `ES_`, `RT_`...), una declaración por sentencia, `WITH EMPTY KEY`, clases `CREATE PRIVATE` con `CREATE_INSTANCE`.
- **Clases helper de RAP**: encapsulan el EML de un business object (`EDIT_` con upsert, `CREATE_`, `DELETE_`, consulta por rangos) con devolución de mensajes en `BAPIRET2`.
- **Ayudas de búsqueda de dominio**: pareja de CDS `I_` + `C_..._SH`, con plantilla distinta para Cloud y para On premise.
- **Revisión de código** contra el estándar, organizada por temas y con propuesta corregida.

Antes de crear objetos, el agente pide (o confirma) el entorno, el paquete, la orden de transporte y el área de nombres. Si tiene acceso al sistema mediante el MCP de ADT, comprueba que los nombres estén libres, lee los objetos de los que depende y revisa qué números de mensaje están ocupados.

Las instrucciones de la skill están en castellano.

## Instalación

Con la CLI de [skills](https://skills.sh/):

```bash
npx skills add irodrigob/navidev-abap-skills
```

O manualmente, copiando la carpeta `skills/navidev-abap-dev-standards/` en la carpeta de skills de tu agente. En Claude Code:

- Para todos tus proyectos: `~/.claude/skills/navidev-abap-dev-standards/`
- Solo para un proyecto: `<proyecto>/.claude/skills/navidev-abap-dev-standards/`

## Configuración: área de nombres

En la skill, `/NS/` es un marcador que representa el área de nombres de tu producto. El agente la obtiene, por este orden:

1. De lo que indiques en la conversación, o del área que lleven los objetos que menciones.
2. De las instrucciones del proyecto (`CLAUDE.md`, `AGENTS.md`...).
3. Si no la encuentra, te la pregunta.

Para no tener que indicarla en cada sesión, añade una línea a las instrucciones de tu proyecto:

```markdown
Área de nombres del producto: /ABC/
Área de nombres para objetos de prueba: Z
```

## Requisitos

- Para crear, leer y activar objetos en el sistema, el agente necesita acceso a ABAP mediante el **MCP de ADT** (ABAP Development Tools). Sin él, la skill sigue sirviendo para generar y revisar código, y el agente indicará qué no ha podido comprobar.

## Casos de prueba

La carpeta `evals/navidev-abap-dev-standards/` contiene casos de prueba de la skill: un helper RAP, una ayuda de búsqueda On premise, la petición de datos antes de crear objetos y una revisión de código. Cada caso tiene la petición (`prompt.md`), el resultado esperado (`case.yaml`) y los criterios de evaluación (`graders/`). Usan un área de nombres ficticia (`/ACME/`).

Están fuera de la carpeta de la skill para que no se instalen junto con ella.

## Licencia

[MIT](LICENSE)
