---
name: calendar_teams
description: >-
  Gestión de agenda, creación de eventos, envío de mensajes Teams
metadata:
  category: office
  agent: m365_assistant
  display-name: "Calendario y Teams"
---

# Calendario y Teams - Skill Específico

## Calendario

### Consultar Agenda
```
Hoy: m365_calendar_events(start_date_time="2026-02-17T00:00:00Z", end_date_time="2026-02-17T23:59:59Z")
Esta semana: calcular lunes a viernes del rango actual
Este mes: primer y último día del mes
```

### Crear Eventos
Formato requerido para start/end:
```json
{
  "dateTime": "2026-02-20T10:00:00",
  "timeZone": "Europe/Madrid"
}
```

Ejemplo completo:
```
m365_calendar_create_event(
  subject="Reunión de equipo",
  start={"dateTime": "2026-02-20T10:00:00", "timeZone": "Europe/Madrid"},
  end={"dateTime": "2026-02-20T11:00:00", "timeZone": "Europe/Madrid"},
  location="Sala Barcelona",
  attendees=["juan@khlloreda.com", "maria@khlloreda.com"]
)
```

### Presentación de Agenda
Organizar por día, mostrando:
- Hora inicio → Hora fin
- Título del evento
- Ubicación (si tiene)
- Asistentes (si tiene)

Ejemplo de presentación:
```
📅 Lunes 17/02/2026
  09:00 - 10:00  Standup diario (Sala Virtual)
  11:00 - 12:30  Revisión de proyecto (Sala Madrid) - 5 asistentes
  
📅 Martes 18/02/2026
  10:00 - 11:00  1:1 con Pedro (Teams)
  14:00 - 15:00  Comité de dirección (Sala Barcelona) - 8 asistentes
```

## Microsoft Teams

### Flujo para Enviar Mensaje a un Canal
1. Listar equipos: `m365_teams_list()`
2. Identificar el equipo correcto por nombre
3. Listar canales: `m365_teams_channels(team_id="...")`
4. Identificar el canal correcto
5. Confirmar con el usuario
6. Enviar: `m365_teams_send_message(team_id="...", channel_id="...", content="...")`

### Formato de Mensajes Teams
- `content_type="html"` para mensajes formateados
- `content_type="text"` para texto plano
- Se puede incluir `subject` como título del hilo

### Ejemplo de Mensaje HTML para Teams
```html
<h3>Actualización semanal</h3>
<ul>
  <li><strong>Completado:</strong> Migración de datos</li>
  <li><strong>En progreso:</strong> Testing de integración</li>
  <li><strong>Pendiente:</strong> Documentación</li>
</ul>
```

## Notas
- Los team_id y channel_id son GUIDs largos, no inventarlos
- Siempre listar primero para obtener los IDs reales
- Para canales tipo "General", el channel_id suele empezar por "19:"
