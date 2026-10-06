---
name: windows_desktop
description: >-
  Operar un escritorio Windows con Windows-MCP: abrir apps, pulsar por el
  árbol UIA y, si el control no existe, por un punto de la misma captura.
  No cubre SAP GUI.
metadata:
  category: infra
  agent: windows_operator
  display-name: "Escritorio Windows"
---

# Escritorio Windows

El servidor es Windows-MCP en la sesión interactiva del PC. Brain registra
cada tool como `mcp_windows_<Nombre>`. Llámalas por su nombre corto
(`Snapshot`, `Click`, `App`, `Screenshot`, `DisplayInventory`, `Shortcut`,
`Type`, `WaitFor`).

Si este hilo ya es el escritorio, llama tú las tools. No pases cada paso a
`windows_operator`: ese agente nace sin la captura anterior.

## Orden

1. `App` abre o enfoca. El nombre es el del menú Inicio.
2. `Snapshot`. Si el control tiene id, `Click` o `Type` sobre ese id.
   `Shortcut` solo para un atajo real (`Ctrl+S`, `Alt+F4`).
3. Tras un cambio de pantalla, otro `Snapshot`. `WaitFor` si la ventana tarda.
4. Entrega el dato pedido. El recorrido de clics no es el resultado.

`Scrape` lee páginas web. No lee la pantalla.

## Si el control no está en el árbol

No recorras la pantalla y no mezcles tamaños. Hay tres rejillas distintas: el
escritorio (`DisplayInventory`), el tamaño que anuncia el programa y la foto.
`Click` solo acierta en la primera.

1. `DisplayInventory` una vez. Ese es el único tamaño de pantalla que cuenta.
2. `Screenshot` del rectángulo de la ventana, en píxeles del escritorio
   virtual (`region=[left, top, right, bottom]`). No uses el tamaño interno
   del programa como `region`.
3. Señala el punto en esa foto. Conversión fija, una sola vez:

   `x = x_foto / ancho_foto * ancho_region + left`

   `y = y_foto / alto_foto * alto_region + top`

   `ancho_foto` y `alto_foto` son los de la imagen devuelta. Si la foto va
   reducida, esa división deshace la escala.
4. Un `Click` con ese `loc`. Otra `Screenshot`. Si la pantalla no cambió, un
   solo ajuste y para. Di que no se pudo pulsar. No sigas inventando números.

`Snapshot` con anotaciones dibuja cajas de controles que ya están en el árbol.
No inventa un botón dibujado por el programa.

## Qué no hacer

- SAP GUI: no abras el Logon ni navegues transacciones. Eso es otro agente.
- `PowerShell`, `FileSystem` y `Registry` solo con un pedido explícito.
  No borres ficheros, no edites el registro y no mates procesos para
  «desbloquear» una ventana.
- No bloquees la sesión ni apagues el equipo.
