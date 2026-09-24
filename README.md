# Brain Packs

Un **pack** es un bundle versionado y declarativo que agrupa **agentes,
capabilities, skills y plantillas de conexión** de un dominio o entorno. Vive en
git y se instala/desinstala **por entorno** con trazabilidad (`source='pack'` +
`pack_id` en cada entidad), de modo que desinstalar un pack poda exactamente lo
que creó.

## Por qué

La configuración de agentes/capacidades/skills/herramientas ya no tiene por qué
estar toda predefinida en el código del Engine. Lo específico de un entorno
(p. ej. `proxy-portainer` (pack `infra-portainer`), Proxmox autocontenido (pack
`infra-proxmox`), Vaultwarden (pack `infra-vaultwarden`), o el DNS de khlloreda) se
empaqueta a parte y cada entorno instala solo lo que usa.

## Catálogo

**Infra / entorno** (servicios, conexiones, VMs):

- `infra-biw` — proxy-biw + subagente `sap_analyst` (datos SAP BIW).
- `infra-sap-gui` — pool de VMs Windows + subagente `sap_s4_operator` (SAP GUI).
  Tile de Aplicaciones `sap` (no aparece en entornos sin este pack).
- `infra-portainer` / `infra-proxmox` — Docker (Portainer) y virtualización (Proxmox) para `basis_agent`.
- `infra-vaultwarden` — proxy-vaultwarden (credenciales) para `basis_agent`.
- `khlloreda-dns` — DNS de KH Lloreda.

**Aplicación** (agente + capability + skills; extraídos del core en v3 — sus tools
siguen en el core de Brain, el pack aporta el agente/capacidad/conocimiento):

- `office-m365` — `m365_assistant` + capability `m365_productivity` (Microsoft 365 / Graph).
- `office-mail` — visor de correo embebible (`handler=email`, agente `mail_viewer`).
  Runtime `mail-viewer`; tools `mail_viewer_*` del OpenAPI. No sustituye Graph.
- `data-databricks` — `databricks_analyst` + capability `databricks_query` (Databricks Lakehouse).
- `creative-blender` — `blender_3d_creator` (creación 3D en Blender vía MCP/VNC).
  Tile de Aplicaciones `blender`.
- `creative-slides` — presentación web (Reveal, autonomía alta). Capability
  `presentation_web` + plantilla «Presentación web». Tools `slides_*` en el core.
- `creative-presenton` — presentación colaborativa (autonomía media).
  `presenton_designer` + capability `presentation_presenton` + plantilla.
  Runtime Presenton; tools `presenton_*` del OpenAPI del servicio.
- `office-pptx` — presentación PowerPoint (autonomía baja). Capability
  `presentation_office` + plantilla. Tools del Bridge OnlyOffice en el core.
- `automation-brainflow` — `brainflow_manager` (automatizaciones BrainFlow, tools `bf_*`).

> Los agentes de investigación web y RAG **no** son packs: en v3 se resuelven como
> agentes dinámicos (`spawn_agent` + `load_capability("web_research"|"knowledge_base")`).

## Instalar un pack

Desde la GUI (admin) o vía API/tools del propio Brain:

```
POST /api/v1/packs/install
{ "repo_url": "https://github.com/KH-LLOREDA/brain-packs.git", "ref": "main", "subdir": "infra-proxmox" }
```

o pídeselo a Brain (agente basis, capability core `pack_management`):
`pack_install(subdir="infra-vaultwarden")`  (usa `repo_url`/`ref` por defecto de `packs.*`).

> En entornos aislados (p. ej. kh7) usa un **mirror git interno** como `repo_url`.
> Los packs viven en la **raíz** de este repo, por lo que `subdir` es el id del pack.

## Estructura de un pack

```
<pack>/
  pack.yaml                     # manifest: id, name, version, description, requires
  capabilities/*.yaml           # capabilities (dict o lista de dicts)
  agents/*.yaml                 # PATCH a un agente existente (add_* )
  agents/<id>/agent.yaml        # DEFINE un agente nuevo (source='pack') + prompts/
  skills/<name>/SKILL.md        # skills (frontmatter YAML + cuerpo markdown)
  connections/
    openapi/*.yaml              # plantilla de conexión OpenAPI (SIN secreto)
    mcp/*.yaml                  # plantilla de conexión MCP
  services/*.yaml               # runtime desplegable (proxy / servidor MCP)
  machines/*.yaml               # runtime en VMs
  handlers/*.yaml               # superficies GUI (preview / editor / live)
  templates/*.yaml              # plantillas de «Crear artefacto»
```

### `pack.yaml`

```yaml
id: infra-portainer                # kebab/snake, único
name: "Infra: Portainer"
version: 1.0.0
description: "..."
requires: []                       # ids de otros packs
# visual_apps:                     # tiles de mesa › Aplicaciones (SAP, Blender…)
#   - kind: sap
#     label: SAP GUI
#     icon: table_chart
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
name: "API externa"
slug: mi-api
spec_url: "https://api.example.com/openapi.json"
base_url: "https://api.example.com"
auth_type: bearer                  # none | bearer | api_key | basic
auth_token_env: MI_API_TOKEN
```

Si el pack **despliega** el proxy (`services/*.yaml` con `provides.connection`),
no declares esta plantilla: el instalador escribe `spec_url` y `base_url` con
la URL que devuelve el despliegue. Un hostname fijo se desalinea en cuanto el
contenedor no se llama así.

Tras instalar un pack con conexiones OpenAPI, sus herramientas se regeneran en
caliente (o reinicia el Engine).
