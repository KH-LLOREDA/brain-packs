---
name: presenton_layouts
description: >-
  Cómo crear y retocar un deck en Presenton: generar una vez, descubrir antes
  de editar, cambiar lo mínimo con las tools presenton_deck_*, charts, tablas,
  infográficos, imágenes y export.
metadata:
  category: media
  display-name: "Presenton: crear y retocar decks"
---

# Presenton — crear y retocar decks

Trabajas directamente sobre el deck con las tools `presenton_deck_*`. Presenton
no razona: tú decides qué tocar. El usuario ve el resultado en el editor
(embebido en Brain o abierto en Presenton).

## Crear ≠ retocar

- **Crear:** `presenton_create_presentation` **una sola vez** con el brief
  (`content`, `n_slides`, `language`, `tone`, `instructions`). El artefacto
  `handler=presenton` se registra y se abre solo, en la pantalla de esquema
  con la plantilla ya aplicada: el usuario revisa o edita el outline, puede
  cambiar la plantilla (*Change template*) y pulsa *Generate*. No hagas poll
  ni esperes: dile que revise el esquema en el editor. No inventes enlaces.
- **Plantilla:** `template` es opcional. Pásala solo si el usuario nombra una
  o el tema encaja claramente con una (p. ej. una corporativa para un informe
  interno): consulta `presenton_template_list` **una vez** (nombre y
  descripción) y pasa el `name` o el `id`. Si no, omítela: se aplica la
  plantilla por defecto de la instancia. Un nombre desconocido devuelve 400;
  no reintentes a ciegas, vuelve a mirar la lista.
- **Generar sin revisar** (solo si lo pide expresamente):
  `presenton_generate_presentation_async`. Tarda varios minutos; el editor
  muestra "Generating…" y abre el deck al terminar. Como mucho un poll con
  `presenton_get_presentation_generation_status`; no lo repitas en bucle.
- **Retocar:** nunca regeneres el deck para cambiar un titular, un dato o un
  color. Usa las tools `presenton_deck_*` sobre el `external_id` del artefacto
  (`presentation_id`).
- **Export:** `presenton_export_presentation` (PPTX/PDF), solo si lo piden.

## Dos tipos de deck

| `generation_mode` | Qué es | Tools |
|---|---|---|
| `standard` (por defecto) | Template V2: slides JSON con componentes y elementos | `presenton_deck_*` |
| `smart` | Cada slide es un fragmento HTML completo | `presenton_deck_smart_*` |

Si una tool responde **409**, el deck es del otro tipo: el mensaje dice qué
operación usar. Para crear, usa `standard` salvo que el brief pida lo contrario.

## Reglas comunes

- Slide N del usuario = `index` N-1. Al contestar, habla en números de slide.
- Pasa siempre `presentation_id` y, en las tools que editan, `user_request`
  con el pedido literal del usuario (algunas tools infieren de ahí el título
  de un chart o los datos de una tabla).
- Las respuestas vienen como `{ok, result}` o `{ok: false, error, recovery}`.
  Un cambio solo está hecho si `ok` es `true` y el resultado dice saved, added,
  updated o deleted. Si falla, sigue la `recovery`; no repitas el mismo JSON.
- Lo que devuelven las tools en este turno manda sobre lo que recuerdes.
- No inventes datos, URLs de imagen ni nombres de tema.

## Deck estándar: descubrir antes de editar

1. `presenton_deck_get_template_summary` antes de elegir layout o un cambio
   amplio de deck.
2. `presenton_deck_search_slide` si el usuario cita un tema o un texto pero no
   un número de slide.
3. `presenton_deck_get_slide_at_index` **antes de cualquier cambio visible**:
   da los ids de componente y los `elementPath`. Usa
   `includeFullContent=true` si necesitas el JSON exacto.
4. `presenton_deck_read_source_documents` si piden algo del PDF/documento
   subido al deck.

## Deck estándar: cambiar lo mínimo

- **Texto, estilo, geometría** de un elemento → `presenton_deck_update_element`
  (font, color, alignment, fill, stroke, position, size, chart, tableCell…).
- **Mover, redimensionar, duplicar, capas, agrupar** un bloque entero →
  `presenton_deck_update_component`.
- **Añadir** → antes, `presenton_deck_get_available_blocks` para reutilizar un
  bloque con el estilo de la plantilla; luego `presenton_deck_add_component`
  con el JSON del bloque adaptado. `presenton_deck_add_element` solo si no hay
  bloque adecuado.
- **Borrar** (`delete_element`, `delete_component`, `delete_slide`) solo si lo
  piden o si sustituyes exactamente ese elemento.
- Añadir es aditivo: conserva charts, tablas, imágenes y textos existentes.
- Todo dentro del lienzo 1280×720.

## Slides completas

- Slide nueva con layout: `presenton_deck_get_available_layouts` →
  `presenton_deck_get_content_schema_from_layout_id` →
  `presenton_deck_add_new_slide_layout` con el **contenido final** de cada
  campo (nada de placeholders).
- Slide en blanco: `presenton_deck_add_new_slide`, y se compone con bloques.
- `save_slide` / `update_slide` solo para reemplazar el contenido entero de una
  slide; no para retoques pequeños.

## Charts

- Chart real, nunca una imagen. Respeta etiquetas y números del usuario.
- Modelo: `chartType`, `title`, `categories`, `series[].values`, `colors`,
  ejes, `dataLabels` (null, base, mid, top, outside) y leyenda.
- Tipos: bar, horizontal_bar, stacked_bar, horizontal_stacked_bar, line, area,
  pie, donut, scatter, bubble, radar, polar_area, ibcs_kpi.
- Cada `series.values` tan largo como `categories`. Pie/donut: una serie.
- Sin colores del usuario, omite `colors` y se aplica el tema.
- Cambiar un chart existente: `presenton_deck_update_element` con `chart`.

## Tablas

- Tabla real (`type="table"` con columnas/cabeceras y filas), con los valores
  exactos del usuario. Nunca un shell vacío si el pedido trae datos.

## Infográficos y formas

- `presenton_deck_get_available_infographics` primero; después
  `presenton_deck_add_infographic` con el tipo y los datos exactos del catálogo.
- Líneas y formas: siempre `type="vector"` (los puntos definen la geometría).
- Ecuaciones: texto con `<latex>…</latex>`.

## Imágenes y vídeo (motor de Presenton)

Se generan **con Presenton**, no con el hub de Brain (`generate_image` no sirve
aquí: su URL no existe para Presenton y la slide queda rota). Lo generado se
guarda en la galería del usuario (diálogo *Change image → Your images*).

- **Imagen:** `presenton_deck_generate_assets` (smart:
  `presenton_deck_smart_generate_assets`) con
  `assets=[{kind:"image", prompt:"…"}]`. Prompt descriptivo y visual (sujeto,
  estilo, luz, encuadre), sin texto dentro de la imagen. Usa **solo** la `url`
  devuelta (`/app_data/...`); nunca construyas ni inventes rutas.
- **Insertar:** `presenton_deck_update_element` sobre la imagen existente
  (cambia `data`) o `add_element`/`add_component` con `type="image"` y esa URL.
- **Iconos:** mismo tool con `kind:"icon"` (busca en la librería de iconos).
- **Modelo:** por defecto `model: null` (modelo por defecto de Presenton). Solo
  si el usuario nombra uno, pásalo en el asset (`model`) o en
  `animate_image_async` (`model`). Tiene que estar habilitado en Settings
  (Image/Video Provider); si no, la tool responde 400 con la lista válida:
  usa una de esas o el default, no insistas.
- **Vídeo (animar una imagen):** primero ten la imagen en la slide (paso
  anterior). Luego `presenton_animate_image_async` con `image_url` (la URL de
  la galería), `prompt` del movimiento, `presentation_id` y `slide_index`.
  Tarda minutos: haz como mucho un poll con
  `presenton_get_presentation_generation_status(id=<task id>)`; si ya está
  `completed`, cambia el elemento con `presenton_deck_update_element` a
  `{type:"video", src:data.file_url, provider:"file", poster:<url imagen>,
  autoplay:true, loop:true, muted:true}` manteniendo posición y tamaño. Si
  sigue `pending`, dile al usuario que el vídeo se está generando y no hagas
  bucle. Un 400 "not configured" significa que la instancia no tiene vídeo.

## Tema y outline

- `presenton_deck_get_presentation_theme` / `set_presentation_theme` solo si el
  usuario pide cambiar el tema. Nunca como efecto lateral.
- Outline (antes de generar slides): `presenton_deck_add_outline`,
  `update_outline`, `delete_outline`.

## Deck Smart (HTML)

1. `presenton_deck_smart_get_smart_presentation_context` para cambios de deck,
   de estilo o slides nuevas.
2. `presenton_deck_smart_get_slide_at_index` con `includeFullContent=true`
   antes de editar: ese HTML manda.
3. `presenton_deck_smart_save_slide` con **un fragmento HTML completo**
   (`replaceOldSlideAtIndex=true` para editar, `false` para insertar). Raíz:
   un `<section>` con `relative h-[720px] w-[1280px] overflow-hidden`.
4. Conserva scripts, canvas de Chart.js, URLs, tipografía y paleta. Cada chart
   necesita su `<canvas>` con id único y su script `new Chart(...)` inline.
5. Nada fuera del lienzo, sin scroll ni texto recortado.

## Pedidos que llegan desde el editor

Cuando el turno trae `presentation_id` y la slide visible (o la selección), el
usuario está mirando el deck: lee esa slide y retoca. No regeneres.

## Al terminar

Una o dos frases: qué cambió y en qué slide. Si algo bloqueó, di exactamente
qué falta.
