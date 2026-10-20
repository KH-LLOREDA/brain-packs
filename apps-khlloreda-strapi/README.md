# Pack `apps-khlloreda-strapi`

Consulta de **solo lectura** a Producteca, a los **fichajes y horas propias** de GAD y al **menú de comida** (hoy o la semana) vía Strapi.
No expone fichajes ni menús de terceros, empresas, empleados externos, CAE ni directorio interno.

| Pieza | Dónde |
|---|---|
| Tools `khlloreda_*` | Core del engine (`services/engine/src/tools/domains/khlloreda`) |
| Capability `khlloreda_apps` | este pack |
| Subagente `khlloreda_assistant` | este pack (`source='pack'`) |
| Conexión `khlloreda-strapi` | plantilla; el token **no** viaja en git |

## Instalar en local

1. `PACKS_ALLOW_LOCAL_DIR=true` en `.env` (el compose ya monta `./packs-dev` → `/packs-dev`).
2. Recrear/reiniciar `api`.
3. GUI › Configuración › Packs › **Instalar carpeta** → `/packs-dev/apps-khlloreda-strapi`
   (o `POST /api/v1/packs/install-dir`).
4. Si no está `STRAPI_KHLLOREDA_TOKEN` en el entorno, rellena el token de **solo lectura** en Conexiones OpenAPI (`khlloreda-strapi`).
5. Fichajes/horas (`/api/accesos-horas`, `/api/lista-general`) van **sin** API token (con token dan 500). No hace falta usuario GAD en el engine.
6. Desde Docker, `strapi.khlloreda.com` debe resolver a la LAN (p.ej. `extra_hosts` en `docker-compose.override.yml`).

Publicación posterior: copiar este directorio al monorepo `brain-packs`.
