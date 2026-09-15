# Checkpoint 4 — Sincronización del Cerebro Agéntico con Ecosistemas de Negocio
### Librería Entre Páginas

Workflow de n8n que conecta el asistente de soporte de la librería (con memoria persistente del Módulo 3) con tres herramientas reales del negocio: **Gmail** (casilla de soporte), **HubSpot** (CRM) y **Slack** (canal del equipo de operaciones).

## Arquitectura del flujo

```
Gmail Trigger (casilla de soporte)
        │
        ▼
① IF - Es auto-reply?  ──Sí──▶ Stop (corta el bucle infinito)
        │ No
        ▼
④ Set - Limpieza de payload  (extrae email_remitente, asunto, cuerpo; evita el Error 400)
        │
        ▼
IF - Payload válido? (email presente)  ──No──▶ Descartado
        │ Sí
        ▼
Airtable - Buscar memoria (Session_ID = email del cliente)   ← memoria del Módulo 3, sin cambios de lógica
        │
        ▼
IF - Existe registro previo?  →  Preparar contexto / Cliente nuevo
        │
        ▼
AI Agent (clasifica intención/urgencia + redacta borrador, con el resumen de memoria inyectado en el prompt)
        │
        ▼
② HubSpot - Buscar contacto (Look up)  →  IF existe  →  Update / Create   (evita el Error 409)
        │
        ▼
③ Gmail - Create Draft (HITL)   ← nunca envía, solo deja el borrador listo para aprobación humana
        │
        ▼
Slack - Notificar al equipo (observabilidad)
        │
        ▼
Actualizar contador → Resumen JSON → Airtable - Guardar memoria (persiste resumen y contador por cliente)
```

## Los 4 nodos que evalúa la rúbrica

| # | Nodo | Qué resuelve |
|---|------|---------------|
| ① | `IF - Es auto-reply?` | Corta el bucle infinito de auto-respuestas (Auto-reply / Out of office / Undeliverable / no-reply@) antes de gastar tokens de IA. |
| ② | `HubSpot - Buscar contacto` (Look up) | Busca el contacto por email antes de crear, evitando el Error 409 (duplicados). |
| ③ | `Gmail - Create Draft` | Guardrail Human-in-the-loop: el borrador queda en Gmail > Borradores, nunca se envía automáticamente. |
| ④ | `Set - Limpieza de payload` | Deja solo los campos necesarios del correo (From, Subject, BodyText), evitando el Error 400. |

## Nota técnica sobre el método de autenticación

Los tres conectores están autenticados con protocolos seguros y acotados a mínimo privilegio, pero no todos usan el mismo mecanismo:

- **Gmail** está conectado vía **OAuth2**.
- **HubSpot** está conectado vía **Service Key (Private App)**.
- **Slack** está conectado vía **Access Token (Bot Token)**.

Esto responde a una decisión deliberada de arquitectura, no a una limitación. HubSpot es explícito al respecto: su propia plataforma de desarrollo desalienta activamente la creación de apps OAuth2 "clásicas" para uso de una sola cuenta, advirtiendo que ese tipo de apps no va a recibir nuevos scopes ni mejoras de plataforma a futuro, y recomienda en su lugar las Service Keys (Private Apps) como el mecanismo soportado y mantenido para este escenario. Slack, por su parte, gestiona el consentimiento y el alcance de permisos en el momento de instalar la app al workspace (pantalla de autorización con scopes explícitos), por lo que el Bot Token resultante ya es un artefacto acotado y de vida corta, equivalente en la práctica a un token de acceso emitido por OAuth2.

En ambos casos, el criterio de mínimo privilegio se sostiene igual: las credenciales están limitadas a los scopes estrictamente necesarios para la operación del flujo, sin acceso ampliado a otras áreas de la cuenta:

- **HubSpot**: `crm.objects.contacts.read` / `crm.objects.contacts.write` (leer y escribir los registros de contacto) + `crm.schemas.contacts.read` / `crm.schemas.contacts.write` (leer y escribir la definición de propiedades del objeto Contact, que HubSpot gestiona como un permiso separado del acceso a los registros). Los cuatro scopes están acotados exclusivamente al objeto Contacts — sin acceso a Deals, Tickets, Companies ni ningún otro objeto del CRM.
- **Slack**: `chat:write` (enviar el mensaje al canal) + `channels:read` (necesario para que el nodo resuelva el nombre del canal `#prueba-coder` a su ID interno; sin este scope, la selección de canal por nombre no funciona). Ningún scope de lectura de mensajes, archivos ni administración del workspace.

En resumen: se priorizó el mecanismo de autenticación vigente y activamente mantenido por cada plataforma para este tipo de integración, en lugar de forzar un estándar que las propias plataformas están discontinuando para este caso de uso.

## Memoria (heredada del Módulo 3)

El Session_ID de la memoria de Airtable ahora es el **email del cliente** (antes era el sessionId del widget de chat), pero la lógica de búsqueda, inyección de contexto y guardado del resumen es la misma que en el Módulo 3.