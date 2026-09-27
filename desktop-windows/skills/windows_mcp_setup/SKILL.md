---
name: windows_mcp_setup
description: >-
  Construir la plantilla Proxmox del escritorio Windows (Windows-MCP en el
  puerto 8000) y dejar el pool del pack desktop-windows sirviendo VMs.
metadata:
  category: infra
  agent: basis_agent
  display-name: "Imagen y pool Windows"
---

# Escritorio Windows en el pool

Este pack es el dueño del hipervisor. La plantilla de aquí es un Windows con
sesión de escritorio y Windows-MCP. No lleva SAP GUI: esa imagen la construye
el pack `infra-sap-gui` como otro perfil, sobre el mismo Proxmox.

## 1. Plantilla

En una VM de Proxmox, con el usuario que va a usar el escritorio ya creado y
con auto-logon (un servicio sin sesión no ve el escritorio):

```powershell
uvx windows-mcp serve --transport streamable-http --host 0.0.0.0 --port 8000
```

Sin `--auth-key`. El cliente MCP de Brain no envía esa cabecera. Deja el
proceso al arranque de la sesión (tarea programada al iniciar sesión, no un
servicio de Session 0). Comprueba desde otra máquina:
`curl http://<ip>:8000/mcp`.

Sysprep y convierte la VM en plantilla. Anota el VMID. El snapshot de reciclado
se llama `clean` y tiene que existir en la plantilla; los clones lo heredan.

El puerto del MCP es 8000 y el prefijo de las VMs es `win-`. No reutilices la
plantilla de SAP (`sap-win-`, MCP en 3001): son imágenes distintas.

## 2. Variables del entorno

En el proceso que instala el pack, o después en Configuración › Sandbox:

- `PROXMOX_URL`, `PROXMOX_TOKEN_ID`, `PROXMOX_TOKEN_SECRET`, `PROXMOX_NODE`
- `PROXMOX_WIN_DESKTOP_TEMPLATE_ID` — VMID de esta plantilla
- `PROXMOX_VM_STORAGE`, `PROXMOX_VM_TARGET_NODE` si no valen los del clúster

Si faltan al instalar, el pool queda inactivo. Al completarlas, el Engine
empuja la configuración al workspace-manager en el primer uso.

## 3. PC fijo, sin pool

`WINDOWS_MCP_URL` en el **Engine** (origen, sin `/mcp`, por ejemplo
`http://192.168.1.50:8000`) conecta ese PC y no pide VM. `WINDOWS_VNC_URL` es
opcional. No hace falta plantilla. No publiques el puerto: es control total
del escritorio.
