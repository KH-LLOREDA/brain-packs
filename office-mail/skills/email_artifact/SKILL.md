---
name: email_artifact
description: >-
  Cómo materializar un correo como artefacto Brain (handler email) y qué
  no hacer (HTML genérico, MTA local, inbox).
metadata:
  category: office
  agent: mail_viewer
  display-name: "Artefacto de correo"
---

# Correo — artefacto y visor

## Dónde vive el mensaje

El cuerpo vive en el runtime **mail-viewer**. Brain guarda un puntero:

- `handler: email`
- `existing_uri: mail://message/<id>`
- `mime: application/vnd.brain.email+json`

El iframe pide JWT (`embed_token`) a Brain. Un enlace desnudo a `/message?id=` sin token no vale.

## Crear

`mail_viewer_ingest_message`:

- `Content-Type: message/rfc822` — snapshot `.eml`
- JSON — `from`, `to`, `subject`, `html` / `text`, adjuntos en base64

La respuesta trae `artifact` (handler, external_id, uri). No crees otro artefacto a mano.

## Ver

El usuario abre el artefacto. Imágenes remotas van bloqueadas hasta que las active. `cid:` se reescribe a adjuntos locales.

## No es

- Un servidor IMAP/SMTP.
- Un recambio de Graph (`office-m365`).
- El handler `html` de Brain (ese iframe permite scripts).
