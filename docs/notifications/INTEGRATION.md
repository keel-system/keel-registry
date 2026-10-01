---
service: notifications
version: 0.1.1
domain: communications
basePath: /api/v1
m2mAuth:
  protocol: client-credentials
  audience: notifications
  validateAudience: true
endpoints:
  - name: requestNotification
    method: POST
    path: /notifications
    access: service notification:send
  - name: findMessageByIdempotencyKey
    method: GET
    path: /notifications/{applicationCode}/{idempotencyKey}
    access: service notification:read
  - name: reportEmailBounce
    method: POST
    path: /messages/{messageId}/bounces
    access: service bounce:report
events:
  envelope: keel
  published:
    - name: EmailSent
      channel: notificationEvents
    - name: EmailDeliveryFailed
      channel: notificationEvents
  consumed:
    - name: NotificationRequested
      channel: notificationRequests
      source: any-registered-system
      envelope: keel
      nature: request
errors:
  - code: VALIDATION_ERROR
    http: 400
  - code: UNAUTHENTICATED
    http: 401
  - code: ACCESS_DENIED
    http: 403
  - code: APPLICATION_FORBIDDEN
    http: 403
  - code: MESSAGE_NOT_FOUND
    http: 404
  - code: IDEMPOTENCY_KEY_REUSED
    http: 409
  - code: IDEMPOTENCY_KEY_IN_PROGRESS
    http: 409
  - code: APPLICATION_SUSPENDED
    http: 409
  - code: MESSAGE_NOT_SENT
    http: 409
  - code: APPLICATION_ADDRESS_ALREADY_EXISTS
    http: 409
  - code: APPLICATION_NOT_FOUND
    http: 422
  - code: TEMPLATE_NOT_FOUND
    http: 422
  - code: TEMPLATE_NOT_PUBLISHED
    http: 422
  - code: DUPLICATE_VARIABLE_VALUE
    http: 422
  - code: MISSING_TEMPLATE_VARIABLES
    http: 422
  - code: RECIPIENT_SUPPRESSED
    http: 422
  - code: RENDERED_SUBJECT_TOO_LONG
    http: 422
  - code: BOUNCE_RECIPIENT_UNKNOWN
    http: 422
---

# Integración con notifications

## Resumen

`notifications` envía correo transaccional en nombre de cada aplicación consumidora (dominio
`communications`). Una aplicación pide un correo con el código de una de sus plantillas, los
destinatarios y los valores de las variables. Lo puede hacer por HTTP (`POST /notifications`) o
publicando `NotificationRequested`: **el contrato es el mismo por las dos puertas**. El servicio
acepta la petición, la encola y la envía en el siguiente ciclo de despacho (cada minuto). La aceptación
(`202`) **no promete la entrega**. El desenlace (`sent` o `failed`, siempre terminal porque el servicio
no reintenta) se consulta por la clave de la petición o se escucha en `notificationEvents`. **La misma
`idempotencyKey` nunca produce dos correos**, por cualquiera de las dos puertas y para siempre. El
proveedor de correo, por su parte, reporta los rebotes, que suprimen la dirección en la aplicación.

Contratos formales: [`openapi.yaml`](openapi.yaml) (HTTP) y [`asyncapi.yaml`](asyncapi.yaml) (eventos).

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con un token de usuario. Para obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno. **Para una
   aplicación consumidora, el nombre del cliente es el `code` de tu aplicación en `notifications`**
   (correspondencia 1:1). Además, tu aplicación tiene que estar dada de alta en el servicio con ese mismo
   code: si no lo está, `POST /notifications` responde `422 APPLICATION_NOT_FOUND`.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los scopes concedidos a tu
   cliente (ver tabla). Fija la audiencia `notifications`: se valida, y un token emitido para otro
   servicio responde `403 ACCESS_DENIED`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=notification:send notification:read&audience=notifications
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| billing | notification:send, notification:read | Aplicación de facturación: pide envíos y consulta los suyos. |
| shipping | notification:send, notification:read | Aplicación de envíos logísticos: pide envíos y consulta los suyos. |
| dispatch-only | notification:send | Aplicación que solo pide envíos; no consulta su estado (mínimo privilegio). |
| mail-provider | bounce:report | Proveedor de correo: reporta rebotes. No representa a ninguna aplicación. |

Todo error responde con la misma forma: `{ timestamp, status, error, code, message, details,
correlationId }`. El contrato es el `code`; `message` es texto libre. Los campos sin valor viajan como
`null` y las colecciones vacías como `[]`. Los instantes son ISO-8601 en UTC.

### requestNotification

Pide un correo a partir de una plantilla de **tu** aplicación. La aplicación no viaja en el cuerpo: la
fija el servidor a partir de tu credencial.

| | |
|---|---|
| Endpoint | `POST /api/v1/notifications` |
| Acceso | `service` — scope `notification:send` |
| Respuesta | `202` — aceptado y en cola; no promete la entrega |
| Idempotencia | sí — campo `idempotencyKey` del cuerpo, **permanente** (no caduca nunca, tampoco tras la purga de datos personales), por aplicación. Es la misma clave que por el canal de eventos. |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| idempotencyKey | string | requerido; `^[A-Za-z0-9._:-]{8,128}$`. Identifica el envío en tu aplicación. |
| templateCode | string | requerido; `^[a-z][a-z0-9-]{2,47}$`. Plantilla de tu aplicación. |
| recipients | string[] | requerido; 1–20 direcciones. Se normalizan a minúsculas y se deduplican; todas van en `To` de un único correo. |
| variables | `{ name, value }[]` | opcional; ≤ 100. `name` `^[A-Za-z][A-Za-z0-9_]{0,63}$`, `value` texto ≤ 2000. Las que la plantilla no declara se descartan en silencio; las declaradas como requeridas tienen que venir; un `name` repetido es un error. |

```json
{
  "idempotencyKey": "inv-2026-0001",
  "templateCode": "invoice-issued",
  "recipients": ["Ana@Example.com", "luis@example.com"],
  "variables": [
    { "name": "invoiceNumber", "value": "F-0001" },
    { "name": "customerName",  "value": "Ana" },
    { "name": "amount",        "value": "10,00 €" }
  ]
}
```

**Repetición.** Repetir la misma `idempotencyKey` con el **mismo contenido** responde `202` con el
mensaje original en su estado actual y **no envía nada**, aunque desde entonces la aplicación se haya
suspendido o la plantilla se haya retirado. «Mismo contenido» significa: el mismo `templateCode`, el
mismo conjunto de destinatarios normalizados y el mismo conjunto de pares de variables declaradas, sin
importar el orden. Con contenido distinto responde `409 IDEMPOTENCY_KEY_REUSED`. Ante un timeout,
**reintenta siempre con la misma clave**: es seguro.

**Response** — `EmailMessage`

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | id del mensaje; es el que viaja en la cabecera `X-Notification-Id` del correo |
| idempotencyKey | string | la tuya |
| recipients | string[] | normalizados; `[]` tras la purga de datos personales |
| renderedSubject | string \| null | asunto ya renderizado al aceptar; `null` tras la purga |
| sender | `{ address, name, replyToAddress }` | From y Reply-To de tu aplicación, congelados al aceptar |
| templateCode | string | |
| templateVersionNumber | int | versión de la plantilla congelada al aceptar |
| templateVersionId | uuid | |
| status | `queued` \| `sending` \| `sent` \| `failed` | `sent` y `failed` son terminales |
| failureReason | `delivery-rejected` \| `delivery-error` \| `render-error` \| `application-suspended` \| `recipient-suppressed` \| `sending-timeout` \| null | con `sending-timeout` se desconoce si el correo llegó a salir |
| failureDetail | string \| null | diagnóstico legible; no es contrato |
| requestedAt | timestamp | |
| sendingSince | timestamp \| null | |
| sentAt | timestamp \| null | cuándo el relay aceptó el correo |
| failedAt | timestamp \| null | |
| personalDataPurgedAt | timestamp \| null | |
| applicationId | uuid | |

Nunca viajan los valores de las variables (`variableValues`) ni la huella del contenido.

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "idempotencyKey": "inv-2026-0001",
  "recipients": ["ana@example.com", "luis@example.com"],
  "renderedSubject": "Factura F-0001",
  "sender": { "address": "facturas@notify.example.com", "name": "Facturación Acme", "replyToAddress": "soporte@example.com" },
  "templateCode": "invoice-issued",
  "templateVersionNumber": 1,
  "templateVersionId": "2f1b8c3e-0a4d-4e6f-9b7a-1c2d3e4f5a6b",
  "status": "queued",
  "failureReason": null,
  "failureDetail": null,
  "requestedAt": "2026-09-30T10:15:02.120Z",
  "sendingSince": null,
  "sentAt": null,
  "failedAt": null,
  "personalDataPurgedAt": null,
  "applicationId": "5b3e9d10-6c7a-4f21-8e44-0d9a2b7c1e35"
}
```

**Orden de evaluación** (con dos fallos a la vez, ves el primero): aplicación de la credencial →
variables repetidas → repetición de la clave → aplicación suspendida → plantilla → versión activa →
variables requeridas → destinatarios suprimidos → longitud del asunto renderizado.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| VALIDATION_ERROR | 400 | El cuerpo no cumple las cotas (clave, destinatarios 1–20, patrones). | Corregir input. |
| UNAUTHENTICATED | 401 | Sin token o token inválido. | Renovar el token y reintentar. |
| ACCESS_DENIED | 403 | El token no tiene `notification:send` o no es de la audiencia `notifications`. | No reintentar; revisar la credencial. |
| IDEMPOTENCY_KEY_REUSED | 409 | La aplicación ya usó esa `idempotencyKey` con un contenido distinto. | No reintentar; usa una clave nueva si es otro envío. |
| IDEMPOTENCY_KEY_IN_PROGRESS | 409 | Otra petición con la misma clave se está aceptando en este momento. | Reintentable con la **misma** clave tras una espera breve: obtendrás el mensaje original. |
| APPLICATION_SUSPENDED | 409 | Tu aplicación está suspendida y la clave no corresponde a un mensaje ya aceptado. | No reintentar hasta que la reactiven. |
| APPLICATION_NOT_FOUND | 422 | Tu credencial no corresponde a ninguna aplicación registrada. | No reintentar; pide el alta de la aplicación. |
| TEMPLATE_NOT_FOUND | 422 | Tu aplicación no tiene una plantilla con `templateCode`. | Corregir input. |
| TEMPLATE_NOT_PUBLISHED | 422 | La plantilla existe pero no tiene versión activa (está retirada). | No reintentar hasta que la publiquen. |
| DUPLICATE_VARIABLE_VALUE | 422 | `variables` trae dos valores con el mismo `name`. | Corregir input. |
| MISSING_TEMPLATE_VARIABLES | 422 | Falta alguna variable que la versión activa declara como requerida. | Corregir input. |
| RECIPIENT_SUPPRESSED | 422 | Algún destinatario tiene una supresión activa en tu aplicación. Se rechaza la petición entera. | Corregir input (quitar la dirección suprimida). |
| RENDERED_SUBJECT_TOO_LONG | 422 | El asunto renderizado supera los 998 caracteres. | Corregir input. |

### findMessageByIdempotencyKey

Consulta el estado de un envío **tuyo** por la clave con la que lo pediste. Es la vía para conocer el
desenlace y para saber si una petición enviada por evento llegó a aceptarse (`404` = no se aceptó).

| | |
|---|---|
| Endpoint | `GET /api/v1/notifications/{applicationCode}/{idempotencyKey}` |
| Acceso | `service` — scope `notification:read` |
| Idempotencia | no aplica (query) |

**Request** — path `applicationCode: string` (requerido; tiene que ser **tu** aplicación) e
`idempotencyKey: string` (requerido).

**Response** — `EmailMessage`, con la misma forma que en [requestNotification](#requestnotification).
Un mensaje ya enviado:

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "idempotencyKey": "inv-2026-0001",
  "recipients": ["ana@example.com", "luis@example.com"],
  "renderedSubject": "Factura F-0001",
  "sender": { "address": "facturas@notify.example.com", "name": "Facturación Acme", "replyToAddress": "soporte@example.com" },
  "templateCode": "invoice-issued",
  "templateVersionNumber": 1,
  "templateVersionId": "2f1b8c3e-0a4d-4e6f-9b7a-1c2d3e4f5a6b",
  "status": "sent",
  "failureReason": null,
  "failureDetail": null,
  "requestedAt": "2026-09-30T10:15:02.120Z",
  "sendingSince": "2026-09-30T10:16:00.031Z",
  "sentAt": "2026-09-30T10:16:00.842Z",
  "failedAt": null,
  "personalDataPurgedAt": null,
  "applicationId": "5b3e9d10-6c7a-4f21-8e44-0d9a2b7c1e35"
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| VALIDATION_ERROR | 400 | `applicationCode` o `idempotencyKey` no cumplen su patrón. | Corregir input. |
| UNAUTHENTICATED | 401 | Sin token o token inválido. | Renovar el token y reintentar. |
| ACCESS_DENIED | 403 | El token no tiene `notification:read` (p. ej. el cliente `dispatch-only`) o es de otra audiencia. | No reintentar. |
| APPLICATION_FORBIDDEN | 403 | `applicationCode` no es la aplicación de tu credencial. | No reintentar; usa tu propio code. |
| MESSAGE_NOT_FOUND | 404 | Tu aplicación no tiene ningún mensaje con esa clave: la petición no se aceptó. | No reintentar la consulta; vuelve a pedir el envío si procede. |

### reportEmailBounce

Lo usa el **proveedor de correo** para reportar el rebote de un destinatario. El `messageId` sale de la
cabecera `X-Notification-Id` que lleva todo correo enviado por el servicio. Traducir el webhook propio
del proveedor a esta llamada es un adaptador de despliegue.

| | |
|---|---|
| Endpoint | `POST /api/v1/messages/{messageId}/bounces` |
| Acceso | `service` — scope `bounce:report` |
| Respuesta | `204` sin cuerpo |
| Idempotencia | idempotente por naturaleza: repetir un rebote no tiene más efecto que reportarlo una vez |

**Request** — path `messageId: uuid` (requerido) y cuerpo:

| Campo | Tipo | Notas |
|---|---|---|
| recipient | string | requerido; dirección que rebotó (se normaliza a minúsculas). Tiene que ser destinataria del mensaje. |
| bounceType | `permanent` \| `transient` | requerido. `permanent` suprime la dirección en la aplicación del mensaje (con `hard-bounce`, aunque un humano la hubiera liberado); `transient` solo queda en el log. |
| diagnostic | string | opcional; ≤ 1000. Se guarda como nota de la supresión. |

```json
{ "recipient": "ana@example.com", "bounceType": "permanent", "diagnostic": "550 mailbox unavailable" }
```

**Response** — `204` sin cuerpo.

Se acepta el rebote de un mensaje en `sending`, `sent` o `failed`: un mensaje rescatado por
`sending-timeout` puede haber salido de verdad.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| VALIDATION_ERROR | 400 | Cuerpo inválido (`bounceType` desconocido, dirección mal formada). | Corregir input. |
| UNAUTHENTICATED | 401 | Sin token o token inválido. | Renovar el token y reintentar. |
| ACCESS_DENIED | 403 | El token no tiene `bounce:report` o es de otra audiencia. | No reintentar. |
| MESSAGE_NOT_FOUND | 404 | No existe un mensaje con ese id. | No reintentar. |
| MESSAGE_NOT_SENT | 409 | El mensaje sigue en `queued`: todavía no ha salido. | No reintentar; un rebote de algo que no salió no es de este mensaje. |
| APPLICATION_ADDRESS_ALREADY_EXISTS | 409 | Otro reporte de la misma dirección creó la supresión en paralelo. | Reintentable: el reintento la encuentra activa y responde `204`. |
| BOUNCE_RECIPIENT_UNKNOWN | 422 | `recipient` no es destinataria del mensaje, o el mensaje ya se purgó (18 meses) y no hay con qué comprobarlo. | Corregir input, o descartar el rebote si el mensaje es antiguo. |

## Eventos

Detalle formal: [`asyncapi.yaml`](asyncapi.yaml). El broker concreto y los nombres físicos de
topic/cola se deciden al desplegar.

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "EmailSent",
    "eventVersion": 1,
    "occurredAt": "2026-09-30T10:16:00.842Z",
    "source": "notifications",
    "correlationId": null,
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": {
    "messageId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "applicationCode": "billing",
    "idempotencyKey": "inv-2026-0001",
    "templateCode": "invoice-issued",
    "templateVersionNumber": 1,
    "sentAt": "2026-09-30T10:16:00.842Z"
  }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`EmailSent`, `EmailDeliveryFailed`). Discriminador: el canal transporta los dos tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| metadata.source | string | Servicio emisor: `notifications`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición (los dos eventos nacen en el despacho programado, así que normalmente es `null`). |
| metadata.traceparent | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Garantía de entrega: best-effort.** Si el broker no está disponible en el instante en que el
servicio confirma el desenlace, el evento se pierde sin reintento. La fuente de verdad del desenlace es
el estado del mensaje, consultable con [findMessageByIdempotencyKey](#findmessagebyidempotencykey).
Ningún evento lleva destinatarios ni datos personales.

### EmailSent

| | |
|---|---|
| Canal | `notificationEvents` — desenlace de cada correo |
| Garantía | best-effort |
| Emitido por | `sendQueuedMessage` (despacho programado) |

El relay aceptó el correo. **No** garantiza que haya llegado al buzón.

| Campo | Tipo | Notas |
|---|---|---|
| messageId | uuid | requerido |
| applicationCode | string | requerido |
| idempotencyKey | string | requerido; la clave con que la aplicación pidió el envío |
| templateCode | string | requerido |
| templateVersionNumber | int | requerido |
| sentAt | timestamp | requerido |

```json
{
  "messageId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "applicationCode": "billing",
  "idempotencyKey": "inv-2026-0001",
  "templateCode": "invoice-issued",
  "templateVersionNumber": 1,
  "sentAt": "2026-09-30T10:16:00.842Z"
}
```

### EmailDeliveryFailed

| | |
|---|---|
| Canal | `notificationEvents` — desenlace de cada correo |
| Garantía | best-effort |
| Emitido por | `sendQueuedMessage`, `dispatchQueuedMessages` (rescate de envíos atascados) |

El correo terminó en `failed`. Es **terminal**: el servicio no lo reintenta. Si hace falta, la aplicación
lo vuelve a pedir con una clave nueva.

| Campo | Tipo | Notas |
|---|---|---|
| messageId | uuid | requerido |
| applicationCode | string | requerido |
| idempotencyKey | string | requerido |
| templateCode | string | requerido |
| templateVersionNumber | int | requerido |
| failureReason | `delivery-rejected` \| `delivery-error` \| `render-error` \| `application-suspended` \| `recipient-suppressed` \| `sending-timeout` | requerido. Con `sending-timeout` se desconoce si el correo llegó a salir. |
| failedAt | timestamp | requerido |

```json
{
  "messageId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "applicationCode": "billing",
  "idempotencyKey": "inv-2026-0001",
  "templateCode": "invoice-issued",
  "templateVersionNumber": 1,
  "failureReason": "recipient-suppressed",
  "failedAt": "2026-09-30T10:16:00.511Z"
}
```

### Suscripciones

**Puertas de entrada (`nature: request`).** Lo que sigue es contrato público: lo publica quien quiera
encargarle un envío a este servicio, y su firma no cambia sin una versión nueva.

### NotificationRequested

| | |
|---|---|
| Naturaleza | `request` — segunda puerta de [requestNotification](#requestnotification), con el mismo contrato que `POST /notifications` |
| Origen | cualquier aplicación registrada (`any-registered-system`) |
| Canal | `notificationRequests` |
| Envoltura | `keel` — la misma *Forma del mensaje* de §Publicados, ahora del lado de quien publica |
| Discriminador | `metadata.eventType = NotificationRequested` |
| Deduplicación | `metadata.eventId` (reentregas del broker) y, por encima, la `idempotencyKey` del payload, permanente |
| Formato | JSON |
| Operación disparada | `requestNotification` |
| Política de fallo | 3 intentos con backoff exponencial (1 s, tope 30 s); agotados, a la deadLetter |

**Identidad: tu aplicación es `metadata.source`.** El servicio toma la aplicación que pide el envío
del `source` de la envoltura, que tiene que ser **el `code` de tu aplicación**. La aplicación no viaja
en `data`. Si `source` no corresponde a ninguna aplicación registrada, el mensaje va a la deadLetter sin
reintentos. El broker solo deja publicar en `notificationRequests` a las aplicaciones registradas, y esa
ACL hace de scope, equivalente a `notification:send`. Por política, cada emisor pone su propio code:
publicar en nombre de otra aplicación mandaría correos reales con su remitente.

**Payload esperado (`data`)** — la firma que hay que cumplir:

| Campo | Tipo | Notas |
|---|---|---|
| idempotencyKey | string | requerido; `^[A-Za-z0-9._:-]{8,128}$` |
| templateCode | string | requerido |
| recipients | string[] | requerido; 1–20 |
| variables | `{ name, value }[]` | opcional; ≤ 100 |

```json
{
  "metadata": {
    "eventId": "0e8b1c2d-3f4a-4b5c-8d6e-7f8091a2b3c4",
    "eventType": "NotificationRequested",
    "eventVersion": 1,
    "occurredAt": "2026-09-30T10:15:01.900Z",
    "source": "billing",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": null
  },
  "data": {
    "idempotencyKey": "inv-2026-0001",
    "templateCode": "invoice-issued",
    "recipients": ["Ana@Example.com", "luis@example.com"],
    "variables": [
      { "name": "invoiceNumber", "value": "F-0001" },
      { "name": "customerName",  "value": "Ana" },
      { "name": "amount",        "value": "10,00 €" }
    ]
  }
}
```

**Qué hace este servicio al recibirlo:** acepta la petición exactamente como `POST /notifications`. Deja
el mensaje en cola y lo envía en el siguiente ciclo del despacho. Si la `idempotencyKey` ya existe con el
mismo contenido, no hace nada, venga la anterior por HTTP o por evento.

**Si falta un campo o la petición se rechaza.** Un payload inválido o un rechazo de negocio
(`TEMPLATE_NOT_FOUND`, `RECIPIENT_SUPPRESSED`, `MISSING_TEMPLATE_VARIABLES`…) no tiene canal de
respuesta: agota los intentos y acaba en la deadLetter, que vigila operación. **No** se publica ningún
evento de rechazo. Para saber si tu petición se aceptó, consulta
[findMessageByIdempotencyKey](#findmessagebyidempotencykey): `404 MESSAGE_NOT_FOUND` significa que no
se aceptó.
