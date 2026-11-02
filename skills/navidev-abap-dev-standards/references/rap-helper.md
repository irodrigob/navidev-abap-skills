# Clase helper de RAP

## Por qué

Los programas no usan directamente las sentencias EML de un RAP. Si un RAP se usa desde muchos sitios y luego cambia, habría que replicar el cambio en todos. Con un helper por RAP, el cambio se hace en un solo punto.

## Nombre

`/NS/CL_[<código app>_]HELPER_<nombre>`. Ej.: `/NS/CL_HELPER_CONCILIATION`. Recuerda el límite de 30 caracteres.

Es una clase normal: `CREATE PRIVATE` + `CREATE_INSTANCE` (ver [abap-code-style.md](abap-code-style.md)).

## Antes de escribirla

Lee con el MCP de ADT:

1. La **behavior definition** del RAP: alias de cada entidad (`define behavior for /NS/I_CONC001 alias Conciliation`), numeración (managed/early/late), campos readonly, draft.
2. El **CDS interface** (`I_`) de cada entidad: nombres reales de los campos y cuáles son clave.
3. La **clase de mensajes** de la aplicación y la clase `_DATA` si existe.

Si el usuario no ha dicho qué campos identifican un registro para el upsert, pregúntalo: no tienen por qué ser los campos clave de la tabla (la clave suele ser un UUID y el registro se identifica por campos de negocio).

## Tipos

Por cada entidad, un tipo estructura y un tipo tabla que referencian al CDS interface, nombrados según el alias de la BDEF:

```abap
TYPES ts_conciliation TYPE /ns/i_conc001.
TYPES tt_conciliation TYPE STANDARD TABLE OF ts_conciliation WITH EMPTY KEY.
```

Como se usan en métodos públicos, van en la **sección pública**, igual que los tipos rango de los métodos de consulta. Los tipos internos van en la protegida.

## Métodos por entidad

| Método | Qué hace | Entrada | Salida |
| --- | --- | --- | --- |
| `EDIT_<entidad>` (público) | Upsert: busca por los campos identificadores; si existe actualiza aquí mismo, si no llama a `CREATE_<entidad>` | `IS_DATA` (tipo `ts_<entidad>`; a veces una tabla `IT_DATA`), `IV_COMMIT TYPE abap_boolean DEFAULT abap_false` | `EV_NODEKEY` (UUID, si la entidad lo tiene), `ES_RETURN TYPE bapiret2` |
| `CREATE_<entidad>` | Crea el registro | Mismos que `EDIT_` | Mismos que `EDIT_` |
| `DELETE_<entidad>` (público) | Borra | Campos clave del CDS interface (`IV_...`), `IV_COMMIT TYPE abap_boolean DEFAULT abap_false` | `ES_RETURN TYPE bapiret2` |
| `GET_<entidad>` (público) | Consulta | Un rango **opcional** por cada campo de búsqueda (inicialmente, los campos clave y los identificadores de negocio): `IT_R_<campo> TYPE tt_r_<campo> OPTIONAL` | `RT_DATA TYPE tt_<entidad>` (RETURNING) |

`ES_RETURN` devuelve un mensaje de la clase de mensajes de la aplicación (éxito o error). Si el RAP devuelve su propio mensaje en `REPORTED` (validaciones, autorizaciones), ese mensaje es más útil que uno genérico: conviértelo a `BAPIRET2` y devuélvelo.

`IV_COMMIT` es `abap_false` por defecto porque el helper puede llamarse desde dentro de otra transacción RAP (determinaciones, acciones, otro helper...), donde un `COMMIT ENTITIES` está prohibido o rompería la LUW. Solo quien inicia la transacción decide hacer commit. Avísalo en la respuesta al usuario.

La consulta lee del **CDS interface** con `SELECT ... WHERE campo IN it_r_campo`. Un rango vacío no filtra, por eso todos son opcionales.

## Sin lógica de negocio

El helper **solo simplifica la llamada al RAP**: no contiene lógica de negocio ni validaciones de los datos de entrada (campos obligatorios, coherencia, existencia de valores...). Esas comprobaciones las hace quien llama al helper (la clase de proceso, la BAdI, el programa...) o el propio RAP con sus validaciones. Los errores que devuelve el RAP sí se trasladan a `ES_RETURN`.

## Comentarios

Solo los comentarios que aportan algo que el código no dice (por qué se busca por campos que no son clave, de dónde sale la clave en la modificación...). Nada de comentarios de una línea que repiten lo que hace la sentencia siguiente.

## Plantilla

Ejemplo para `/NS/I_CONC001` (alias `Conciliation`, clave `NodeKey` UUID con numeración managed, identificada por negocio con `CompanyCode` + `FiscalYear` + `DocumentNumber`). Adapta nombres de campos, mensajes y numeración a lo que leas en el sistema.

Puntos que aporta frente a una versión mínima, y que conviene mantener:

- Números de mensaje en una estructura de constantes `cs_msgno`, en vez de literales repartidos.
- Conversión del `%msg` del RAP a `BAPIRET2` cuando existe.
- Un único método `commit` para no repetir el `COMMIT ENTITIES` y su tratamiento de errores.
- La clase de mensajes y los tipos de mensaje (`E`, `S`...) se toman de la clase `_DATA` de la aplicación (`cs_message-id`, `cs_message-types-...`), nunca como literales.
- La creación y la modificación se preparan en una tabla interna (`TYPE TABLE FOR CREATE` / `TYPE TABLE FOR UPDATE`) rellenada con `INSERT CORRESPONDING #( is_data ) ... ASSIGNING`, para no repetir los valores campo a campo. Después se activa el `%control` de cada campo que se envía (`<ls_create>-%control-<campo> = if_abap_behv=>mk-on`) y se llama a `CREATE FROM` / `UPDATE FROM`. En la modificación se asigna la clave del registro encontrado y solo se activan los campos de datos, no los identificadores.
- **Todos** los métodos (públicos, protegidos y privados) llevan ABAP Doc con el patrón de [abap-code-style.md](abap-code-style.md#abap-doc), y cada parámetro su descripción.

```abap
"! <p class="shorttext synchronized">Helper RAP de conciliaciones (/NS/I_CONC001)</p>
"! Encapsula las sentencias EML del BO para que ningún programa acceda directamente al RAP.
CLASS /ns/cl_helper_conciliation DEFINITION
  PUBLIC
  FINAL
  CREATE PRIVATE.

  PUBLIC SECTION.
    TYPES ts_conciliation TYPE /ns/i_conc001.
    TYPES tt_conciliation TYPE STANDARD TABLE OF ts_conciliation WITH EMPTY KEY.
    TYPES tt_r_node_key TYPE RANGE OF sysuuid_x16.
    TYPES tt_r_company TYPE RANGE OF bukrs.
    TYPES tt_r_fiscal_year TYPE RANGE OF gjahr.

    "! <p class="shorttext synchronized">Devuelve una nueva instancia del helper</p>
    "! @parameter ro_instance | <p class="shorttext">Instancia</p>
    CLASS-METHODS create_instance
      RETURNING VALUE(ro_instance) TYPE REF TO /ns/cl_helper_conciliation.

    "! <p class="shorttext synchronized">Crea o actualiza una conciliación (upsert por sociedad/ejercicio/documento)</p>
    "! @parameter is_data    | <p class="shorttext">Datos de la conciliación</p>
    "! @parameter iv_commit  | <p class="shorttext">Ejecutar COMMIT ENTITIES (no usar desde handlers RAP)</p>
    "! @parameter ev_nodekey | <p class="shorttext">UUID de la conciliación creada o modificada</p>
    "! @parameter es_return  | <p class="shorttext">Mensaje de retorno</p>
    METHODS edit_conciliation
      IMPORTING is_data    TYPE ts_conciliation
                iv_commit  TYPE abap_boolean DEFAULT abap_false
      EXPORTING ev_nodekey TYPE sysuuid_x16
                es_return  TYPE bapiret2.

    "! <p class="shorttext synchronized">Borra una conciliación</p>
    "! @parameter iv_node_key | <p class="shorttext">UUID de la conciliación</p>
    "! @parameter iv_commit   | <p class="shorttext">Ejecutar COMMIT ENTITIES (no usar desde handlers RAP)</p>
    "! @parameter es_return   | <p class="shorttext">Mensaje de retorno</p>
    METHODS delete_conciliation
      IMPORTING iv_node_key TYPE sysuuid_x16
                iv_commit   TYPE abap_boolean DEFAULT abap_false
      EXPORTING es_return   TYPE bapiret2.

    "! <p class="shorttext synchronized">Consulta de conciliaciones. Rango vacío = sin filtro</p>
    "! @parameter it_r_node_key    | <p class="shorttext">Rango de UUID</p>
    "! @parameter it_r_company     | <p class="shorttext">Rango de sociedades</p>
    "! @parameter it_r_fiscal_year | <p class="shorttext">Rango de ejercicios</p>
    "! @parameter rt_data          | <p class="shorttext">Conciliaciones encontradas</p>
    METHODS get_conciliation
      IMPORTING it_r_node_key    TYPE tt_r_node_key OPTIONAL
                it_r_company     TYPE tt_r_company OPTIONAL
                it_r_fiscal_year TYPE tt_r_fiscal_year OPTIONAL
      RETURNING VALUE(rt_data)   TYPE tt_conciliation.

  PROTECTED SECTION.
    CONSTANTS: BEGIN OF cs_msgno,
                 created          TYPE symsgno VALUE '001',
                 updated          TYPE symsgno VALUE '002',
                 deleted          TYPE symsgno VALUE '003',
                 error_create     TYPE symsgno VALUE '004',
                 error_update     TYPE symsgno VALUE '005',
                 error_delete     TYPE symsgno VALUE '006',
                 error_commit     TYPE symsgno VALUE '007',
               END OF cs_msgno.

    "! <p class="shorttext synchronized">Crea una conciliación</p>
    "! @parameter is_data    | <p class="shorttext">Datos de la conciliación</p>
    "! @parameter iv_commit  | <p class="shorttext">Ejecutar COMMIT ENTITIES</p>
    "! @parameter ev_nodekey | <p class="shorttext">UUID de la conciliación creada</p>
    "! @parameter es_return  | <p class="shorttext">Mensaje de retorno</p>
    METHODS create_conciliation
      IMPORTING is_data    TYPE ts_conciliation
                iv_commit  TYPE abap_boolean DEFAULT abap_false
      EXPORTING ev_nodekey TYPE sysuuid_x16
                es_return  TYPE bapiret2.

    "! <p class="shorttext synchronized">Ejecuta COMMIT ENTITIES y devuelve el error si falla</p>
    "! @parameter rs_return | <p class="shorttext">Mensaje de error; vacío si el commit es correcto</p>
    METHODS commit
      RETURNING VALUE(rs_return) TYPE bapiret2.

    "! <p class="shorttext synchronized">Construye un BAPIRET2 con un mensaje de la aplicación</p>
    "! @parameter iv_type       | <p class="shorttext">Tipo de mensaje</p>
    "! @parameter iv_number     | <p class="shorttext">Número de mensaje</p>
    "! @parameter iv_message_v1 | <p class="shorttext">Variable 1</p>
    "! @parameter iv_message_v2 | <p class="shorttext">Variable 2</p>
    "! @parameter iv_message_v3 | <p class="shorttext">Variable 3</p>
    "! @parameter iv_message_v4 | <p class="shorttext">Variable 4</p>
    "! @parameter rs_return     | <p class="shorttext">Mensaje construido</p>
    METHODS build_return
      IMPORTING iv_type          TYPE symsgty
                iv_number        TYPE symsgno
                iv_message_v1    TYPE simple OPTIONAL
                iv_message_v2    TYPE simple OPTIONAL
                iv_message_v3    TYPE simple OPTIONAL
                iv_message_v4    TYPE simple OPTIONAL
      RETURNING VALUE(rs_return) TYPE bapiret2.

    "! <p class="shorttext synchronized">Convierte un mensaje RAP (%msg de REPORTED) en BAPIRET2</p>
    "! @parameter io_message | <p class="shorttext">Mensaje RAP</p>
    "! @parameter rs_return  | <p class="shorttext">Mensaje convertido</p>
    METHODS build_return_from_msg
      IMPORTING io_message       TYPE REF TO if_abap_behv_message
      RETURNING VALUE(rs_return) TYPE bapiret2.

  PRIVATE SECTION.
ENDCLASS.

CLASS /ns/cl_helper_conciliation IMPLEMENTATION.

  METHOD create_instance.
    ro_instance = NEW #( ).
  ENDMETHOD.

  METHOD edit_conciliation.
    DATA lt_update TYPE TABLE FOR UPDATE /ns/i_conc001.

    CLEAR ev_nodekey.
    CLEAR es_return.

    " Búsqueda por los campos que identifican el registro (no tienen por qué ser clave)
    SELECT SINGLE NodeKey
      FROM /ns/i_conc001
      WHERE CompanyCode    = @is_data-CompanyCode
        AND FiscalYear     = @is_data-FiscalYear
        AND DocumentNumber = @is_data-DocumentNumber
      INTO @DATA(lv_node_key).

    IF sy-subrc <> 0.
      create_conciliation( EXPORTING is_data    = is_data
                                     iv_commit  = iv_commit
                           IMPORTING ev_nodekey = ev_nodekey
                                     es_return  = es_return ).
      RETURN.
    ENDIF.

    INSERT CORRESPONDING #( is_data ) INTO TABLE lt_update ASSIGNING FIELD-SYMBOL(<ls_update>).
    " La clave es la del registro encontrado: IS_DATA puede no traerla
    <ls_update>-NodeKey = lv_node_key.
    " Solo se modifican los campos de datos, no los identificadores
    <ls_update>-%control-Amount = if_abap_behv=>mk-on.
    <ls_update>-%control-Status = if_abap_behv=>mk-on.

    MODIFY ENTITIES OF /ns/i_conc001
      ENTITY Conciliation
        UPDATE FROM lt_update
      FAILED DATA(ls_failed)
      REPORTED DATA(ls_reported).

    IF ls_failed-conciliation IS NOT INITIAL.
      DATA lo_message TYPE REF TO if_abap_behv_message.
      lo_message = VALUE #( ls_reported-conciliation[ 1 ]-%msg OPTIONAL ).
      es_return = COND #( WHEN lo_message IS BOUND
                          THEN build_return_from_msg( lo_message )
                          ELSE build_return( iv_type       = /ns/cl_conc_data=>cs_message-types-error
                                             iv_number     = cs_msgno-error_update
                                             iv_message_v1 = is_data-DocumentNumber ) ).
      RETURN.
    ENDIF.

    IF iv_commit = abap_true.
      es_return = commit( ).
      IF es_return IS NOT INITIAL.
        RETURN.
      ENDIF.
    ENDIF.

    ev_nodekey = lv_node_key.
    es_return = build_return( iv_type       = /ns/cl_conc_data=>cs_message-types-success
                              iv_number     = cs_msgno-updated
                              iv_message_v1 = is_data-DocumentNumber ).
  ENDMETHOD.

  METHOD create_conciliation.
    DATA lt_create TYPE TABLE FOR CREATE /ns/i_conc001.

    CLEAR ev_nodekey.
    CLEAR es_return.

    INSERT CORRESPONDING #( is_data ) INTO TABLE lt_create ASSIGNING FIELD-SYMBOL(<ls_create>).
    <ls_create>-%cid = 'CONCILIATION'.
    <ls_create>-%control-CompanyCode    = if_abap_behv=>mk-on.
    <ls_create>-%control-FiscalYear     = if_abap_behv=>mk-on.
    <ls_create>-%control-DocumentNumber = if_abap_behv=>mk-on.
    <ls_create>-%control-Amount         = if_abap_behv=>mk-on.
    <ls_create>-%control-Status         = if_abap_behv=>mk-on.

    MODIFY ENTITIES OF /ns/i_conc001
      ENTITY Conciliation
        CREATE FROM lt_create
      MAPPED DATA(ls_mapped)
      FAILED DATA(ls_failed)
      REPORTED DATA(ls_reported).

    IF ls_failed-conciliation IS NOT INITIAL.
      DATA lo_message TYPE REF TO if_abap_behv_message.
      lo_message = VALUE #( ls_reported-conciliation[ 1 ]-%msg OPTIONAL ).
      es_return = COND #( WHEN lo_message IS BOUND
                          THEN build_return_from_msg( lo_message )
                          ELSE build_return( iv_type       = /ns/cl_conc_data=>cs_message-types-error
                                             iv_number     = cs_msgno-error_create
                                             iv_message_v1 = is_data-DocumentNumber ) ).
      RETURN.
    ENDIF.

    IF iv_commit = abap_true.
      es_return = commit( ).
      IF es_return IS NOT INITIAL.
        RETURN.
      ENDIF.
    ENDIF.

    ev_nodekey = VALUE #( ls_mapped-conciliation[ 1 ]-NodeKey OPTIONAL ).
    es_return = build_return( iv_type       = /ns/cl_conc_data=>cs_message-types-success
                              iv_number     = cs_msgno-created
                              iv_message_v1 = is_data-DocumentNumber ).
  ENDMETHOD.

  METHOD delete_conciliation.
    CLEAR es_return.

    MODIFY ENTITIES OF /ns/i_conc001
      ENTITY Conciliation
        DELETE FROM VALUE #( ( NodeKey = iv_node_key ) )
      FAILED DATA(ls_failed)
      REPORTED DATA(ls_reported).

    IF ls_failed-conciliation IS NOT INITIAL.
      DATA lo_message TYPE REF TO if_abap_behv_message.
      lo_message = VALUE #( ls_reported-conciliation[ 1 ]-%msg OPTIONAL ).
      es_return = COND #( WHEN lo_message IS BOUND
                          THEN build_return_from_msg( lo_message )
                          ELSE build_return( iv_type   = /ns/cl_conc_data=>cs_message-types-error
                                             iv_number = cs_msgno-error_delete ) ).
      RETURN.
    ENDIF.

    IF iv_commit = abap_true.
      es_return = commit( ).
      IF es_return IS NOT INITIAL.
        RETURN.
      ENDIF.
    ENDIF.

    es_return = build_return( iv_type   = /ns/cl_conc_data=>cs_message-types-success
                              iv_number = cs_msgno-deleted ).
  ENDMETHOD.

  METHOD get_conciliation.
    SELECT *
      FROM /ns/i_conc001
      WHERE NodeKey     IN @it_r_node_key
        AND CompanyCode IN @it_r_company
        AND FiscalYear  IN @it_r_fiscal_year
      INTO TABLE @rt_data.
  ENDMETHOD.

  METHOD commit.
    COMMIT ENTITIES RESPONSE OF /ns/i_conc001
      FAILED DATA(ls_failed)
      REPORTED DATA(ls_reported).

    IF sy-subrc = 0 AND ls_failed IS INITIAL.
      RETURN.
    ENDIF.

    DATA lo_message TYPE REF TO if_abap_behv_message.
    lo_message = VALUE #( ls_reported-conciliation[ 1 ]-%msg OPTIONAL ).
    IF lo_message IS BOUND.
      rs_return = build_return_from_msg( lo_message ).
      rs_return-type = /ns/cl_conc_data=>cs_message-types-error.
    ELSE.
      rs_return = build_return( iv_type   = /ns/cl_conc_data=>cs_message-types-error
                                iv_number = cs_msgno-error_commit ).
    ENDIF.
  ENDMETHOD.

  METHOD build_return.
    rs_return-type       = iv_type.
    rs_return-id         = /ns/cl_conc_data=>cs_message-id.
    rs_return-number     = iv_number.
    rs_return-message_v1 = |{ iv_message_v1 }|.
    rs_return-message_v2 = |{ iv_message_v2 }|.
    rs_return-message_v3 = |{ iv_message_v3 }|.
    rs_return-message_v4 = |{ iv_message_v4 }|.
    MESSAGE ID rs_return-id TYPE rs_return-type NUMBER rs_return-number
      WITH rs_return-message_v1 rs_return-message_v2 rs_return-message_v3 rs_return-message_v4
      INTO rs_return-message.
  ENDMETHOD.

  METHOD build_return_from_msg.
    rs_return-type       = COND #( WHEN io_message->m_severity IS INITIAL
                                   THEN /ns/cl_conc_data=>cs_message-types-error
                                   ELSE io_message->m_severity ).
    rs_return-id         = io_message->if_t100_message~t100key-msgid.
    rs_return-number     = io_message->if_t100_message~t100key-msgno.
    rs_return-message_v1 = io_message->if_t100_dyn_msg~msgv1.
    rs_return-message_v2 = io_message->if_t100_dyn_msg~msgv2.
    rs_return-message_v3 = io_message->if_t100_dyn_msg~msgv3.
    rs_return-message_v4 = io_message->if_t100_dyn_msg~msgv4.
    rs_return-message    = io_message->if_message~get_text( ).
  ENDMETHOD.

ENDCLASS.
```

Notas sobre la plantilla:

- Los números de mensaje son ilustrativos: usa o propone los que correspondan en la clase de mensajes de la aplicación y, en la respuesta, lista cada número con su texto propuesto para que el usuario los cree.
- Si la clase `_DATA` de la aplicación no existe, propón crearla con la estructura de constantes `cs_message` (ver [abap-code-style.md](abap-code-style.md)).
- La marca de `%control` va en el componente `%control`: `<ls_create>-%control-CompanyCode = if_abap_behv=>mk-on`. Escribir `<ls_create>-CompanyCode = if_abap_behv=>mk-on` compila, pero **sobrescribe el valor del campo** con `01` (convertido al tipo del campo) y graba datos erróneos.
- En `UPDATE` y `DELETE`, la tabla `WITH` / `FROM` tiene que llevar **siempre la clave** de la instancia (`NodeKey = lv_node_key`). Sin ella, el RAP no encuentra el registro y la operación falla.
- Si hay varias entidades (padre/hijo), repite tipos y métodos `EDIT_/CREATE_/DELETE_/GET_` por entidad, usando el alias de cada una. Para crear hijos se usa `CREATE BY \_<asociación>`.
- Si la BDEF tiene draft, al operar desde el helper sobre registros activos no informes `%is_draft` (por defecto es activo).
- `lo_message` se declara con su tipo (`TYPE REF TO if_abap_behv_message`) antes de asignarlo: `DATA(lo_message) = VALUE #( ... )` no compila porque `#` no tiene tipo del que deducirse.
- Si `build_return` ya existe en otro helper o en una clase común de la aplicación, reutilízalo en vez de duplicarlo.
