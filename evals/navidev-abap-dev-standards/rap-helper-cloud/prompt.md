---
tags: [skill-creator]
max_turns: 20
timeout_seconds: 600
allowed_tools: [Read, Glob, Grep, Skill, Write, Edit, Bash]
---

Estoy en ABAP Cloud, paquete /ACME/TRADING, orden DEVK900123. Necesito la clase helper para el RAP de operaciones de trading. La behavior definition es:

managed implementation in class /acme/bp_i_trd002 unique;
strict ( 2 );
define behavior for /ACME/I_TRD002 alias Operation
persistent table /acme/a_trd002
lock master
authorization master ( instance )
{
  create; update; delete;
  field ( readonly, numbering : managed ) NodeKey;
}

El CDS /ACME/I_TRD002 tiene los campos: key NodeKey (sysuuid_x16), CompanyCode (bukrs), OperationId (/acme/e_trd_operation_id), OperationDate (datum), Amount (/acme/e_trd_amount), Currency (waers), Status (/acme/e_trd_status). Una operación se identifica por sociedad + OperationId. La clase de mensajes es /ACME/TRADING.

No lo crees en el sistema: déjame el código fuente completo de la clase en un fichero .abap.
