---
name: khlloreda-apps-usage
description: Cómo consultar Producteca, fichajes/horas y el menú de hoy o de la semana (Strapi) en solo lectura.
metadata:
  display-name: "GAD, menú y Producteca (Strapi)"
  category: data
---

# Producteca, GAD fichajes y menú (solo lectura)

Backend compartido: Strapi (`khlloreda-strapi` en Conexiones OpenAPI).
Las tools son Python del engine (`khlloreda_*`), no se generan desde el spec OpenAPI.

## Producteca
- `khlloreda_buscar_productos(query?, pais?, tipo?, fabricante?)` — búsqueda.
- `khlloreda_ver_producto(id)` — ficha completa (GHS, eficacia, packaging).
- `khlloreda_buscar_mpqs(query?, tipo?, incluir_obsoletos?)` — materias primas; por defecto sin obsoletos.
- `khlloreda_listar_comparativas(estado?)` — solicitudes de comparación.

Tipos de producto: `bano`, `cocina`, `hogar`, `ropa`, `bbq`, `otros`.
País habitual: `España`.

## GAD (solo fichajes del usuario logueado)
- `khlloreda_fichajes(periodo?, desde?, hasta?)` — fichajes y horas (hoy, ayer, semana, mes o rango). No admite nombre.

## Comedor (solo el menú del usuario logueado)
- `khlloreda_menu_hoy()` — menú de hoy (primero, segundo, postre).
- `khlloreda_menu_semana()` — pedido de toda la semana (lun–jue). No admite email ajeno.

## Pautas
- Empieza por buscar; usa `ver_producto` solo para el detalle.
- No inventes datos. No vuelques JSON crudo.
- Si piden crear/editar/borrar: la integración es solo consulta.
- Si piden fichajes o menú de otra persona, empresas, CAE o directorio: rechaza con claridad.
- Error de conexión: suele faltar el token en Conexiones o DNS interno de `strapi.khlloreda.com`.
