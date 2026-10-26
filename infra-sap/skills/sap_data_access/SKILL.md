---
name: sap_data_access
description: >-
  Consultar SAP con las tools proxy-sap_*: pedidos de venta, productos,
  bancos y saldos, tablas por RFC, MD04 y variable logística, con el usuario
  SAP de cada persona.
metadata:
  category: data
  display-name: "Consultas SAP"
---

# Consultas SAP

Cada llamada usa la conexión SAP que la persona tiene en Mi perfil, así que
solo ve lo que su usuario SAP puede ver. Si la respuesta dice que faltan las
credenciales SAP o que el sistema no está en SAP_SYSTEMS, díselo: tiene que
revisar su conexión en Mi perfil › Conexiones.

## Qué usar

- **Pedidos de venta**: `getSalesOrders` con filtros (`SoldToParty`,
  `SalesOrderType`, `SalesOrganization`, `SalesOrderDateFrom/To`,
  `CustomerSearch`) y paginación (`page`, `pageSize`). Detalle con
  `getSalesOrdersBySalesOrder` y posiciones con `getSalesOrdersBySalesOrderItems`.
  Los valores válidos de tipo de pedido y cliente salen de los `ValueHelp`.
- **Productos**: `getProducts` (paginado) y `getProductsByProductId`.
- **Bancos y tesorería**: `getHouseBanks`, cuentas de un banco, saldos
  (`getBankBalances`, `Summary`, `GlAccounts`) y posición de caja.
- **Tablas**: `postRfcTableReaderReadTable` con `tableName`, `fields` y
  `filters` en sintaxis ABAP (`"MATNR LIKE '1*'"`, `"MATKL = '001'"`). Pide
  siempre los campos que necesitas y limita filas: tablas como MARA o VBAP son
  enormes. Si no conoces los campos, `getRfcTableReaderTableFieldsByTableName`.
- **Fabricación**: `getManipuladorFlowCheckMd04` (material y fecha) y
  `getManipuladorFlowOrderDetail` (orden fija con sus SolPed).
- **Variable logística**: `getVariableLogisticaByProductCode` devuelve la
  ficha en PDF.

## Buenas prácticas

- Filtra en SAP, no en la respuesta: pide la página o el rango de fechas
  concreto en vez de descargar todo.
- Las claves SAP llevan ceros a la izquierda (cliente `0000100001`, material
  de 18 posiciones en algunas tablas). Si no encuentras algo, prueba con el
  formato interno.
- Estas tools solo leen. Para crear o modificar datos hace falta la
  capacidad `sap_operations`.
