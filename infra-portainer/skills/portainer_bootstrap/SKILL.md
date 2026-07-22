---
name: portainer_bootstrap
description: >-
  Dar de alta y operar las conexiones del proxy-portainer a los servidores
  Portainer reales (día-2): crear/probar/eliminar instancias y seleccionar la
  instancia destino en las operaciones. El proxy se despliega vacío; aquí se
  conecta a los Portainer del entorno.
metadata:
  category: infra
  agent: portainer_operator
  display-name: "Portainer — Bootstrap de instancias"
---

# Portainer — Bootstrap de instancias (día-2)

## Trigger conditions
- El pack `infra-portainer` está instalado pero el proxy no tiene ninguna
  instancia Portainer configurada (las tools `portainer_listEnvironments`, etc.
  devuelven vacío o error de "sin instancia activa").
- El usuario quiere conectar Brain a un servidor Portainer nuevo.
- Hay que probar, actualizar o eliminar una conexión a un Portainer.

## Modelo mental

```
Brain (tools portainer_*)  →  proxy-portainer  →  Portainer (uno o varios servidores)
                                │
                                └─ tabla de "instancias" (URL + API key por servidor)
```

- **Brain ↔ proxy**: ya resuelto por el pack. El API token se generó en la
  instalación y vive en la conexión OpenAPI `portainer` (no hay que tocarlo).
- **proxy ↔ Portainer**: es lo que configuramos aquí, con las tools
  `portainer_*Instance`. Cada "instancia" es un servidor Portainer con su URL y
  su API key.

## Prerrequisito humano (secreto real, no se puede inventar)

La `api_key` es una credencial del Portainer del cliente. Hay que obtenerla una
vez (por el admin, o desde el vault si el entorno lo tiene):

1. En Portainer: *(icono de usuario) → My account → Access tokens → Add access
   token*.
2. Anota el token (empieza por `ptr_...`); solo se muestra al crearlo.
3. El usuario del token debe tener permiso sobre los environments a gestionar.

Si el entorno tiene un pack de vault (p. ej. Vaultwarden), obtén de ahí la
`api_key` en lugar de pedírsela al usuario.

## Alta de una instancia

Usa `portainer_createInstance` con:

| Campo | Obligatorio | Ejemplo |
|-------|-------------|---------|
| `slug` | sí (único) | `ptr-prod` |
| `name` | sí | `Portainer Producción` |
| `url` | sí | `https://portainer.acme.com` |
| `api_key` | sí (para operar) | `ptr_xxxxxxxxxxxx` |
| `default_environment_id` | no (default `1`) | `1` |
| `environment_type` | no (`test`/`production`) | `production` |
| `tls_verify` | no (default `false`) | `false` |

Después:
1. `portainer_testInstance(id=<slug>)` para validar la conexión al Portainer.
2. `portainer_listEnvironments` (sin más args si es la única instancia) para
   confirmar y descubrir los `environmentId` disponibles.

## Seleccionar instancia cuando hay varias

Las tools de lectura/admin aceptan seleccionar el Portainer destino:
- Query param `instance=<slug>` (p. ej. `portainer_listStacks` con
  `instance=ptr-prod`).
- Si no se indica, el proxy usa la primera instancia activa.

## Otras operaciones
- `portainer_listInstances` / `portainer_listActiveInstances`: qué servidores hay.
- `portainer_getInstance(id)`: detalle (la `api_key` sale enmascarada `***`).
- `portainer_updateInstance(id, ...)`: cambiar URL, api_key o flags.
- `portainer_deleteInstance(id)`: quitar una conexión.

## Reglas / Pitfalls
- **Nunca** imprimas la `api_key` en claro en el chat. Al leer instancias el
  proxy ya la enmascara; al crearlas, no la repitas en la respuesta.
- `tls_verify: false` es lo normal si el Portainer usa certificado self-signed.
- Un `slug` duplicado falla: reutiliza `portainer_updateInstance` para cambios.
- Muchas tools Docker (containers/stacks) necesitan un `environmentId`: si no lo
  sabes, sácalo de `portainer_listEnvironments`.
- Si `portainer_listEnvironments` da "sin instancia activa", falta el alta:
  sigue esta skill. No es un fallo del token Brain↔proxy.
