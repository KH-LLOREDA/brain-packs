---
name: kh7_proxy_vw_ops
description: >-
  Operar con credenciales en kh7 a través de proxy-vaultwarden: estado/unlock,
  CRUD de ítems y handles opacos (issue/resolve/inject) para no exponer secretos
  en claro.
metadata:
  category: infra
  agent: basis_agent
---

# KH7 Proxy Vaultwarden — Operativa

## Trigger conditions
- Necesitas obtener, crear, editar o eliminar secretos en Vaultwarden
- Necesitas inyectar secretos en configuraciones, scripts o containers
- Necesitas listar qué secretos hay disponibles
- Necesitas entender cómo Brain interactúa con Vaultwarden

## Qué es

Broker de secretos entre LLMs (Brain, Cursor, Claude) y Vaultwarden. El LLM nunca ve secretos en claro: trabaja con **handles opacos** (`vw://h_xxx`) que el proxy resuelve a valores reales en tiempo de ejecución.

```
Brain/LLM  →  handle (vw://h_xxx)  →  Proxy VW  →  Vaultwarden  →  valor real
     ↑                                                        |
     └──────────── valor inyectado (solo en output final) ────┘
```

## Integración Brain

### Conexión OpenAPI
- **Slug**: proxy-vaultwarden
- **specUrl**: `http://brain-proxy-vaultwarden:3001/openapi.json` (nombre DNS del servicio en brain-network; no usar la IP del contenedor, es efímera)
- **base_url**: `https://proxy-vw.kh7.com`
- **11 herramientas** registradas y operativas (capability `kh7_infra`)

### Autenticación
- Bearer token con scope=brain
- Token: vault item **"Proxy Vaultwarden - Brain Bearer Token"** (secureNote)
- El token se configura en la conexión OpenAPI (`PROXY_VW_TOKEN`), no se hardcodea.

### Policies
- Scope **brain**: allow all (create/edit/delete/list/get/inject/resolve)
- Default: deny (otros scopes no tienen acceso)

## Herramientas Brain (`proxy-vaultwarden_vw_*`)

### Lectura (8)
| Tool | Endpoint | Descripción |
|------|----------|-------------|
| proxy-vaultwarden_vw_status | GET /api/vw/status | Estado del proxy + vault (locked/unlocked) |
| proxy-vaultwarden_vw_unlock | POST /api/vw/unlock | Desbloquear vault (si está locked) |
| proxy-vaultwarden_vw_lock | POST /api/vw/lock | Bloquear vault |
| proxy-vaultwarden_vw_list_items | GET /api/vw/items | Listar items del vault (nombre, tipo, id) |
| proxy-vaultwarden_vw_get_item | GET /api/vw/items/{id} | Obtener item completo (campos en claro) |
| proxy-vaultwarden_vw_issue_handle | POST /api/vw/items/{id}/handle | Emitir handle opaco para un campo de un item |
| proxy-vaultwarden_vw_resolve | POST /api/vw/resolve | Resolver handle → valor real |
| proxy-vaultwarden_vw_inject | POST /api/vw/inject | Inyectar valores en template con placeholders |

### Escritura (3)
| Tool | Endpoint | Descripción |
|------|----------|-------------|
| proxy-vaultwarden_vw_create_item | POST /api/vw/items | Crear login/secureNote/card/identity |
| proxy-vaultwarden_vw_edit_item | PUT /api/vw/items/{id} | Editar item (merge parcial) |
| proxy-vaultwarden_vw_delete_item | DELETE /api/vw/items/{id} | Eliminar item |

## Flujos comunes

### Obtener un secreto por nombre de item
```
1. proxy-vaultwarden_vw_list_items → encontrar el item por nombre
2. proxy-vaultwarden_vw_get_item(id) → ver campos (username, password, uris, notes)
```

### Emitir y resolver un handle
```
1. proxy-vaultwarden_vw_issue_handle({ item_id, field: "password" }) → "vw://h_abc123"
2. (usar el handle en configs, scripts, env vars)
3. proxy-vaultwarden_vw_resolve({ handle: "vw://h_abc123" }) → valor real (en output final)
```

### Inyectar secretos en un template
```
proxy-vaultwarden_vw_inject({ template: "PORTAINER_PASSWORD={{vw://h_xxx}}", ... })
→ "PORTAINER_PASSWORD=<valor real>"
```

## Reglas / Pitfalls
- Nunca imprimas contraseñas/tokens en claro en el chat; prefiere handles.
- **Vault locked tras restart**: el entrypoint desbloquea automáticamente; si falla, usa `proxy-vaultwarden_vw_unlock`.
- **Handles expiran** (TTL): si caducan, re-emítelos.
- **Scope brain required**: sin el bearer token correcto, todo devuelve 403 (revisa el token en *Conexiones OpenAPI*, solo admin).
