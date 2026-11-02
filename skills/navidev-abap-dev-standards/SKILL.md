---
name: navidev-abap-dev-standards
description: Estándar de desarrollo ABAP (área de nombres propia /XXX/ o Z/Y) para ABAP Cloud (BTP/Steampunk) y On premise vía el MCP de ADT - nomenclatura de objetos de repositorio, diccionario, CDS, RAP, servicios, IAM y comunicación; prefijos de variables, parámetros y tipos; clases CREATE PRIVATE con CREATE_INSTANCE; clases helper de RAP (EDIT_/CREATE_/DELETE_ y consulta por rangos); ayudas de búsqueda de dominio (CDS I_ y C_..._SH); clase _DATA de constantes. Usar SIEMPRE que se vaya a crear, modificar, nombrar o revisar cualquier objeto o código ABAP - clases, interfaces, tablas, elementos de datos, dominios, CDS, behavior definitions, metadata extensions, service definitions/bindings, communication scenarios, IAM apps, clases de mensajes, maintenance objects - aunque el usuario no mencione las reglas ni la nomenclatura. También cuando pida revisar si un código cumple el estándar.
---

# Estándar de desarrollo ABAP

Esta skill recoge unas convenciones de desarrollo ABAP. Su objetivo es que todo lo que se cree en el sistema sea predecible: cualquiera debe poder adivinar el nombre de un objeto, saber dónde están sus tipos y constantes, y cambiar un RAP en un único sitio.

Las reglas y plantillas de esta skill son **el mínimo obligatorio, no el techo**. Cumple siempre las convenciones, pero aporta además lo que haría un buen desarrollador ABAP sénior: tratamiento de errores, explicaciones de las decisiones y avisos de lo que el usuario debe revisar. Respeta la responsabilidad de cada clase: un helper de RAP no lleva lógica de negocio ni validaciones de entrada, que corresponden a quien lo llama.

## 1. Datos de la sesión: entorno, paquete, orden y área

Crear un objeto en el sistema es difícil de deshacer: queda en una orden de transporte, en un paquete y con un nombre que no se puede renombrar. Por eso se necesitan cuatro datos:

1. **Entorno**: ABAP Cloud (BTP ABAP Environment / Steampunk) u On premise. Cambia qué APIs y tipos se pueden usar y cómo se construyen algunas CDS (p. ej. las ayudas de búsqueda de dominio).
2. **Paquete** de desarrollo.
3. **Orden de transporte**.
4. **Área de nombres**: el área de los desarrollos de producto (un namespace registrado como `/ABC/`, o un prefijo de cliente como `ZABC_`) y el área para objetos de prueba (normalmente `Z` o `Y`). Si el usuario habla de "prueba" o "test", usa el área de prueba.

### Área de nombres: `/NS/` es un marcador

En esta skill y sus referencias, **`/NS/` (y `/ns/` en minúsculas) significa "el área de nombres del producto"**; no es un namespace real. Sustitúyelo siempre por el área que corresponda. Para saber cuál es:

1. Si el usuario la ha indicado en la conversación, o los objetos que menciona ya la llevan (p. ej. `/ACME/I_TRD002`), úsala.
2. Si no, búscala en las instrucciones del proyecto (`CLAUDE.md`, `AGENTS.md` o similares), por ejemplo una línea como `Área de nombres del producto: /ABC/`.
3. Si tampoco está, **pregúntala** junto con el resto de datos de esta sección. Nunca inventes un área ni uses `/NS/` literalmente en un objeto.

Cómo gestionar estos datos a lo largo de la conversación:

- **La primera vez** que haya que crear objetos y falte alguno de estos datos, pregúntalos todos en un solo mensaje antes de crear nada.
- **Recuérdalos** durante el resto de la sesión. Cuando el usuario pida más objetos después, no vuelvas a preguntarlos desde cero, pero tampoco los des por supuestos en silencio: confírmalos en una línea, por ejemplo: *"Voy a usar el área /ABC/, el paquete /ABC/TRADING, la orden DEVK900123 y entorno ABAP Cloud. ¿Es correcto?"*. Si el usuario los indicó en la misma petición, úsalos sin preguntar.
- Sugiere al usuario que añada el área de nombres a las instrucciones del proyecto para no tener que repetirla en cada sesión.
- Si solo se pide código para ver o revisar (sin crear en el sistema), no hace falta paquete ni orden, pero sí el entorno si afecta al código.

En ABAP Cloud usa solo APIs liberadas y sintaxis ABAP Cloud; avisa si un tipo estándar que vas a usar (p. ej. `bukrs`, `waers`) podría no estar liberado.

## 2. Proponer antes de crear

Junto con las preguntas de la sección 1 (o cuando ya estén respondidas), presenta una **propuesta concreta** para que el usuario la valide de un vistazo, en vez de preguntas abiertas. Por ejemplo, para una tabla:

| Campo | Tipo | Clave |
| --- | --- | --- |
| `client` | `abap.clnt` | Sí |
| `company_code` | `bukrs` | Sí |
| ... | ... | ... |

Incluye en la propuesta los nombres de todos los objetos según la nomenclatura y las dudas de diseño que cambien el resultado (p. ej. si "tipo" es un importe o una clase de tipo de cambio; si hacen falta campos de auditoría porque se montará un RAP encima; qué constantes llevará la clase `_DATA`).

## 3. Acceso al sistema

Los objetos no existen como ficheros locales: se leen, crean, editan y activan con el MCP de ADT (`abap_repository_search_object`, `abap_object_create`, `abap_object_read_file`, `abap_object_edit_file`, `abap_object_activate`, `abap_object_check`...). Si `abap_destination_list` devuelve una lista vacía, no hay conexión: pide al usuario que inicie sesión en el sistema desde ADT en VS Code.

Usa el sistema para **comprobar en vez de suponer**. Antes de proponer o crear:

- **Nombres libres**: busca con `abap_repository_search_object` cada nombre que vayas a proponer (clase helper, tabla, CDS...). Si ya existe, dilo y propone otro. Para tablas con contador, busca el patrón (`/NS/A_TRD*`) y propone el siguiente número libre.
- **Objetos de los que dependes**: lee la BDEF, el CDS interface, el dominio, el elemento de datos o la clase `_DATA` para usar sus campos y constantes reales. Si la clase `_DATA` ya existe, amplíala en vez de crear otra.
- **Clase de mensajes**: léela (`abap_object_read_file`, tipo `MSAG`) para ver qué números están ocupados. Reutiliza mensajes existentes con el mismo texto y propone solo números libres para los nuevos.
- **Objetos liberados (ABAP Cloud)**: el MCP no tiene una herramienta que devuelva el estado de liberación de un objeto. La comprobación fiable es la de sintaxis: tras guardar el código, ejecuta `abap_object_check`; en ABAP Cloud marca como error cualquier uso de un objeto no liberado. Si aparece, busca el sucesor liberado (normalmente una CDS `I_*` o una clase `CL_*` de SAP) y corrígelo.

Después de crear o editar, activa y comprueba errores.

Si no hay conexión al sistema (o se pidió no tocarlo), indica en la respuesta **qué no has podido comprobar** (nombres libres, números de mensaje, liberación de tipos) para que el usuario lo sepa.

## 4. Idioma

Nombres de objetos, variables, métodos, tipos, etc. en **inglés**. Comentarios en **castellano** si se quiere. Las respuestas al usuario, en el idioma en que escriba.

## 5. Regla general de nombres

```
<área><prefijo>_<descriptivo>[_<sufijo>]
```

Ejemplo: `/NS/CL_HELPER_CONCILIATION`, con `/NS/` sustituido por el área real (o `ZCL_HELPER_CONCILIATION` en pruebas; el área `Z` es correcta para objetos de prueba, no es un incumplimiento). Resumen de los prefijos más usados:

| Objeto | Patrón |
| --- | --- |
| Clase / Interface | `CL_` / `IF_` |
| Behavior pool | `BP_` |
| Tabla BD / draft | `A_` / `D_` (mismo nombre cambiando A por D) |
| Elemento de datos / Dominio | `E_` / `D_` |
| CDS base / interface / consumer | `R_` / `I_` / `C_` |
| Ayuda de búsqueda | `C_<nombre>_SH` |
| Service definition / binding V2 / V4 | `UI_` / `UI_<n>_O2` / `UI_<n>_O4` |

En un RAP `managed`, la clave técnica `sysuuid_x16` se llama `node_key` en las tablas y `NodeKey` en las CDS y la behavior definition. Las entidades hijas añaden, **sin ser clave**, `node_parent`/`NodeParent` (desde el nivel 2) y `node_root`/`NodeRoot` (desde el nivel 3). Ver [references/naming.md](references/naming.md), sección 10.

La tabla completa (servicios, comunicación, IAM, mensajes, maintenance objects, parámetros de métodos...) está en **[references/naming.md](references/naming.md)**. Léela siempre que vayas a dar nombre a un objeto de repositorio o a los parámetros de un método.

## 6. Código ABAP: lo imprescindible

Las reglas completas con ejemplos están en **[references/abap-code-style.md](references/abap-code-style.md)**. Léelo antes de escribir o revisar código ABAP. Lo más importante:

- Prefijos: `ts_`/`tt_` tipos; `mt_`/`ms_`/`mv_` atributos; `lt_`/`ls_`/`lv_` locales; `cv_`/`cs_` constantes; field-symbol `<ls_xxx>` derivado del nombre de la tabla.
- Parámetros: `IV_/IS_/IT_/IT_R_/IO_`, `EV_/ES_/ET_...`, `CV_/CS_/CT_...`, `RV_/RS_/RT_/RT_R_/RO_`.
- Tipos tabla siempre `WITH EMPTY KEY`.
- **Una declaración por sentencia** (nada de `DATA: a ..., b ...`). La única excepción son las estructuras `BEGIN OF ... END OF`, que van en una sola sentencia encadenada.
- Clases `CREATE PRIVATE` con método estático `CREATE_INSTANCE` que devuelve siempre `NEW #( )` en `RO_INSTANCE` (no singleton), salvo que se indique lo contrario.
- Constantes y tipos generales de la aplicación en `/NS/CL_<identificador>_DATA`, no repartidos por otras clases.
- Tipos usados en la firma de métodos públicos, en la sección pública; el resto, en la protegida.
- **Todos** los métodos (públicos, protegidos y privados) llevan ABAP Doc en castellano, con la descripción en `"! <p class="shorttext synchronized">...</p>` y cada parámetro en `"! @parameter <nombre> | <p class="shorttext">...</p>` (ver [references/abap-code-style.md](references/abap-code-style.md#abap-doc)).
- Clase de mensajes y tipos de mensaje desde la clase `_DATA` (`cs_message-id`, `cs_message-types-error`...), nunca literales `'E'`/`'S'`.

## 7. Tareas específicas

| Si la tarea es... | Lee |
| --- | --- |
| Crear una clase helper para un RAP (crear/editar/borrar/consultar entidades) o cualquier código que vaya a usar EML (`MODIFY ENTITIES`, `READ ENTITIES`) desde fuera del behavior pool | [references/rap-helper.md](references/rap-helper.md) |
| Crear una ayuda de búsqueda para los valores fijos de un dominio | [references/cds-value-help.md](references/cds-value-help.md) |
| Crear un maintenance object (Business Configuration) o ajustar nombres generados | [references/naming.md](references/naming.md) (sección RAP) |

Sobre RAP: los programas no deben usar directamente las sentencias EML de un RAP, sino pasar por su clase helper. Así, si el RAP cambia, se toca solo el helper. Si ves que el código que vas a escribir llama a `MODIFY ENTITIES` de un RAP fuera de su helper, crea o amplía el helper en su lugar.

## 8. Revisar código existente

Cuando el usuario pida revisar si algo cumple el estándar, organiza la respuesta **por temas**, en este orden, omitiendo los que no tengan incidencias:

1. Nombre e instanciación de la clase
2. Tipos (`TYPES`)
3. Firma de los métodos (parámetros)
4. Atributos y variables (`DATA`)
5. Constantes
6. Field-symbols y bucles
7. Lógica y otras observaciones (fuera del estándar, pero relevantes: parámetros sin usar, datos que nunca se rellenan, tipos sin semántica...)

En cada tema indica: lo que hay, por qué incumple y la corrección. Ten en cuenta el contexto antes de marcar algo: `Z` es válido en objetos de prueba; la regla de rangos opcionales `IT_R_` aplica a los métodos de consulta de los helpers, no a cualquier parámetro. Termina con la **versión corregida** del código y con las preguntas que queden abiertas (entorno, de dónde salen los datos...). No crees nada en el sistema si solo se pidió revisión.

## 9. Cómo presentar el resultado

Cuando entregues código u objetos, estructura la respuesta con encabezados para que se lea de un vistazo:

1. **Qué se ha hecho**: objetos creados o ficheros generados (nombre completo, tipo, paquete, orden) y si se han activado sin errores. Si no se ha tocado el sistema, dilo.
2. **Qué hace cada objeto**: los métodos o campos principales y las decisiones de diseño con su motivo (p. ej. "busca por CompanyCode + OperationId, no por la clave técnica").
3. **Cómo usarlo**: la anotación, llamada o paso que necesita el usuario para aprovecharlo.
4. **Lo que tienes que crear o revisar**: mensajes propuestos con su texto, supuestos que conviene confirmar, restricciones de uso (p. ej. no hacer commit desde un handler RAP).
