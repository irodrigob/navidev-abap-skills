# Ayudas de búsqueda de valores de dominio

Para un dominio con valores fijos se crean **dos CDS**:

1. `I_<nombre>` (interface): obtiene valores y textos del dominio.
2. `C_<nombre>_SH` (consumer): la que se usa en otros CDS como ayuda de búsqueda (`@Consumption.valueHelpDefinition`). El sufijo `_SH` la identifica.

El nombre debe dejar claro a qué dominio apunta: dominio `/NS/D_CONC_LOAD_STATUS` → `/NS/I_CONC_LOAD_STATUS` y `/NS/C_CONC_LOAD_STATUS_SH`.

**La CDS `I_` es distinta en ABAP Cloud y On premise**, porque en Cloud las tablas DD07L/DD07T no están liberadas. Si no sabes el entorno, pregúntalo.

## CDS `I_`

- Solo valor, idioma y descripción. Clave única: el valor.
- El valor se convierte con `cast` al **elemento de datos** del dominio (no al dominio).
- `@ObjectModel.dataCategory: #TEXT`, `@Semantics.language` en el idioma y `@Semantics.text` en la descripción.

### ABAP Cloud

```
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Conciliación - Datos  estado carga'
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType:{
    serviceQuality: #X,
    sizeCategory: #S,
    dataClass: #MIXED
}
@ObjectModel.dataCategory: #TEXT
define view entity /NS/I_CONC_LOAD_STATUS
  as select from DDCDS_CUSTOMER_DOMAIN_VALUE( p_domain_name: '/NS/D_CONC_LOAD_STATUS' ) as value
  association [0..1] to DDCDS_CUSTOMER_DOMAIN_VALUE_T as _text on  value.domain_name    = _text.domain_name
                                                               and value.value_position = _text.value_position
                                                               and _text.language       = $session.system_language
{
  key cast( value_low as /ns/e_conc_load_status )           as Status,
      @Semantics.language: true
      _text( p_domain_name : '/NS/D_CONC_LOAD_STATUS').language,
      @Semantics.text: true
      _text( p_domain_name : '/NS/D_CONC_LOAD_STATUS').text as StatusDesc
}
```

### On premise

```
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Conciliación - Datos  estado carga'
@Metadata.ignorePropagatedAnnotations: true
@ObjectModel.usageType:{
    serviceQuality: #X,
    sizeCategory: #S,
    dataClass: #MIXED
}
@ObjectModel.dataCategory: #TEXT
define view entity /NS/I_CONC_LOAD_STATUS
  as select from dd07l as value
  association [0..1] to dd07t as _text on  value.domname    = _text.domname
                                       and value.valpos     = _text.valpos
                                       and value.as4local   = _text.as4local
                                       and value.as4vers    = _text.as4vers
                                       and _text.ddlanguage = $session.system_language
{
  key cast( value.domvalue_l as /ns/e_conc_load_status )           as Status,
      @Semantics.language: true
      _text.ddlanguage as Language,
      @Semantics.text: true
      _text.ddtext as StatusDesc
}
where value.as4local = 'A'
  and value.domname  = '/NS/D_CONC_LOAD_STATUS'
```

El enlace con `dd07t` incluye `as4vers` para completar la clave de `dd07t`; sin él podrían salir textos duplicados si el dominio tiene varias versiones.

El nombre del dominio aparece varias veces (en Cloud, también en cada parámetro de `_text`): sustitúyelo en todas.

## CDS `C_..._SH`

Solo valor y descripción. Clave única: el valor. Igual en Cloud y On premise.

```
@AbapCatalog.viewEnhancementCategory: [#NONE]
@AccessControl.authorizationCheck: #NOT_REQUIRED
@EndUserText.label: 'Ayuda para búsqueda de estado de carga'
@Metadata.ignorePropagatedAnnotations: false
@ObjectModel.dataCategory: #VALUE_HELP
@ObjectModel.usageType:{
    serviceQuality: #X,
    sizeCategory: #S,
    dataClass: #MIXED
}
@ObjectModel.resultSet.sizeCategory: #XS
define view entity /NS/C_CONC_LOAD_STATUS_SH
  as select from /NS/I_CONC_LOAD_STATUS
{
      @ObjectModel.text.element    : ['StatusDesc']
      @UI.textArrangement: #TEXT_ONLY
  key Status,
      @Semantics.text: true
      StatusDesc
}
```

## Pasos

1. Lee el dominio y su elemento de datos en el sistema para confirmar nombres y que tiene valores fijos.
2. Crea y activa primero la `I_`, después la `C_..._SH` (depende de la `I_`).
3. Nombra los campos según lo que representan (`Status`/`StatusDesc`, `DocumentType`/`DocumentTypeDesc`...), con la descripción terminada en `Desc`.
4. En la respuesta explica cómo usarla: el campo del CDS que la consume lleva
   ```
   @Consumption.valueHelpDefinition: [{ entity: { name: '/NS/C_CONC_LOAD_STATUS_SH', element: 'Status' } }]
   ```
   Si el CDS consumidor tiene metadata extension, la anotación va allí (las anotaciones de UI nunca van en el CDS consumer).
5. Explica brevemente qué hace cada CDS (de dónde lee, cuál es la clave, el `cast`, el filtro) y el orden de creación.
