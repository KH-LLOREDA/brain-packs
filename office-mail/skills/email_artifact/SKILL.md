---
name: email_artifact
description: >-
  Cómo materializar correo como artefactos Brain (handlers email y mailbox),
  el circuito borrador → enviar, y qué no hacer (HTML genérico, MTA local).
metadata:
  category: office
  agent: mail_viewer
  display-name: "Artefacto de correo"
---

# Correo — artefactos, buzón y borradores

## Dónde vive el mensaje

El cuerpo vive en el runtime **mail-viewer**. Brain guarda punteros:

- Mensaje suelto: `handler: email`, `mime: application/vnd.brain.email+json`,
  `uri: mail://message/<id>`.
- Buzón: `handler: mailbox`, `mime: application/vnd.brain.mailbox+json`,
  `uri: mail://mailbox/default`. Uno por workspace, agrega las tres carpetas.

El iframe pide JWT (`embed_token`) a Brain. Un enlace desnudo a `/message?id=`
o `/mailbox` sin token no vale.

Cada mensaje tiene `folder` (`inbox`/`drafts`/`sent`) y `status`
(`received`/`draft`/`sent`). Un **borrador** es un mensaje `status=draft`; no es
otro tipo de artefacto.

## Materializar un mensaje

`mail_viewer_ingest_message`:

- `Content-Type: message/rfc822` — snapshot `.eml`, o
- JSON — `from`, `to`, `subject`, `html` / `text`, adjuntos en base64.

La respuesta trae `artifact` (handler, external_id, uri). Ese bloque es el único
origen del tile en Archivos. **Nunca inventes** un `mail://…` ni un id: si no has
llamado a `ingest_message` y no te ha devuelto `artifact.external_id`, el usuario
no ve nada.

## Buzón

- `mail_viewer_open_mailbox` — abre/crea el artefacto buzón (idempotente).
- `mail_viewer_list_messages?folder=inbox|drafts|sent` — lista por carpeta.

Cuando el usuario quiere «ver su correo», abre el buzón, no un mensaje suelto.

## Escribir y enviar (borrador → envío)

- **Escribir un correo** = `mail_viewer_ingest_message` con `status=draft`. Queda
  en Borradores, pendiente de envío. Para una respuesta, enlaza el original con
  `in_reply_to`. **Nunca** llames `m365_mail_send` al redactar.
- **Enviar** = `mail_viewer_send_message` (o el usuario pulsa Enviar en el
  artefacto). El visor NO habla con Graph: devuelve un bloque `send` que el
  executor de Brain enruta a `m365_mail_send` (pack `office-m365`) y, al
  confirmar, marca el borrador como `sent`.

Si `office-m365` no está instalado, el envío no es posible: dilo claramente y
deja el borrador guardado.

## Recibir y automatizar

- `mail_viewer_sync_inbox` trae la bandeja de 365 al buzón (carpeta recibidos),
  deduplicando por el id de Graph. Llena el buzón sin IMAP local.
- BrainFlow `email_inbox` (tools `bf_*`): al llegar correo, el paso de la
  automatización es esa misma ingesta y, **si el usuario lo pidió**, una tarea
  (`user_tasks_*`) o un borrador de respuesta (otro `ingest_message` con
  `status=draft` e `in_reply_to` al original). Nunca inventes un envío: un correo
  nuevo puede crear tarea o borrador, no mandar solo.

## No es

- Un servidor IMAP/SMTP ni un MTA local.
- Un recambio de Graph (`office-m365`): el transporte de envío es suyo.
- El handler `html` de Brain (ese iframe permite scripts).
