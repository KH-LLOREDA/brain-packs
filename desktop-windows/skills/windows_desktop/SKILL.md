---
name: windows_desktop
description: >-
  Operar un escritorio Windows con Windows-MCP: abrir apps, localizar
  controles por el árbol UIA y actuar sobre ellos. No cubre SAP GUI.
metadata:
  category: infra
  agent: windows_operator
  display-name: "Escritorio Windows"
---

# Escritorio Windows

El servidor es Windows-MCP en la sesión interactiva del PC. Brain habla con él
por HTTP y registra cada tool como `mcp_windows_<Nombre>`. Tú las llamas por
su nombre corto (`Snapshot`, `Click`, `App`).

## Orden

1. `App` abre o enfoca. El nombre es el del menú Inicio.
2. `Snapshot` devuelve ids de controles. Trabaja con esos ids.
3. `Click` / `Type` / `MultiEdit` sobre el id. `Shortcut` para atajos
   (`Ctrl+S`, `Alt+F4`).
4. Confirma con otro `Snapshot`. `WaitFor` si esperas un texto o una ventana.
5. Entrega el dato pedido. El recorrido de clics no es el resultado.

`Screenshot` es una foto. No sustituye al Snapshot para decidir dónde pulsar.

## Qué no hacer

- SAP GUI: no abras el Logon ni navegues transacciones. Eso es otro agente.
- Coordenadas a ciegas. Si el control no está en el árbol, cambia de ventana
  o acota `region`; no recorras la pantalla a saltos.
- `PowerShell`, `FileSystem` y `Registry` solo con un pedido explícito.
  No borres ficheros, no edites el registro y no mates procesos para
  «desbloquear» una ventana.
- No bloquees la sesión ni apagues el equipo.
