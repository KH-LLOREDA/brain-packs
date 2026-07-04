# Brain Packs

Un **pack** es un bundle versionado y declarativo que agrupa **agentes,
capabilities, skills y plantillas de conexión** de un dominio o entorno. Vive en
git y se instala/desinstala **por entorno** con trazabilidad (`source='pack'` +
`pack_id` en cada entidad), de modo que desinstalar un pack poda exactamente lo
que creó.

## Por qué

La configuración de agentes/capacidades/skills/herramientas ya no tiene por qué
estar toda predefinida en el código del Engine. Lo específico de un entorno
(p. ej. `proxy-portainer`/`proxy-proxmox`, la infra de kh7 con Vaultwarden, o el
DNS de khlloreda) se empaqueta a parte y cada entorno instala solo lo que usa.

## Instalar un pack

Desde la GUI (admin) o vía API/tools del propio Brain:

```
POST /api/v1/packs/install
{ "repo_url": "https://github.com/KH-LLOREDA/brain-packs.git", "ref": "main", "subdir": "infra-portainer-proxmox" }
```

o pídeselo a Brain (agente basis, capability core `pack_management`):
`pack_install(subdir="kh7-infra")`  (usa `repo_url`/`ref` por defecto de `packs.*`).

> En entornos aislados (p. ej. kh7) usa un **mirror git interno** como `repo_url`.
> Los packs viven en la **raíz** de este repo, por lo que `subdir` es el id del pack.

## Estructura de un pack

```
<pack>/
  pack.yaml                     # manifest: id, name, version, description, requires
  capabilities/*.yaml           # capabilities (dict o lista de dicts)
  agents/*.yaml                 # PATCHES a agentes existentes (no los reemplaza)
  skills/<name>/SKILL.md        # skills (frontmatter YAML + cuerpo markdown)
  connections/
    openapi/*.yaml              # plantilla de conexión OpenAPI (SIN secreto)
    mcp/*.yaml                  # plantilla de conexión MCP
```

### `pack.yaml`

```yaml
id: infra-portainer-proxmox        # kebab/snake, único
name: "Infra: Portainer & Proxmox"
version: 1.0.0
description: "..."
requires: []                       # ids de otros packs
```

### Agent patch (`agents/*.yaml`)

No redefine el agente; le **añade** capabilities/skills/tools:

```yaml
agent_id: basis_agent
add_default_capabilities: [portainer_read, proxmox_read]
add_allowed_capabilities: [portainer_admin, proxmox_admin]
add_domain_tools: []               # ⚠️ en agentes 'core' no persiste (usa capabilities)
add_skills: []                     # slugs de skills del propio pack
```

### Connection template (`connections/openapi/*.yaml`)

Los **secretos no viajan** en el pack: se declara `auth_token_env` y el token se
resuelve del entorno al instalar (si falta, la conexión se crea sin credencial y
se rellena en *Conexiones OpenAPI*).

```yaml
name: "Proxy Vaultwarden"
slug: proxy-vaultwarden
spec_url: "http://brain-proxy-vaultwarden:3001/openapi.json"
base_url: "https://proxy-vw.example.com"
auth_type: bearer                  # none | bearer | api_key | basic
auth_token_env: PROXY_VW_TOKEN
```

Tras instalar un pack con conexiones OpenAPI, sus herramientas se regeneran en
caliente (o reinicia el Engine).
