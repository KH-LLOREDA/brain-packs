---
name: sap_gui_operation
description: >-
  Operar SAP GUI for Windows por scripting (IDs de elementos, no píxeles):
  transacciones, formularios, grids, listas, menús, popups y exportes al
  workspace. Para el agente principal o sap_s4_operator.
metadata:
  category: infra
  agent: sap_s4_operator
  display-name: "Operar SAP GUI"
---

# Operar SAP GUI

Controlas SAP GUI for Windows con las tools MCP de scripting (`mcp_sap_*`).
Interactúas con la interfaz por el ID de cada elemento SAP GUI, no por
coordenadas de píxel.

## Entorno

- SAP GUI for Windows con la SAP GUI Scripting API nativa (COM). Scripting habilitado en cliente y servidor.
- Los elementos se identifican por su ID SAP GUI (ej: `wnd[0]/usr/ctxtSUID_ST_BNAME-BNAME`), NO por posición de píxel.
- El idioma del sistema SAP es **ES** (español), pero SAP mezcla etiquetas en inglés (en SE38 subir un fuente es «Utilidades > Más utilidades > Upload/Download > Upload...»). No adivines el texto de un menú: léelo con `sap_get_menus(contains=…)`.
- Brain inyecta las credenciales del usuario con `sap_login` al asignar la VM. Si al empezar **no hay sesión** (error COM de índice/enumerator, o `sap_session_info` vacío), llama **tú** a `sap_login` y espera. No pulses «Acceder al sistema» ni «Abrir» en el SAP Logon pad: eso no abre una sesión scriptable.

## Windows en la misma VM

La VM SAP también corre Windows-MCP. Al cargar `sap_gui` tienes sus tools
`mcp_windows_*` sobre **la misma máquina** que SAP GUI:

- **Meter un fichero de texto** en la VM (un fuente ABAP, un CSV pequeño): `mcp_windows_FileSystem` en modo escritura, en `C:\brain\imports\`. Luego en SAP apunta a esa ruta.
- **Comandos** (listar, copiar, descomprimir, descargar con `Invoke-WebRequest` desde una URL accesible): `mcp_windows_PowerShell`.
- **Sacar** ficheros al workspace sigue siendo `sap_list_files` + `import_sap_file`.
- No pases por el contexto ficheros grandes o binarios (Excel, ZIP): si no hay una URL desde la que la VM pueda descargarlos, **para y explica qué falta**.

## Límites

- `sap_win_keys`/`sap_win_click` son solo para diálogos nativos que bloquean a SAP. No los uses para abrir «Ejecutar», PowerShell o el explorador: para eso está `mcp_windows_*`.
- No uses los clics ni las teclas de Windows-MCP (`Click`, `Type`, `Shortcut`) dentro de SAP GUI: SAP se opera por scripting (`mcp_sap_*`), que no depende de qué ventana esté delante. Las teclas de Windows van a la ventana en primer plano, que puede ser el SAP Logon pad.
- **Nunca cierres la ventana «SAP Logon»**: cerrarla termina todas las sesiones y se pierde lo no guardado. Si se pone delante, al scripting le da igual.
- Lee el final de cada resultado: las acciones informan de las ventanas que aparecen o se cierran y de a qué ventana fue el foco. Si una tecla o clic acabó en una ventana que no esperabas, para y corrige antes de seguir.
- Si un menú o botón aparece desactivado, comprueba el modo de la pantalla (visualizar vs. modificar) antes de buscar caminos alternativos.

## Herramientas disponibles

### Sesión
- **sap_login(client?, user?, password?, language?)**: Abre la conexión guardada de la VM e inicia sesión. Brain suele llamarla al asignar la máquina; si no hay sesión, es tu primer paso. No hace falta host: el sistema destino es el de la VM.
- **sap_logoff()**: Cierra la sesión (`/nex`) para dejar la VM limpia.

### Navegación
- **sap_navigate(transaction)**: Navega a un código de transacción SAP (ej: "MM03", "VA01", "SE16N"). Usa prefijo `/n` internamente.

### Percepción (descubrir la pantalla)
- **sap_session_info()**: Contexto barato: sistema, mandante, usuario, idioma y transacción/programa/pantalla actuales. Úsala para orientarte (confirmar en qué sistema/mandante estás) antes de una acción sensible.
- **sap_get_window()**: Obtiene título de ventana activa y texto de barra de estado. Solo necesaria cuando quieras comprobar el título de ventana o no tengas feedback de mensaje (las acciones mutantes ya devuelven el mensaje SAP automáticamente).
- **sap_get_fields()**: Escanea todos los campos editables en pantalla. Devuelve IDs, tipos, valores actuales, labels y tooltips. Úsala al llegar a una pantalla o pestaña nueva para descubrir los IDs. No la repitas si ya conoces los IDs de la pantalla actual.
- **sap_get_buttons()**: Lista todos los botones visibles con sus IDs, textos y tooltips. Úsala cuando no sepas qué botones hay en una pantalla desconocida. Para transacciones habituales (Enter, Guardar, Ejecutar) usa directamente `sap_press_key`.
- **sap_get_tabs()**: Lista las pestañas visibles con sus IDs. No necesaria si los tabs ya aparecen en `sap_get_fields()` o ya conoces el nombre de la pestaña.
- **sap_find_element(id_pattern?, element_type?)**: Busca elementos por patrón parcial de ID o tipo. Devuelve ID, tipo, texto y tooltip de cada elemento encontrado.
- **sap_get_tree(max_depth?)**: Dump jerárquico completo del árbol GUI. Último recurso para depuración y descubrimiento de estructura de pantalla.

### Interacción con campos
- **sap_set_field(field_id, value)**: Establece un valor en un campo por su ID SAP GUI.
- **sap_fill_form(fields_json)**: Rellena múltiples campos de golpe. Acepta un JSON con pares ID→valor. Más eficiente que llamar `sap_set_field` repetidamente.
- **sap_click_button(button_id)**: Pulsa un botón por su ID.
- **sap_select_tab(tab_id)**: Selecciona una pestaña por su ID.
- **sap_press_key(vkey)**: Envía una tecla virtual a la ventana SAP.

### Leer datos y listas
- **sap_read_grid(grid_id?, start_row?, max_rows?)**: Lee datos de un **ALV grid** (tabla interactiva). Devuelve columnas, conteo de filas y datos de celdas como JSON. Si no se especifica grid_id, auto-descubre el primer grid. Para muchas filas, pagina con `start_row`/`max_rows`.
- **sap_read_list(full?, max_pages?)**: Lee una **lista ABAP clásica** (salida de informes financieros, SE16, listados de SU01… cuando la pantalla muestra filas de texto y NO un grid). Con `full=false` (por defecto) devuelve solo la página visible e info de scroll; con `full=true` pagina toda la lista y devuelve todas las filas (la cabecera reimpresa se deduplica automáticamente).
- **sap_modify_cell(row, column, value, grid_id?)**: Escribe un valor en una celda de ALV. Los nombres de columna son nombres técnicos SAP (ej: TXZ01, MENGE, EEIND).

### Menús
- **sap_get_menus(contains?)**: Lista las entradas seleccionables de la barra de menús con su id y ruta completa (ej: "Lista > Exportar > Hoja de cálculo"). `contains` filtra por texto de la ruta sin distinguir mayúsculas ni acentos (ej: `"upload"`, `"Utilidades"`).
- **sap_select_menu(path | menu_id)**: Dispara una entrada de menú por su ruta visible (`"Utilidades > Más utilidades > Upload/Download > Upload..."`, o solo la hoja) o por su id, sin abrir los menús padre. Si no encuentra la entrada devuelve las más parecidas: elige una de ellas en vez de inventar otra.

### Diálogos nativos de Windows (fuera de SAP)
El scripting SAP solo ve los dynpros SAP (`wnd[0]`, `wnd[1]`…). Los diálogos **nativos de Windows** (aviso de seguridad de scripting de SAP GUI al acceder a ficheros, diálogos abrir/guardar de Windows, message boxes, popups ActiveX/HTML) son invisibles para el scripting y pueden **bloquear** la llamada. Un watchdog interno los detecta y resuelve automáticamente (marca "Recordar mi decisión" y pulsa "Permitir"), así que normalmente NO tienes que hacer nada. Solo si una acción informa que sigue bloqueada:
- **sap_win_dialogs()**: Lista los diálogos nativos por encima de SAP con sus botones/checkboxes/texto.
- **sap_win_click(button?, title_contains?, handle?)**: Pulsa un botón de un diálogo nativo (ej: "Permitir", "Abrir"). Sin `button` envía Enter.
- **sap_win_keys(keys, title_contains?, handle?)**: Envía teclas a un diálogo nativo (sintaxis pywinauto: "{ENTER}", "{ESC}", rutas de fichero…). Sin `title_contains`/`handle` van al diálogo nativo de más arriba.
- Ambas devuelven `target` (a qué ventana fueron y por qué), `foreground` y las ventanas nuevas o cerradas. Comprueba que `target` es el diálogo que querías.

## Virtual Keys (vkeys) — referencia

| vkey | Tecla              |
|------|--------------------|
| 0    | Enter              |
| 2    | F2                 |
| 3    | F3 (Atrás)         |
| 4    | F4 (Matchcode)     |
| 5    | F5                 |
| 8    | F8 (Ejecutar)      |
| 11   | Ctrl+S (Guardar)   |
| 12   | Ctrl+F12 (Cancelar)|
| 27   | Ctrl+F3            |

## Flujo de trabajo

1. `sap_navigate("MM03")` → el resultado incluye el mensaje SAP si lo hay.
2. `sap_get_fields()` → IDs, tipos, valores, labels y tooltips.
3. Rellena con `sap_fill_form({"wnd[0]/usr/ctxtFIELD1": "valor1", "wnd[0]/usr/txtFIELD2": "valor2"})` (o `sap_set_field` para un solo campo).
4. Ejecuta o confirma: `sap_press_key(0)` Enter, `sap_press_key(8)` Ejecutar, `sap_press_key(11)` Guardar.
5. Lee el mensaje SAP devuelto para decidir el siguiente paso.

## Patrones de interacción

### Formulario con múltiples pestañas
1. Rellena los campos de la pestaña actual con `sap_fill_form(...)`
2. `sap_select_tab(...)` → cambia a la siguiente pestaña
3. `sap_get_fields()` → descubre los campos de la nueva pestaña
4. Repite hasta completar todas las pestañas necesarias
5. `sap_press_key(11)` para guardar al final

### Leer datos / informes
Tras navegar, llama `sap_read_screen()` (o mira `widgets` en `sap_get_window` / `sap_session_info`):
- `recommended=sap_read_grid` → ALV (`GuiShell/GridView` o `GuiGridView`). SM04, SM50, SE16N…
- `recommended=sap_read_table` → `GuiTableControl`
- `recommended=sap_read_list` → lista ABAP clásica
- `recommended=sap_get_fields` → formulario
- `widgets.unknown` → control que aún no leemos (anota type+subtype+id; no inventes un reader)

No uses `vnc_screenshot` en la VM del pool. `sap_get_tree()` parte de `/usr`.

### Traspasar ficheros exportados al workspace (descargas)
Los ficheros que SAP genera (export a XLSX/TXT/PDF/CSV vía `%pc` → "Fichero local", o descargas) quedan en el disco de la VM; el usuario NO puede descargarlos hasta que los traigas al workspace:
1. Exporta desde SAP. En el diálogo de exportación, escribe el **directorio** `C:\brain\exports` y un nombre de fichero claro. El aviso de seguridad nativo lo resuelve el watchdog automáticamente.
2. **sap_list_files(directory?)**: localiza el fichero (sin argumentos escanea `C:\brain\exports` + carpetas habituales, más recientes primero). Copia la `path` absoluta.
3. **import_sap_file(path, name?)**: trae el fichero al workspace de Brain como **artefacto descargable**. Informa al usuario de que ya lo tiene en el panel de artefactos.

Para datos tabulares que solo necesitas *leer/analizar*, `sap_read_grid`/`sap_read_list` suele bastar y evita el ciclo de export.

### Modificar celdas de ALV Grid
1. `sap_read_grid()` → identifica nombres de columnas
2. `sap_modify_cell(row=0, column="TXZ01", value="Texto nuevo")`
3. `sap_press_key(0)` para confirmar cambios

### Descubrir estructura de pantalla desconocida
1. `sap_get_fields()` para campos editables (incluye labels y tooltips)
2. `sap_get_buttons()` si necesitas descubrir botones disponibles
3. `sap_get_tabs()` si necesitas pestañas no obvias
4. `sap_get_tree()` como último recurso para dump completo

## Navegación SAP — referencia rápida

### Barra de comandos (OKCode)
- `/nXXXX` — Abre transacción XXXX en la sesión actual
- `/oXXXX` — Abre transacción XXXX en nueva sesión
- `/n` — Vuelve al menú principal
- `/nend` — Cierra la sesión
- `/nex` — Cierra sin confirmar

### Transacciones comunes

**Logística / Materiales (MM)**: MM01 crear material · MM02 modificar · MM03 visualizar · ME21N crear pedido de compras · ME23N visualizar pedido · ME51N crear solicitud de pedido · MIGO movimiento de mercancías · MB52 stocks en almacén.

**Ventas (SD)**: VA01 crear pedido de ventas · VA03 visualizar pedido · VL01N crear entrega · VF01 crear factura.

**Finanzas (FI)**: FB03 visualizar documento contable · FBL1N partidas de acreedor · FBL3N partidas de cuenta de mayor · FBL5N partidas de deudor · FS10N balance de cuentas.

**Controlling (CO)**: KSB1 partidas individuales de centro de coste · KE5Z P&L.

**Herramientas / Desarrollo**: SE16 / SE16N navegador de datos · SM37 monitorización de jobs · SU01 gestión de usuarios · ST05 SQL Trace · SE11 diccionario ABAP · SM04 / AL08 usuarios conectados.

## Mensajes de SAP

Cada acción mutante (set_field, click_button, select_tab, press_key, fill_form, navigate) devuelve el mensaje de la barra de estado con su tipo:
- **SAP [OK]**: Operación exitosa (ej: "El usuario X ha sido creado")
- **SAP [ERROR]**: Error de validación (ej: "Longitud clave acceso insuficiente")
- **SAP [Warning]**: Aviso que puede requerir acción
- **SAP [Info]**: Mensaje informativo

**Lee SIEMPRE el mensaje** incluido en la respuesta de cada acción. Si hay un error, corrige la causa antes de continuar. No llames a `sap_get_window()` solo para leer el mensaje.

## Diálogos modales / popups — CRÍTICO

Cuando SAP abre una ventana modal (confirmaciones, avisos, errores de validación, petición de clave inicial, campos obligatorios…), la respuesta de la acción incluye un bloque **`POPUP «título»: texto`** con sus botones:

- **Nunca ignores un POPUP ni reintentes la acción anterior mientras esté abierto.** Un popup abierto bloquea la pantalla principal.
- Si pide datos, `sap_get_fields()` devuelve **los campos del propio popup** (la percepción apunta a la ventana frontal); rellénalos y confirma. Si solo hay que decidir, pulsa el botón adecuado (`sap_click_button` con el ID del popup, `sap_press_key(0)` para "Continuar"/"Sí", `sap_press_key(12)` para "Cancelar").
- Tras cerrar el popup, vuelve a leer el mensaje/estado para confirmar el resultado real.

## Regla anti-bucle — CRÍTICO

Si una misma acción falla **dos veces** (mismo error o mismo popup sin resolver), NO la repitas una tercera vez. Llama a `sap_get_window()` + `sap_get_fields()` para entender el estado real y, si sigues bloqueado, informa del mensaje/popup exacto en lugar de seguir intentando. No cambies datos de negocio (p. ej. el nombre de usuario) para "esquivar" un error: corrige la causa raíz.

## Manejo de errores y timeouts

Si una acción devuelve `timeout`:
1. `sap_press_key(12)` — Cancelar/Escape
2. `sap_press_key(0)` — Enter para aceptar un posible diálogo
3. `sap_win_dialogs()` — por si hay un diálogo nativo bloqueando
4. `sap_navigate("/n")` — volver al menú como último recurso

**Diálogos SAP comunes:** información/aviso → Enter o "Continuar"; confirmación → "Sí"/"Aceptar"; error de autorización → informa del error exacto.

## Campos de password

Los campos de contraseña (`GuiPasswordField`) aparecen en `sap_get_fields()` con `type: "GuiPasswordField"`. Rellena ambos campos (password + confirmación) con `sap_fill_form(...)` en una sola llamada. La contraseña debe cumplir la política de SAP (mínimo 10 caracteres típicamente, con mayúsculas, minúsculas, dígitos y símbolos).

## Crear un usuario en SU01 — clave inicial OBLIGATORIA

1. `sap_navigate("SU01")`, escribe el nombre en `...SUID_ST_BNAME-BNAME` y pulsa "Crear".
2. En la pestaña **Datos de acceso**: `sap_get_fields()` para localizar `...PASSWORD` y `...PASSWORD2`, y rellénalos AMBOS con la misma clave válida mediante `sap_fill_form(...)`.
3. Rellena el resto de datos obligatorios y asigna perfiles/roles en su pestaña.
4. Guarda con `sap_press_key(11)`.
5. Verifica el mensaje SAP (`[OK]` "Usuario … creado"). Si aparece un POPUP o `[ERROR]`, resuélvelo — NO reintentes con otro nombre de usuario.

## Principios de eficiencia

- **Minimiza llamadas**: agrupa campos con `sap_fill_form`; usa `sap_press_key` para acciones conocidas.
- **No repitas percepción innecesaria**: si ya conoces los IDs de la pantalla, no vuelvas a llamar `sap_get_fields()`.
- **Pagina las lecturas grandes** (`start_row`/`max_rows`, `full=false`): cada resultado ocupa contexto.

## Reglas de seguridad

- **NUNCA** ejecutes transacciones que borren datos en producción sin confirmación explícita del usuario.
- **NUNCA** cambies configuración de sistema (SPRO, SM30) sin autorización.
- Si aparece un popup o diálogo inesperado, inspecciónalo (`sap_get_window()`, `sap_find_element()`) antes de actuar.
- Si el sistema pide cambio de contraseña, informa al usuario.
