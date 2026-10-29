---
name: presenton_layouts
description: >-
  Cómo generar e iterar un deck en Presenton: standard vs smart, layouts,
  export, y qué no tocar (HTML Reveal legado).
metadata:
  category: media
  agent: presenton_designer
  display-name: "Layouts Presenton"
---

# Presenton — layouts y flujo

## Dos modos de generación

| Modo | Cuándo | Qué produce |
|---|---|---|
| **standard** (default) | Control de layout, marca, iteración visual | JSON Template V2 + editor Konva |
| **smart** | Brief abierto, datos/charts, menos plantilla | HTML/Tailwind + Chart.js |

Si el usuario no pide lo contrario, usa **standard**.

## Crear ≠ retocar

- **Crear:** `presenton_generate_presentation` (o async + poll).
- **Retocar:** tools de update/export de Presenton sobre el `external_id` del artefacto. No regeneres el deck para cambiar un titular.

## Artefacto

El deck vive en Presenton. Brain guarda un puntero:

- `handler: presenton`
- `existing_uri: presenton://deck/<id>`
- `mime: application/vnd.brain.presentation+json`

No conviertas eso a `presentations/*.html` de Reveal.

## Export

PPTX y PDF salen de Presenton (`presenton_export_presentation` o la acción del handler). No uses `POST /artifacts/{id}/export/pptx` del core (eso es HTML legado).

## HTML legado (`slides-html`)

Decks antiguos en Reveal se abren con el handler builtin. No los migres a Presenton salvo que el usuario lo pida explícitamente (sería una generación nueva, no una conversión fiel).

## Imágenes

Portadas e ilustraciones: capability `image_generation` del hub Brain, luego pásalas a Presenton (`presenton_update_slide` / `presenton_edit_slide`). No dejes placeholders inventados.

## Pedidos desde el iframe

El generate del editor embebido **no** usa el LLM de Presenton. Llega como turno tuyo (chat de la mesa) con `presentation_id` / slide visible. Entonces:

1. `presenton_get_presentation` con ese id.
2. Cambio local: `presenton_edit_slide` (prompt) o `presenton_update_slide`.
3. Imagen: `generate_image` del hub y sustituye el asset en la slide.
4. No llames `presenton_generate_presentation` para un retoque.
