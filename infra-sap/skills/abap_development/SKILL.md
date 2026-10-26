---
name: abap_development
description: >-
  Leer y programar ABAP en SAP por ADT con las tools proxy-sap_abap*: buscar,
  leer, escribir con comprobación de sintaxis, activar, ABAP Unit y órdenes de
  transporte, con el usuario SAP de cada persona.
metadata:
  category: infra
  display-name: "Desarrollo ABAP"
---

# Desarrollo ABAP

Las tools `proxy-sap_abap*` hablan con SAP por ADT, la misma API que usan
Eclipse y VS Code. Cada llamada lleva la conexión SAP que la persona tiene en
Mi perfil: SAP la registra como autora de cada cambio y aplica sus
autorizaciones. Solo hay acceso a los sistemas que el proxy permite
(`abapSystems` los lista); solo los que tienen `write` admiten cambios.

## Flujo

1. **Localizar.** `abapSearch` con `query: "ZFI_*"` devuelve tipo, paquete y URI.
2. **Leer.** `abapRead` devuelve 400 líneas por defecto y el total. Si la
   respuesta trae `hint`, el objeto sigue: lee el siguiente tramo con
   `startLine` solo si lo necesitas.
3. **Cambiar.** `abapWrite` con `edits` `[{old, new}]` para tocar partes; cada
   `old` debe aparecer una sola vez, así que incluye contexto suficiente. Usa
   `source` (código completo) solo para objetos nuevos o pequeños.
4. **Corregir.** El resultado de `abapWrite` trae `syntaxOk` y cada error con
   su `line` y el texto de SAP. Corrige y vuelve a escribir hasta que
   `syntaxOk` sea `true`. No actives con errores.
5. **Activar.** `abapActivate` con todos los objetos que dependen entre sí a
   la vez (clase e interfaz, programa e includes). Pide aprobación.
6. **Probar.** Si el objeto tiene tests, `abapRunUnitTests`. Un fallo trae el
   valor esperado y el real.
7. **Informar.** Di qué objetos cambiaste, en qué orden de transporte y el
   resultado de activación y tests.

Para solo leer código de funciones o programas también sirven las tools
`proxy-sap_getAbapSource*` (por RFC).

## Objetos

Se identifican con `object: {type, name}` (o `{uri}` de la búsqueda).

| Tipo      | Qué es                         | Notas                         |
|-----------|--------------------------------|-------------------------------|
| `PROG/P`  | Programa (report)              |                               |
| `PROG/I`  | Include                        |                               |
| `CLAS/OC` | Clase                          | `include`: main, testclasses… |
| `INTF/OI` | Interfaz                       |                               |
| `FUGR/F`  | Grupo de funciones             | Sin código propio             |
| `FUGR/FF` | Módulo de funciones            | Necesita `group`              |
| `FUGR/I`  | Include de grupo de funciones  | Necesita `group`              |
| `DDLS/DF` | Vista CDS                      |                               |

Los tests ABAP Unit de una clase van en `include: "testclasses"`; si no
existe, `abapWrite` lo crea. Clases de test con `FOR TESTING DURATION SHORT
RISK LEVEL HARMLESS`.

## Paquetes y transportes

- Objetos de prueba o temporales: paquete `$TMP` (local, sin transporte).
- Nombres en el espacio del cliente: `Z*` o `Y*`.
- Fuera de `$TMP`, antes de crear o escribir llama a `abapTransportCheck`. Si
  el objeto ya está en una orden (`lockedIn`), usa esa. Si no, propón al
  usuario una de `requests` o pide permiso para crear una con
  `abapTransportCreate` y un texto claro.
- No liberes órdenes: eso lo hace una persona.

## Límites

- `abapDelete` es irreversible: solo si el usuario lo pide expresamente.
- Tablas, elementos de datos y dominios (SE11), mensajes (SE91), vistas de
  mantenimiento (SM30), ejecutar un report con pantalla de selección o
  depurar no van por estas tools: usa la capacidad `sap_gui` si está.
- Si una tool responde que el sistema no está permitido o es de solo
  lectura, no busques otra vía: díselo al usuario.
- Si responde que faltan las credenciales SAP, el usuario tiene que crear su
  conexión SAP en Mi perfil › Conexiones.
