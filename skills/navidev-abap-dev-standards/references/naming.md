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
