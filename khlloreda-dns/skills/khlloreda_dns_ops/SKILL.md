---
name: khlloreda_dns_ops
description: >-
  Gestionar registros DNS de khlloreda.com con split-horizon vía proxy-dns:
  externos en Arsys e internos en el Windows Domain Controller.
metadata:
  category: infra
  agent: basis_agent
---

# Gestión DNS de khlloreda.com (proxy-dns)

`proxy-dns` orquesta dos backends para khlloreda.com:

- **Arsys (externo)**: resolución pública. La API de Arsys **no** soporta
  registros wildcard A/CNAME (sí TXT para retos DNS-01 de ACME/lego).
- **Windows Domain Controller (interno)**: resolución en la red interna
  (split-horizon), gestionado por WinRM.

Las herramientas generadas se llaman `proxy-dns_*`.

## Operaciones típicas

- `proxy-dns_list_records` / `proxy-dns_get_record` — inspecciona la zona antes
  de cambiar nada.
- `proxy-dns_create_record` — crea A/CNAME/TXT. Indica la vista (externa/interna)
  según el orquestador lo exponga.
- `proxy-dns_update_record` / `proxy-dns_delete_record` — modifica/borra. Recuerda
  que en zonas grandes hay que paginar/buscar el registro exacto por nombre+tipo.

## Reglas

- No crees wildcard A/CNAME en Arsys (no soportado); para `*.ws.khlloreda.com`
  usa delegación de sub-zona + certificado lego con reto TXT DNS-01.
- Mantén coherencia entre la vista externa (Arsys) y la interna (DC) cuando el
  servicio deba resolver en ambas.
- El token de proxy-dns se configura en la conexión OpenAPI (`PROXY_DNS_TOKEN`).
