---
name: abap_development
description: >-
  Programar ABAP en el sistema SAP de desarrollo con las tools abap_* (ADT):
  leer, escribir con comprobación de sintaxis, activar, ABAP Unit y órdenes de
  transporte, con el usuario SAP de cada persona.
metadata:
  category: infra
  display-name: "Desarrollo ABAP"
---

# Desarrollo ABAP

Las tools `abap_*` hablan con SAP por ADT, la misma API que usan Eclipse y VS
Code. Trabajan con el usuario SAP que la persona tiene guardado en Brain: SAP
la registra como autora de cada cambio y aplica sus autorizaciones. Solo hay
acceso a los sistemas de desarrollo que el proxy permite; producción nunca.

## Flujo

1. **Localizar.** `abap_search("ZFI_*")` devuelve tipo, paquete y URI.
2. **Leer.** `abap_read` devuelve 400 líneas por defecto y el total. En
   objetos grandes lee por tramos (`start_line`) solo lo que necesites.
3. **Cambiar.** `abap_write` con `edits` `[{old, new}]` para tocar partes;
   cada `old` debe aparecer una sola vez, así que incluye contexto suficiente.
   Usa `source` (código completo) solo para objetos nuevos o pequeños.
4. **Corregir.** El resultado de `abap_write` trae `syntaxOk`, y cada error con
   su `line` y el texto de SAP. Corrige y vuelve a escribir hasta que
   `syntaxOk` sea `true`. No actives con errores.
5. **Activar.** `abap_activate` con todos los objetos que dependen entre sí a
   la vez (clase e interfaz, programa e includes).
6. **Probar.** Si el objeto tiene tests, `abap_run_unit_tests`. Un fallo trae
   el valor esperado y el real.
7. **Informar.** Di qué objetos cambiaste, en qué orden de transporte y el
   resultado de activación y tests.

## Objetos

| Tipo      | Qué es                                      | Notas                          |
|-----------|---------------------------------------------|--------------------------------|
| `PROG/P`  | Programa (report)                           |                                |
| `PROG/I`  | Include                                     |                                |
| `CLAS/OC` | Clase                                       | `include`: main, testclasses…  |
| `INTF/OI` | Interfaz                                    |                                |
| `FUGR/F`  | Grupo de funciones                          | Sin código propio              |
| `FUGR/FF` | Módulo de funciones                         | Necesita `group`               |
| `FUGR/I`  | Include de grupo de funciones               | Necesita `group`               |
| `DDLS/DF` | Vista CDS                                   |                                |

Los tests ABAP Unit de una clase van en `include: "testclasses"`; si no
existe, `abap_write` lo crea. Clases de test con `FOR TESTING DURATION SHORT
RISK LEVEL HARMLESS`.

## Paquetes y transportes

- Objetos de prueba o temporales: paquete `$TMP` (local, sin transporte).
- Nombres en el espacio del cliente: `Z*` o `Y*`.
- Fuera de `$TMP`, antes de crear o escribir llama a
  `abap_transport(action="check")`. Si el objeto ya está en una orden
  (`lockedIn`), se usa esa. Si no, propón al usuario una de `requests` o
  pide permiso para crear una con `action="create"` y un texto claro.
- No liberes órdenes: eso lo hace una persona.

## Límites

- `abap_delete` es irreversible: solo si el usuario lo pide expresamente.
- Tablas, elementos de datos y dominios (SE11), mensajes (SE91), vistas de
  mantenimiento (SM30), ejecutar un report con pantalla de selección o
  depurar no van por estas tools: usa la capacidad `sap_gui`.
- Si una tool responde que el sistema no está permitido o es de solo
  lectura, no busques otra vía: díselo al usuario.
