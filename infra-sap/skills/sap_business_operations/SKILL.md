---
name: sap_business_operations
description: >-
  Escribir datos de negocio en SAP con las tools proxy-sap_*: productos,
  condiciones de venta, órdenes de fabricación y pedidos de compra. Simular
  primero, confirmar y después ejecutar.
metadata:
  category: data
  display-name: "Operaciones SAP"
---

# Operaciones SAP

Estas tools cambian datos reales en SAP con el usuario SAP de la persona: SAP
registra el cambio a su nombre y aplica sus autorizaciones. Todas piden
aprobación antes de ejecutarse.

## Regla

1. **Lee antes.** Consulta el estado actual (`getProductsByProductId`,
   `getManipuladorFlowCheckMd04`, `getManipuladorFlowOrderDetail`,
   `getSalesConditionsInfo`).
2. **Simula.** Las operaciones de fabricación y compras aceptan `dryRun` y por
   defecto es `true`: devuelven lo que harían sin tocar nada. Enseña al
   usuario el resultado de la simulación.
3. **Confirma.** Solo con el visto bueno explícito del usuario repite la
   llamada con `dryRun: false`.
4. **Verifica.** Vuelve a leer y resume qué cambió (números de orden o
   pedido creados).

## Operaciones

- **Productos**: `postProducts` (crear), `patchProductsByProductId` (cambiar
  campos concretos; preferible) y `putProductsByProductId` (sustituir todo).
- **Condiciones de venta**: `postSalesConditionsCreate` crea o actualiza una
  condición (organización de ventas obligatoria; fechas `YYYYMMDD`). No tiene
  simulación: confirma los valores con el usuario antes.
- **Fabricación y compras** (Plan Integral de Manipuladores):
  `postManipuladorFlowReleaseOrder` convierte una orden planificada en orden
  fija liberada, `postManipuladorFlowCreatePurchaseOrder` crea el pedido desde
  la SolPed y `postManipuladorFlowReleasePurchaseOrder` lo libera.
  `postManipuladorFlowRunAll` encadena los tres pasos; con `dryRun: true` solo
  simula el primero.

## Límites

- No encadenes escrituras sin enseñar el resultado de cada una.
- Si SAP rechaza la operación, transmite el mensaje de SAP tal cual; no
  reintentes con otros valores sin preguntar.
- Si la respuesta dice que faltan las credenciales SAP, el usuario tiene que
  crear su conexión SAP en Mi perfil › Conexiones.
