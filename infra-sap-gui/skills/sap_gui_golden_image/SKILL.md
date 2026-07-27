---
name: sap_gui_golden_image
description: >-
  Construir y refrescar la imagen dorada (plantilla Proxmox) de las VMs Windows
  con SAP GUI que sirve el pool: preparación, sysprep, plantilla y snapshot.
metadata:
  category: infra
  agent: basis_agent
  display-name: "Imagen dorada SAP GUI"
---

# Imagen dorada de SAP GUI (Windows sobre Proxmox)

El pool no crea máquinas de la nada: clona una **plantilla** de Proxmox que ya
lleva Windows, SAP GUI y el servidor MCP instalados. Este documento es el
procedimiento para construirla la primera vez y para refrescarla cuando cambie
SAP GUI, el MCP o el sistema destino.

Es trabajo de infraestructura con pasos interactivos (instalar Windows, instalar
SAP GUI): no se automatiza desde Brain. Lo que sí es automático, una vez existe
la plantilla, es todo el ciclo de vida del pool.

Los artefactos que se citan (`install.ps1`, `start-pool-vm.ps1`, `unattend.xml`,
el servidor MCP) viven en el repositorio del workspace-manager, en
`sandbox-sap-gui-win/`.

## Cuándo aplica cada modo

- **Pool multiusuario** (lo que instala este pack): la imagen NO lleva
  credenciales SAP. Cada usuario recibe una VM y Brain hace `sap_login` con las
  credenciales SAP de esa persona. Licencia y auditoría quedan por usuario real,
  aunque Windows corra siempre como el usuario genérico `sapbot`.
- **VM estática** (solo desarrollo): credenciales horneadas en la imagen y
  `SAP_WIN_MCP_URL` apuntando a ella. No usa pool ni plantilla.

## 1. VM base en Proxmox

Windows 11 exige una configuración concreta; si falta algo, el instalador ni
arranca:

- BIOS **OVMF (UEFI)** con disco EFI, máquina **q35** y dispositivo **TPM 2.0**
- CPU `host` 4 núcleos, 8 GB de RAM, disco de 80 GB en `virtio-scsi single`
- Red `virtio` en un bridge desde el que Brain alcance la VM
- ISO de Windows 11 y, en un segundo CD, la ISO `virtio-win`
- Agente QEMU habilitado en las opciones de la VM

Durante la instalación hay que cargar el driver VirtIO SCSI (`vioscsi\w11\amd64`)
o Windows no verá el disco.

## 2. Software dentro de Windows

1. Usuario local `sapbot` con permisos de administrador.
2. **SAP GUI for Windows** (NWSAPSetup) y una conexión al sistema destino
   verificada entrando a mano una vez.
3. En SAP Logon → Opciones → Accesibilidad y scripting: activar **scripting**.
4. En el servidor SAP, `sapgui/user_scripting = TRUE` (RZ11). Sin esto el
   scripting falla desde el cliente aunque esté activado en local.
5. `qemu-guest-agent` desde la ISO virtio. **El pool depende de él**: sin agente
   no hay forma de averiguar la IP del clon y la VM se descarta por timeout.

## 3. Bootstrap

Copiar `sandbox-sap-gui-win/` a la VM (por ejemplo `C:\brain\src`) y, en una
PowerShell **como administrador y con la sesión del usuario del pool**:

```powershell
cd C:\brain\src\bootstrap
.\install.ps1 -SapHost <host-sap> -SapSystem <nombre-paisaje> `
              -WinUser sapbot -WinPassword "<clave-windows>" -McpPort 3001
```

Sin `-SapUser`/`-SapPassword`: en el pool las credenciales son de cada persona y
se inyectan en caliente. Si aparecen en `C:\brain\sap-mcp\.env`, la imagen está
mal construida y todos compartirían usuario SAP.

El script instala Python y dependencias, activa el auto-logon de Windows,
desactiva suspensión y bloqueo, abre el firewall (3001 y RDP), monta el stack
VNC de vista previa y registra la tarea de inicio de sesión `BrainSapMcp`.

Tras reiniciar, comprobar desde otra máquina: `curl http://<ip-vm>:3001/mcp`.
Los registros están en `C:\brain\sap-mcp\logs\`.

El MCP corre como tarea de sesión, no como servicio, porque el scripting de SAP
GUI solo existe dentro de la sesión interactiva de escritorio. Por eso el
auto-logon no es una comodidad: es un requisito.

## 4. Generalizar (sysprep)

Cada clon necesita SID y nombre de host propios; clonar sin generalizar provoca
choques de red y de dominio difíciles de diagnosticar.

```powershell
C:\Windows\System32\Sysprep\sysprep.exe /generalize /oobe /shutdown `
  /unattend:C:\brain\src\bootstrap\unattend.xml
```

El `unattend.xml` recrea el usuario `sapbot` con auto-logon para que la tarea del
MCP siga funcionando después de generalizar. Hay que rellenar sus marcadores de
contraseña antes de usarlo.

## 5. Plantilla y snapshot

1. Con la VM parada, convertirla en **plantilla** en Proxmox y anotar su VMID:
   es el valor de `PROXMOX_WIN_TEMPLATE_ID`.
2. Clonar una primera VM, dejarla arrancar hasta que el MCP responda y, **antes
   de que nadie inicie sesión en SAP**, tomar un snapshot llamado `clean`.

El snapshot es el mecanismo de reciclado: al soltar una VM se revierte a él, y
eso es lo que garantiza que la siguiente persona no herede la sesión SAP ni los
ficheros de la anterior. Si el snapshot no existe con ese nombre exacto, el pool
recicla reiniciando la VM, que es más lento y no borra el estado.

## 6. Comprobación del pool

Con la plantilla lista, completar en Brain (Configuración › Sandbox) la URL del
hipervisor, el token, el nodo y el id de plantilla. El resto de la política
(prefijo, VMs calientes, reciclado, puertos) la trae ya el pack `infra-sap-gui`.

La primera petición a SAP debería: adquirir una VM caliente, conectar su MCP por
IP de LAN, hacer `sap_login` con las credenciales del usuario y devolver una URL
de vista previa VNC. Si falla, mirar en este orden:

- ¿responde el agente QEMU y devuelve IP? Sin IP no hay MCP.
- ¿resuelve `curl http://<ip>:3001/mcp` desde el host del workspace-manager?
- ¿tiene la persona su conexión SAP configurada en su perfil? El pool no lleva
  credenciales.
- ¿existe el snapshot `clean` en el clon, no solo en la plantilla?

## Refrescar la imagen

Al actualizar SAP GUI o el MCP: clonar la plantilla, aplicar los cambios en el
clon, repetir sysprep, convertirlo en plantilla nueva y apuntar
`PROXMOX_WIN_TEMPLATE_ID` al VMID nuevo. Las VMs vivas del pool siguen con la
plantilla anterior hasta que se reciclan, así que conviene bajar `max` y dejar
que se vacíen, o retirarlas a mano, antes de dar el cambio por hecho.
