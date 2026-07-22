---
name: proxmox_bootstrap
description: >-
  Dar de alta y operar las conexiones del proxy-proxmox a los clusters Proxmox
  reales (día-2): crear/probar/eliminar instancias y seleccionar la instancia
  destino en las operaciones. El proxy se despliega vacío; aquí se conecta a los
  Proxmox del entorno.
metadata:
  category: infra
  agent: proxmox_operator
  display-name: "Proxmox — Bootstrap de instancias"
---

# Proxmox — Bootstrap de instancias (día-2)

## Trigger conditions
- El pack `infra-proxmox` está instalado pero el proxy no tiene ninguna
  instancia Proxmox configurada (las tools `proxmox_listNodes`, etc. devuelven
  vacío o error de "sin instancia activa").
- El usuario quiere conectar Brain a un cluster Proxmox nuevo.
- Hay que probar, actualizar o eliminar una conexión a un Proxmox.

## Modelo mental

```
Brain (tools proxmox_*)  →  proxy-proxmox  →  Proxmox VE (uno o varios clusters)
                              │
                              └─ tabla de "instancias" (URL + token PVE por cluster)
```

- **Brain ↔ proxy**: ya resuelto por el pack. El API token se generó en la
  instalación y vive en la conexión OpenAPI `proxmox` (no hay que tocarlo).
- **proxy ↔ Proxmox**: es lo que configuramos aquí, con las tools
  `proxmox_*Instance`. Cada "instancia" es un cluster Proxmox con su URL y su
  token PVE (`user@realm!nombre=secreto`).

## Prerrequisito humano (secreto real, no se puede inventar)

El `token_secret` es una credencial del Proxmox del cliente. Hay que obtenerla
una vez (por el admin, o desde el vault si el entorno lo tiene):

1. En Proxmox: *Datacenter → Permissions → API Tokens → Add*.
2. Anota `Token ID` con formato `usuario@realm!nombre` (p. ej. `root@pam!brain`)
   y el `Secret` (UUID) que solo se muestra al crearlo.
3. Da al token los permisos necesarios (lectura para `proxmox_read`, más
   permisos para administración).

Si el entorno tiene un pack de vault (p. ej. Vaultwarden), obtén de ahí el
`token_id`/`token_secret` en lugar de pedírselos al usuario.

## Alta de una instancia

Usa `proxmox_createInstance` con:

| Campo | Obligatorio | Ejemplo |
|-------|-------------|---------|
| `slug` | sí (único) | `pve-lab` |
| `name` | sí | `Proxmox Lab` |
| `url` | sí | `https://192.168.7.100:8006` |
| `token_id` | sí (para operar) | `root@pam!brain` |
| `token_secret` | sí (para operar) | `xxxxxxxx-...` |
| `environment_type` | no (`test`/`production`) | `test` |
| `node` | no | `pve` |
| `tls_verify` | no (default `false`) | `false` |

Después:
1. `proxmox_testInstance(id=<slug>)` para validar la conexión al cluster.
2. `proxmox_listNodes` (sin más args si es la única instancia) para confirmar.

## Seleccionar instancia cuando hay varias

Las tools de lectura/admin aceptan seleccionar el cluster destino:
- Query param `instance=<slug>` (p. ej. `proxmox_listVms` con `instance=pve-lab`).
- Si no se indica, el proxy usa la primera instancia activa.

## Otras operaciones
- `proxmox_listInstances` / `proxmox_listActiveInstances`: qué clusters hay.
- `proxmox_getInstance(id)`: detalle (el `token_secret` sale enmascarado `***`).
- `proxmox_updateInstance(id, ...)`: cambiar URL, token o flags.
- `proxmox_deleteInstance(id)`: quitar una conexión.

## Reglas / Pitfalls
- **Nunca** imprimas el `token_secret` en claro en el chat. Al leer instancias
  el proxy ya lo enmascara; al crearlas, no lo repitas en la respuesta.
- `tls_verify: false` es lo normal en Proxmox (certificados self-signed).
- Un `slug` duplicado falla: reutiliza `proxmox_updateInstance` para cambios.
- Si `proxmox_listNodes` da "sin instancia activa", falta el alta: sigue esta
  skill. No es un fallo del token Brain↔proxy.
