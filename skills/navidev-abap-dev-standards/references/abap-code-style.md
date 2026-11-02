# Estilo de código ABAP

## Prefijos

| Elemento | Prefijo |
| --- | --- |
| Tipo estructura (`TYPES`) | `ts_` |
| Tipo tabla interna (`TYPES`) | `tt_` |
| Tipo rango (`TYPES`) | `tt_r_` |
| Atributo / variable global – tabla | `mt_` |
| Atributo / variable global – estructura | `ms_` |
| Atributo / variable global – simple | `mv_` |
| Constante simple | `cv_` |
| Constante estructura | `cs_` |
| Variable local – tabla | `lt_` |
| Variable local – estructura | `ls_` |
| Variable local – simple | `lv_` |
| Field-symbol | Nombre de la tabla cambiando **solo** la primera letra por `l`, sin tocar el resto (se mantiene el plural si lo tiene): `mt_company` → `<ls_company>`, `mt_rates` → `<ls_rates>` |

Las declaraciones inline siguen los mismos prefijos: `DATA(lt_company)`, `DATA(lv_count)`, `FIELD-SYMBOL(<ls_company>)`.

Parámetros de métodos: ver tabla en [naming.md](naming.md) (sección 2).

## Tipos tabla

Siempre `WITH EMPTY KEY`:

```abap
TYPES tt_company TYPE STANDARD TABLE OF ts_company WITH EMPTY KEY.
```

## Una declaración por sentencia

Cada `DATA`, `TYPES` o `CONSTANTS` declara un solo elemento, para poder copiar o mover líneas sin reajustar comas y puntos. Poner `:` tras la palabra clave es indiferente (por costumbre se suele poner).

```abap
" Correcto
DATA: mt_company TYPE tt_company.
DATA: ms_company TYPE ts_company.

" Correcto
DATA mt_company TYPE tt_company.
DATA ms_company TYPE ts_company.

" Incorrecto
DATA: mt_company TYPE tt_company,
      ms_company TYPE ts_company.
```

Lo mismo aplica a `METHODS`/`CLASS-METHODS` en la definición: un método por sentencia.

### Excepción: estructuras

Las estructuras de constantes, tipos o variables se declaran en **una sola sentencia** con `BEGIN OF ... END OF`:

```abap
CONSTANTS: BEGIN OF cs_load_status,
             in_process TYPE /ns/e_conc_load_status VALUE 'IN',
             completed  TYPE /ns/e_conc_load_status VALUE 'CO',
           END OF cs_load_status.
```

Nunca así:

```abap
CONSTANTS BEGIN OF cs_load_status.
CONSTANTS in_process TYPE /ns/e_conc_load_status VALUE 'IN'.
CONSTANTS completed TYPE /ns/e_conc_load_status VALUE 'CO'.
CONSTANTS END OF cs_load_status.
```

## Bucles

```abap
LOOP AT mt_company ASSIGNING FIELD-SYMBOL(<ls_company>).
ENDLOOP.
```

## Clases

Salvo que se indique lo contrario, las clases se declaran `CREATE PRIVATE` y solo se instancian con el método estático público `CREATE_INSTANCE`, cuyo parámetro RETURNING se llama `RO_INSTANCE` y es del tipo de la propia clase. Devuelve **siempre una instancia nueva** (no es singleton):

```abap
CLASS /ns/cl_helper_conciliation DEFINITION
  PUBLIC
  FINAL
  CREATE PRIVATE.

  PUBLIC SECTION.
    CLASS-METHODS create_instance
      RETURNING VALUE(ro_instance) TYPE REF TO /ns/cl_helper_conciliation.
  PROTECTED SECTION.
  PRIVATE SECTION.
ENDCLASS.

CLASS /ns/cl_helper_conciliation IMPLEMENTATION.
  METHOD create_instance.
    ro_instance = NEW #( ).
  ENDMETHOD.
ENDCLASS.
```

Si la clase tiene constructor con parámetros, `CREATE_INSTANCE` tiene los mismos parámetros y se los pasa al `NEW #( )`:

```abap
    CLASS-METHODS create_instance
      IMPORTING iv_company_code    TYPE bukrs
      RETURNING VALUE(ro_instance) TYPE REF TO /ns/cl_xxx.
    METHODS constructor
      IMPORTING iv_company_code TYPE bukrs.
...
  METHOD create_instance.
    ro_instance = NEW #( iv_company_code = iv_company_code ).
  ENDMETHOD.
```

### Visibilidad de los tipos

Los tipos usados en la firma de métodos públicos van en la **sección pública** (ABAP no permite usar tipos protegidos en métodos públicos). El resto de tipos, en la **sección protegida**.

## Clase de datos de la aplicación

Cada aplicación tiene normalmente una clase `/NS/CL_<identificador>_DATA` con las constantes y tipos generales de esa aplicación (se pueden crear clases similares en subpaquetes si hace falta). Así están centralizados y no se duplican en varias clases.

Antes de declarar una constante o tipo "de aplicación" (estados, tipos de documento, clase de mensajes, rangos comunes...) en otra clase, busca si existe la clase `_DATA` y úsala o amplíala.

```abap
CLASS /ns/cl_conc_data DEFINITION
  PUBLIC
  FINAL
  CREATE PRIVATE.

  PUBLIC SECTION.
    CONSTANTS cv_msgid TYPE symsgid VALUE '/NS/CONC'.

    CONSTANTS: BEGIN OF cs_load_status,
                 in_process TYPE /ns/e_conc_load_status VALUE 'IN',
                 completed  TYPE /ns/e_conc_load_status VALUE 'CO',
               END OF cs_load_status.

    TYPES tt_r_company TYPE RANGE OF bukrs.
  PROTECTED SECTION.
  PRIVATE SECTION.
ENDCLASS.

CLASS /ns/cl_conc_data IMPLEMENTATION.
ENDCLASS.
```

Una clase que solo contiene constantes y tipos no necesita `CREATE_INSTANCE`.
