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
| `EDIT_<entidad>` (público) | Upsert: valida que vengan los campos identificadores, busca por ellos; si existe actualiza aquí mismo, si no llama a `CREATE_<entidad>` | `IS_DATA` (tipo `ts_<entidad>`; a veces una tabla `IT_DATA`), `IV_COMMIT TYPE abap_boolean DEFAULT abap_false` | `EV_NODEKEY` (UUID, si la entidad lo tiene), `ES_RETURN TYPE bapiret2` |
| `CREATE_<entidad>` | Crea el registro | Mismos que `EDIT_` | Mismos que `EDIT_` |
| `DELETE_<entidad>` (público) | Borra | Campos clave del CDS interface (`IV_...`), `IV_COMMIT TYPE abap_boolean DEFAULT abap_false` | `ES_RETURN TYPE bapiret2` |
| `GET_<entidad>` (público) | Consulta | Un rango **opcional** por cada campo de búsqueda (inicialmente, los campos clave y los identificadores de negocio): `IT_R_<campo> TYPE tt_r_<campo> OPTIONAL` | `RT_DATA TYPE tt_<entidad>` (RETURNING) |

`ES_RETURN` devuelve un mensaje de la clase de mensajes de la aplicación (éxito o error). Si el RAP devuelve su propio mensaje en `REPORTED` (validaciones, autorizaciones), ese mensaje es más útil que uno genérico: conviértelo a `BAPIRET2` y devuélvelo.

`IV_COMMIT` es `abap_false` por defecto porque el helper puede llamarse desde dentro de otra transacción RAP (determinaciones, acciones, otro helper...), donde un `COMMIT ENTITIES` está prohibido o rompería la LUW. Solo quien inicia la transacción decide hacer commit. Avísalo en la respuesta al usuario.

La consulta lee del **CDS interface** con `SELECT ... WHERE campo IN it_r_campo`. Un rango vacío no filtra, por eso todos son opcionales.

## Plantilla

Ejemplo para `/NS/I_CONC001` (alias `Conciliation`, clave `NodeKey` UUID con numeración managed, identificada por negocio con `CompanyCode` + `FiscalYear` + `DocumentNumber`). Adapta nombres de campos, mensajes y numeración a lo que leas en el sistema.

Puntos que aporta frente a una versión mínima, y que conviene mantener:

- Validación de los campos identificadores antes de buscar.
- Números de mensaje en una estructura de constantes `cs_msgno`, en vez de literales repartidos.
- Conversión del `%msg` del RAP a `BAPIRET2` cuando existe.
- Un único método `commit` para no repetir el `COMMIT ENTITIES` y su tratamiento de errores.
- La clase de mensajes se toma de la clase `_DATA` de la aplicación.

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

    "! Devuelve una nueva instancia del helper
    CLASS-METHODS create_instance
      RETURNING VALUE(ro_instance) TYPE REF TO /ns/cl_helper_conciliation.

    "! Crea o actualiza una conciliación (upsert por sociedad/ejercicio/documento)
    "! @parameter is_data    | Datos de la conciliación
    "! @parameter iv_commit  | Ejecutar COMMIT ENTITIES (no usar desde handlers RAP)
    "! @parameter ev_nodekey | UUID de la conciliación creada o modificada
    "! @parameter es_return  | Mensaje de retorno
    METHODS edit_conciliation
      IMPORTING is_data    TYPE ts_conciliation
                iv_commit  TYPE abap_boolean DEFAULT abap_false
      EXPORTING ev_nodekey TYPE sysuuid_x16
                es_return  TYPE bapiret2.

    "! Borra una conciliación
    "! @parameter iv_node_key | UUID de la conciliación
    "! @parameter iv_commit   | Ejecutar COMMIT ENTITIES (no usar desde handlers RAP)
    "! @parameter es_return   | Mensaje de retorno
    METHODS delete_conciliation
      IMPORTING iv_node_key TYPE sysuuid_x16
                iv_commit   TYPE abap_boolean DEFAULT abap_false
      EXPORTING es_return   TYPE bapiret2.

    "! Consulta de conciliaciones. Rango vacío = sin filtro
    "! @parameter it_r_node_key    | Rango de UUID
    "! @parameter it_r_company     | Rango de sociedades
    "! @parameter it_r_fiscal_year | Rango de ejercicios
    "! @parameter rt_data          | Conciliaciones encontradas
    METHODS get_conciliation
      IMPORTING it_r_node_key    TYPE tt_r_node_key OPTIONAL
                it_r_company     TYPE tt_r_company OPTIONAL
                it_r_fiscal_year TYPE tt_r_fiscal_year OPTIONAL
      RETURNING VALUE(rt_data)   TYPE tt_conciliation.

  PROTECTED SECTION.
    " Números de mensaje de la clase de mensajes de la aplicación
    CONSTANTS: BEGIN OF cs_msgno,
                 created          TYPE symsgno VALUE '001',
                 updated          TYPE symsgno VALUE '002',
                 deleted          TYPE symsgno VALUE '003',
                 error_create     TYPE symsgno VALUE '004',
                 error_update     TYPE symsgno VALUE '005',
                 error_delete     TYPE symsgno VALUE '006',
                 error_commit     TYPE symsgno VALUE '007',
                 missing_key_data TYPE symsgno VALUE '008',
               END OF cs_msgno.

    METHODS create_conciliation
      IMPORTING is_data    TYPE ts_conciliation
                iv_commit  TYPE abap_boolean DEFAULT abap_false
      EXPORTING ev_nodekey TYPE sysuuid_x16
                es_return  TYPE bapiret2.

    "! Ejecuta COMMIT ENTITIES y devuelve el error si falla
    METHODS commit
      RETURNING VALUE(rs_return) TYPE bapiret2.

    "! Construye un BAPIRET2 con un mensaje de la clase de mensajes de la aplicación
    METHODS build_return
      IMPORTING iv_type          TYPE symsgty
                iv_number        TYPE symsgno
                iv_v1            TYPE simple OPTIONAL
                iv_v2            TYPE simple OPTIONAL
                iv_v3            TYPE simple OPTIONAL
      RETURNING VALUE(rs_return) TYPE bapiret2.

    "! Convierte un mensaje RAP (%msg de REPORTED) en BAPIRET2
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
    CLEAR ev_nodekey.
    CLEAR es_return.

    " Los campos que identifican el registro son obligatorios
    IF is_data-CompanyCode IS INITIAL OR is_data-FiscalYear IS INITIAL OR is_data-DocumentNumber IS INITIAL.
      es_return = build_return( iv_type = 'E' iv_number = cs_msgno-missing_key_data ).
      RETURN.
    ENDIF.

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

    " Solo se modifican los campos de datos, no los identificadores
    MODIFY ENTITIES OF /ns/i_conc001
      ENTITY Conciliation
        UPDATE FIELDS ( Amount Status )
        WITH VALUE #( ( NodeKey = lv_node_key
                        Amount  = is_data-Amount
                        Status  = is_data-Status ) )
      FAILED DATA(ls_failed)
      REPORTED DATA(ls_reported).

    IF ls_failed-conciliation IS NOT INITIAL.
      DATA lo_message TYPE REF TO if_abap_behv_message.
      lo_message = VALUE #( ls_reported-conciliation[ 1 ]-%msg OPTIONAL ).
      es_return = COND #( WHEN lo_message IS BOUND
                          THEN build_return_from_msg( lo_message )
                          ELSE build_return( iv_type   = 'E'
                                             iv_number = cs_msgno-error_update
                                             iv_v1     = is_data-DocumentNumber ) ).
      RETURN.
    ENDIF.

    IF iv_commit = abap_true.
      es_return = commit( ).
      IF es_return IS NOT INITIAL.
        RETURN.
      ENDIF.
    ENDIF.

    ev_nodekey = lv_node_key.
    es_return = build_return( iv_type = 'S' iv_number = cs_msgno-updated iv_v1 = is_data-DocumentNumber ).
  ENDMETHOD.

  METHOD create_conciliation.
    CLEAR ev_nodekey.
    CLEAR es_return.

    " NodeKey es readonly con numeración managed: no se informa
    MODIFY ENTITIES OF /ns/i_conc001
      ENTITY Conciliation
        CREATE FIELDS ( CompanyCode FiscalYear DocumentNumber Amount Status )
        WITH VALUE #( ( %cid           = 'CONCILIATION'
                        CompanyCode    = is_data-CompanyCode
                        FiscalYear     = is_data-FiscalYear
                        DocumentNumber = is_data-DocumentNumber
                        Amount         = is_data-Amount
                        Status         = is_data-Status ) )
      MAPPED DATA(ls_mapped)
      FAILED DATA(ls_failed)
      REPORTED DATA(ls_reported).

    IF ls_failed-conciliation IS NOT INITIAL.
      DATA lo_message TYPE REF TO if_abap_behv_message.
      lo_message = VALUE #( ls_reported-conciliation[ 1 ]-%msg OPTIONAL ).
      es_return = COND #( WHEN lo_message IS BOUND
                          THEN build_return_from_msg( lo_message )
                          ELSE build_return( iv_type   = 'E'
                                             iv_number = cs_msgno-error_create
                                             iv_v1     = is_data-DocumentNumber ) ).
      RETURN.
    ENDIF.

    IF iv_commit = abap_true.
      es_return = commit( ).
      IF es_return IS NOT INITIAL.
        RETURN.
      ENDIF.
    ENDIF.

    " Con numeración early/managed el UUID ya está en MAPPED
    ev_nodekey = VALUE #( ls_mapped-conciliation[ 1 ]-NodeKey OPTIONAL ).
    es_return = build_return( iv_type = 'S' iv_number = cs_msgno-created iv_v1 = is_data-DocumentNumber ).
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
                          ELSE build_return( iv_type = 'E' iv_number = cs_msgno-error_delete ) ).
      RETURN.
    ENDIF.

    IF iv_commit = abap_true.
      es_return = commit( ).
      IF es_return IS NOT INITIAL.
        RETURN.
      ENDIF.
    ENDIF.

    es_return = build_return( iv_type = 'S' iv_number = cs_msgno-deleted ).
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
      rs_return-type = 'E'.
    ELSE.
      rs_return = build_return( iv_type = 'E' iv_number = cs_msgno-error_commit ).
    ENDIF.
  ENDMETHOD.

  METHOD build_return.
    " La clase de mensajes sale de la clase _DATA de la aplicación
    rs_return-type       = iv_type.
    rs_return-id         = /ns/cl_conc_data=>cv_msgid.
    rs_return-number     = iv_number.
    rs_return-message_v1 = |{ iv_v1 }|.
    rs_return-message_v2 = |{ iv_v2 }|.
    rs_return-message_v3 = |{ iv_v3 }|.
    MESSAGE ID rs_return-id TYPE rs_return-type NUMBER rs_return-number
      WITH rs_return-message_v1 rs_return-message_v2 rs_return-message_v3
      INTO rs_return-message.
  ENDMETHOD.

  METHOD build_return_from_msg.
    rs_return-type       = COND #( WHEN io_message->m_severity IS INITIAL
                                   THEN 'E'
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
- Si la clase `_DATA` de la aplicación no existe, propón crearla con la constante `cv_msgid` (ver [abap-code-style.md](abap-code-style.md)).
- Si hay varias entidades (padre/hijo), repite tipos y métodos `EDIT_/CREATE_/DELETE_/GET_` por entidad, usando el alias de cada una. Para crear hijos se usa `CREATE BY \_<asociación>`.
- Si la BDEF tiene draft, al operar desde el helper sobre registros activos no informes `%is_draft` (por defecto es activo).
- `lo_message` se declara con su tipo (`TYPE REF TO if_abap_behv_message`) antes de asignarlo: `DATA(lo_message) = VALUE #( ... )` no compila porque `#` no tiene tipo del que deducirse.
- Si `build_return` ya existe en otro helper o en una clase común de la aplicación, reutilízalo en vez de duplicarlo.
