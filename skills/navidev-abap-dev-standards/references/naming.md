# Nomenclatura de objetos

Patrón general: `<área><prefijo>_<descriptivo>[_<sufijo>]`.

- Área del producto: el namespace o prefijo del proyecto (por defecto). Ej.: `/NS/CL_HELPER_CONCILIATION`.
- Área de prueba: programas u objetos de prueba, normalmente `Z` o `Y`. Ej.: `ZCL_HELPER_CONCILIATION`.

**`/NS/` es un marcador** del área de nombres del producto (ver SKILL.md, sección 1): sustitúyelo siempre por el área real del proyecto. En objetos de prueba se sustituye por `Z`.

## Índice

1. Clases e interfaces
2. Parámetros de métodos
3. Diccionario
4. CDS
5. Business services
6. Cloud communication management
7. Cloud identity and access management
8. Clases de mensajes
9. Objetos de repositorio RAP (transport object, maintenance object)
10. Nombres de campos (clave UUID de un RAP managed y enlaces con padre y raíz)

## 1. Clases e interfaces

| Objeto | Patrón |
| --- | --- |
| Clase | `/NS/CL_<nombre>` |
| Interface | `/NS/IF_<nombre>` |
| Behavior pool (implementación de BDEF) | `/NS/BP_<nombre>` |
| Clase local handler (autorizaciones, validaciones, acciones) | `LHC_<entidad>` |
| Clase local saver (grabación) | `LSC_<entidad>` |
| Clase de constantes/tipos generales de una aplicación | `/NS/CL_<identificador>_DATA` |
| Clase helper de RAP | `/NS/CL_[<código app>_]HELPER_<nombre>` |

## 2. Parámetros de métodos

| Tipo parámetro | Simple | Estructura | Tabla interna | Rango | Objeto |
| --- | --- | --- | --- | --- | --- |
| Importing | `IV_` | `IS_` | `IT_` | `IT_R_` | `IO_` |
| Exporting | `EV_` | `ES_` | `ET_` | `ET_R_` | `EO_` |
| Changing | `CV_` | `CS_` | `CT_` | `CT_R_` | `CO_` |
| Returning | `RV_` | `RS_` | `RT_` | `RT_R_` | `RO_` |

## 3. Diccionario

| Objeto | Patrón | Notas |
| --- | --- | --- |
| Tabla de base de datos | `A_<nombre>` | Para nombres cortos usar contador: `/NS/A_TRD001` |
| Tabla borrador (draft) | `D_<nombre>` | Mismo nombre que la tabla principal cambiando `A` por `D`: `/NS/D_TRD001` |
| Elemento de datos | `E_<nombre>` | |
| Dominio | `D_<nombre>` | |

Ojo: tabla draft y dominio comparten prefijo `D_`; el nombre descriptivo los distingue (`D_TRD001` es draft de `A_TRD001`; `D_CONC_LOAD_STATUS` es un dominio).

## 4. CDS

| Objeto | Patrón | Notas |
| --- | --- | --- |
| Entidad base | `R_<nombre>` | Lecturas generales para construir entidades más complejas |
| Interface (entidad root / modelado) | `I_<nombre>` | |
| Projection / Consumer | `C_<nombre>` | Lo que se consume externamente (servicios, ayudas de búsqueda…) |
| Ayuda de búsqueda | `C_<nombre>_SH` | Consumer con sufijo `_SH` |
| Custom entity | `C_<nombre>` | Igual que Projection/Consumer |
| Behavior definition | = nombre del CDS al que apunta | |
| Metadata extension | = nombre del CDS al que apunta | Las anotaciones de UI van aquí, **nunca** en el CDS consumer |

## 5. Business services

| Objeto | Patrón |
| --- | --- |
| Service definition | `UI_<nombre>` |
| Service binding OData v2 | `UI_<nombre>_O2` |
| Service binding OData v4 | `UI_<nombre>_O4` |
| Service consumption model | `SC_<nombre>` (clase generada: `CL_SC_<nombre>`) |

## 6. Cloud communication management

| Objeto | Patrón | Notas |
| --- | --- | --- |
| Communication scenario | `API_<nombre>` | Convención propia (SAP suele generar `UI_`). Separar entrada y salida |
| Inbound | `IN_<nombre>` | Servicios que proveen datos a sistemas externos |
| Outbound | `OUT_<nombre>` | Servicios que consumen datos de sistemas externos |

## 7. Cloud identity and access management

| Objeto | Patrón |
| --- | --- |
| IAM App | `UI_<nombre>` (generadas) o `ROL_<nombre>` |
| IAM Business Catalog | `BC_<nombre>` |

## 8. Clases de mensajes

`<área de nombres><texto libre>`, sin prefijo. Ejemplos: `/NS/SALES`, `ZSD_MESSAGES`.

## 9. Objetos de repositorio RAP

### Transport object

- Dentro de un maintenance object: `BCM_TR_<nombre>`
- Individual: `TR_<nombre>`

### Business configuration management (maintenance object)

Se crea con *Generate ABAP Repository Objects*; hay que ajustar los nombres que propone el generador:

- BC Maintenance object → `BCM_<nombre>`; Transport object → `BCM_TR_<nombre>`.
- Business object → Data model (singleton):
  - Draft table, Interface view, Projection view → nomenclatura normal **respetando el sufijo final `_S`**.
  - Entity → `ST<NombreTabla>` (tabla `A_TRD014` → `STTrd014`).
- Dentro de *tables*:
  - Draft table, Interface view, Projection view → nomenclatura normal.
  - Entity → `<NombreTabla>` (tabla `A_TRD014` → `Trd014`).

Ejemplo completo para la tabla `/NS/A_TRD014`:

| Elemento | Nombre |
| --- | --- |
| Maintenance object | `/NS/BCM_TRD014` |
| Transport object | `/NS/BCM_TR_TRD014` |
| Singleton: draft / interface / projection | `/NS/D_TRD014_S`, `/NS/I_TRD014_S`, `/NS/C_TRD014_S` |
| Singleton: entity | `STTrd014` |
| Tabla: draft / interface / projection | `/NS/D_TRD014`, `/NS/I_TRD014`, `/NS/C_TRD014` |
| Tabla: entity | `Trd014` |

## 10. Nombres de campos

### Clave UUID de un RAP managed

En un RAP de tipo `managed`, el campo clave técnico de tipo `sysuuid_x16` se llama siempre igual, para que cualquiera lo reconozca en cualquier aplicación:

| Dónde | Nombre |
| --- | --- |
| Tabla de base de datos y tabla draft | `node_key` |
| CDS (`R_`, `I_`, `C_`) y behavior definition | `NodeKey` |

### Campos de enlace con el padre y la raíz

Las entidades hijas tienen su propio `node_key` como clave y, además, campos que las enlazan con los niveles superiores. Estos campos **no son clave**, ni en la tabla ni en las CDS:

| Nivel | Tabla | CDS y behavior definition | Contiene | Clave |
| --- | --- | --- | --- | --- |
| 1 (raíz) | `node_key` | `NodeKey` | UUID propio | Sí |
| 2 | `node_key` | `NodeKey` | UUID propio | Sí |
| 2 | `node_parent` | `NodeParent` | `node_key` del nivel 1 | No |
| 3 | `node_key` | `NodeKey` | UUID propio | Sí |
| 3 | `node_parent` | `NodeParent` | `node_key` del nivel 2 | No |
| 3 | `node_root` | `NodeRoot` | `node_key` del nivel 1 | No |
| 4 | `node_key` | `NodeKey` | UUID propio | Sí |
| 4 | `node_parent` | `NodeParent` | `node_key` del nivel 3 | No |
| 4 | `node_root` | `NodeRoot` | `node_key` del nivel 1 | No |

La regla para cualquier nivel N:

- `node_parent` = `node_key` del nivel N-1 (el padre directo). Existe desde el nivel 2.
- `node_root` = `node_key` del nivel 1 (la raíz), sea cual sea la profundidad. Existe desde el nivel 3. En el nivel 2 no hace falta, porque el padre ya es la raíz y coincidiría con `node_parent`.

Ejemplo de la raíz:

```abap
// Tabla /NS/A_TRD002
define table /ns/a_trd002 {
  key client   : abap.clnt not null;
  key node_key : sysuuid_x16 not null;
  ...
}
```

```
// CDS interface /NS/I_TRD002
define root view entity /NS/I_TRD002
  as select from /ns/a_trd002
{
  key node_key as NodeKey,
  ...
}
```

```
// Behavior definition
define behavior for /NS/I_TRD002 alias Operation
persistent table /ns/a_trd002
...
{
  field ( readonly, numbering : managed ) NodeKey;

  mapping for /ns/a_trd002
  {
    NodeKey = node_key;
    ...
  }
}
```

Ejemplo de una entidad de tercer nivel:

```abap
// Tabla /NS/A_TRD004 (nivel 3)
define table /ns/a_trd004 {
  key client   : abap.clnt not null;
  key node_key : sysuuid_x16 not null;
  node_parent  : sysuuid_x16;
  node_root    : sysuuid_x16;
  ...
}
```

```
// CDS interface /NS/I_TRD004 (nivel 3)
define view entity /NS/I_TRD004
  as select from /ns/a_trd004
  association        to parent /NS/I_TRD003 as _Parent on $projection.NodeParent = _Parent.NodeKey
  association [1..1] to /NS/I_TRD002        as _Root   on $projection.NodeRoot   = _Root.NodeKey
{
  key node_key    as NodeKey,
      node_parent as NodeParent,
      node_root   as NodeRoot,
      ...
      _Parent,
      _Root
}
```

En la behavior definition se mapean igual (`NodeParent = node_parent;`, `NodeRoot = node_root;`).

Al revisar código o diseñar una tabla, señala como incumplimiento cualquier otra denominación de estos campos (`uuid`, `guid`, `id`, `key_uuid`, `parent_uuid`, `root_uuid`...), y también que `node_parent` o `node_root` se declaren como clave.

