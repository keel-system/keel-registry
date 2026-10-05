---
service: notifications
version: 0.1.0
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
    path: /notifications/{idempotencyKey}
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
errors:
  - code: VALIDATION_ERROR
    http: 400
  - code: UNAUTHENTICATED
    http: 401
  - code: ACCESS_DENIED
    http: 403
  - code: DUPLICATE_TEMPLATE_VARIABLES
    http: 422
  - code: IDEMPOTENCY_KEY_REUSED
    http: 409
  - code: IDEMPOTENCY_KEY_IN_PROGRESS
    http: 409
  - code: SENDER_SETTINGS_NOT_CONFIGURED
    http: 404
  - code: TEMPLATE_NOT_FOUND
    http: 404
  - code: TEMPLATE_NOT_PUBLISHED
    http: 409
  - code: MISSING_TEMPLATE_VARIABLES
    http: 422
  - code: RECIPIENT_SUPPRESSED
    http: 422
  - code: MESSAGE_NOT_FOUND
    http: 404
  - code: RECIPIENT_NOT_IN_MESSAGE
    http: 422
  - code: SUPPRESSED_ADDRESS_ALREADY_EXISTS
    http: 409
  - code: CONCURRENT_MODIFICATION
    http: 409
---

# Integración con notifications

## Resumen

`notifications` envía correo transaccional en nombre de los sistemas de la organización, a partir de
plantillas que gestiona negocio en el propio servicio. Un sistema le **encarga un correo** (plantilla +
destinatarios + variables) por HTTP o publicando un evento, con el mismo contrato; el servicio lo acepta,
lo congela y lo entrega de forma asíncrona, y publica el desenlace (`EmailSent` o
`EmailDeliveryFailed`). La garantía central es que **el mismo correo no se envía dos veces**: cada
petición lleva una `idempotencyKey` del llamante que se recuerda para siempre. El servicio **no
reintenta** un envío fallido: el fallo es terminal y se publica. Además, el proveedor de correo le
notifica los rebotes, con los que mantiene la lista de direcciones suprimidas.

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los `scopes` que tu
   cliente tiene concedidos (ver tabla). Fija la audiencia `aud: notifications` (se valida: un token
   emitido para otro servicio responde `403 ACCESS_DENIED`).

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=notification:send notification:read&audience=notifications
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| billing | notification:send, notification:read | Facturación; pide correos y reconcilia sus peticiones. |
| shipping | notification:send, notification:read | Envíos; pide correos y reconcilia sus peticiones. |
| dispatch-only | notification:send | Sistema que solo pide correos, con el mínimo privilegio. |
| mail-provider | bounce:report | Proveedor de correo; notifica rebotes. |

Tu **identidad de llamante** (`requestedBy`) es el nombre de tu cliente máquina (`billing`): la fija el
servidor a partir de la credencial y no viaja en el cuerpo. Cada llamante tiene su propio espacio de
`idempotencyKey`, y solo puede reconciliar sus propias peticiones.

Todos los errores tienen la misma forma:

```json
{ "timestamp": "2026-10-05T09:21:07.482Z", "status": 409, "error": "Conflict", "code": "IDEMPOTENCY_KEY_REUSED", "message": "…", "details": null, "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1" }
```

El `code` es el contrato; el `message` es texto libre.

### requestNotification

Pide el envío de un correo a partir de una plantilla publicada. Responde `202`: la petición queda en
cola (`queued`) y el envío ocurre después, en menos de un par de minutos. **`202` no promete la
entrega**: el desenlace se publica como `EmailSent` o `EmailDeliveryFailed`, o se consulta con
`findMessageByIdempotencyKey`.

| | |
|---|---|
| Endpoint | `POST /api/v1/notifications` |
| Acceso | `service` — scopes `notification:send` |
| Idempotencia | sí — campo `idempotencyKey` del cuerpo, permanente y por llamante |
| También por evento | `NotificationRequested` (ver §Suscripciones), con el mismo contrato y la misma clave |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| idempotencyKey | string | requerido; `^[A-Za-z0-9._:-]{8,128}$`. Una por correo que quieres enviar; repítela en los reintentos. |
| templateCode | string | requerido; `^[a-z][a-z0-9-]{2,47}$`. Código de una plantilla con versión publicada. |
| recipients | string[] | requerido; 1–20 direcciones. Se normalizan a minúsculas y se deduplican conservando el orden de llegada; todas van en `To` del mismo correo. |
| variables | `{ name, value }[]` | opcional; ≤50. `value` ≤10000 caracteres. Las que la plantilla no declara se descartan en silencio; las `required` son obligatorias. |

```json
{
  "idempotencyKey": "inv-2026-0001",
  "templateCode": "invoice-issued",
  "recipients": ["ana@example.com", "luis@example.com"],
  "variables": [
    { "name": "invoiceNumber", "value": "F-0001" },
    { "name": "customerName", "value": "Ana" },
    { "name": "amount", "value": "10,00 €" }
  ]
}
```

**Response** `202` — el mensaje aceptado.

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | Id del mensaje; es también la parte local del `Message-ID` del correo. |
| requestedBy | string | Tu cliente máquina. |
| idempotencyKey | string | La que enviaste. |
| recipients | string[] | Normalizados. `[]` tras la purga de datos personales (18 meses). |
| renderedSubject | string \| null | Asunto ya renderizado. `null` tras la purga. |
| sender | `{ address, name, replyToAddress }` | Remitente congelado al aceptar; `name` y `replyToAddress` pueden ser `null`. |
| templateCode | string | |
| templateVersionNumber | int | Versión de la plantilla congelada al aceptar. |
| templateVersionId | uuid | |
| status | `queued \| sending \| sent \| failed` | |
| requestedAt | timestamp | |
| sendingSince, sentAt, failedAt | timestamp \| null | |
| failureReason | `delivery-rejected \| delivery-error \| stuck-in-sending \| recipient-suppressed` \| null | |
| personalDataPurgedAt | timestamp \| null | |

Los valores de las variables **no** se devuelven nunca (son sensibles).

```json
{
  "id": "6b1e2c9a-4f0d-4c7e-9a51-2d8f3e7b1c40",
  "requestedBy": "billing",
  "idempotencyKey": "inv-2026-0001",
  "recipients": ["ana@example.com", "luis@example.com"],
  "renderedSubject": "Factura F-0001",
  "sender": { "address": "avisos@notify.example.com", "name": "Acme Avisos", "replyToAddress": "soporte@example.com" },
  "templateCode": "invoice-issued",
  "templateVersionNumber": 1,
  "templateVersionId": "0c9d3f1a-7b2e-4e55-8a6d-41f0b9c2e7a3",
  "status": "queued",
  "requestedAt": "2026-10-05T09:21:07.482Z",
  "sendingSince": null,
  "sentAt": null,
  "failedAt": null,
  "failureReason": null,
  "personalDataPurgedAt": null
}
```

**Repetir la petición es seguro.** Con la misma `idempotencyKey` y el mismo contenido (misma plantilla,
mismo conjunto de destinatarios y mismos pares de variables declaradas, sin importar el orden) recibes
`202` con **el mismo mensaje** en su estado actual y no se envía otro correo, aunque desde entonces la
plantilla se haya retirado o un destinatario se haya suprimido. Tras la purga de datos personales basta
con que coincida `templateCode`. Si cambias el contenido, `409 IDEMPOTENCY_KEY_REUSED`.

**Un envío fallido no se reintenta solo.** Si quieres volver a intentarlo, pídelo con una
`idempotencyKey` **nueva**; con la misma recibirías el mensaje fallido.

**Orden de evaluación** (si fallan varias a la vez, ves el primer error): forma de la petición y
variables repetidas → repetición de la clave → remitente configurado → plantilla existente → plantilla
publicada → variables required → destinatarios suprimidos.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| VALIDATION_ERROR | 400 | El cuerpo no cumple las cotas (destinatarios, clave, formato de dirección). | Corregir input. |
| DUPLICATE_TEMPLATE_VARIABLES | 422 | `variables` trae dos valores con el mismo `name`. | Corregir input. |
| IDEMPOTENCY_KEY_REUSED | 409 | Ya existe un mensaje tuyo con esa clave y distinto contenido. | No reintentar; usa otra clave si es otro correo. |
| IDEMPOTENCY_KEY_IN_PROGRESS | 409 | Otra petición tuya con la misma clave se está aceptando a la vez. | Reintentar con la **misma** clave tras una pausa breve: recibirás el mensaje. |
| SENDER_SETTINGS_NOT_CONFIGURED | 404 | El servicio no tiene remitente configurado. | No reintentar; avisa al dueño del servicio. |
| TEMPLATE_NOT_FOUND | 404 | No existe una plantilla con ese `templateCode`. | No reintentar; corrige el código. |
| TEMPLATE_NOT_PUBLISHED | 409 | La plantilla no tiene versión activa. | No reintentar hasta que negocio la publique. |
| MISSING_TEMPLATE_VARIABLES | 422 | Falta una variable que la versión activa declara `required`. | Corregir input. |
| RECIPIENT_SUPPRESSED | 422 | Algún destinatario tiene una supresión activa. | Corregir input: quita la dirección. |
| UNAUTHENTICATED | 401 | Falta el token o no es válido. | Obtener un token y reintentar. |
| ACCESS_DENIED | 403 | El token no tiene `notification:send` o se emitió para otra audiencia. | No reintentar; revisa scopes y `aud`. |

### findMessageByIdempotencyKey

Reconcilia el estado de una petición tuya a partir de tu clave. Es la forma de saber qué pasó tras un
timeout, o si una petición enviada por evento se aceptó.

| | |
|---|---|
| Endpoint | `GET /api/v1/notifications/{idempotencyKey}` |
| Acceso | `service` — scopes `notification:read` |
| Idempotencia | no aplica (query) |

**Request** — path `idempotencyKey: string` (requerido). Solo encuentra mensajes **del propio
llamante**.

**Response** `200` — el mensaje, con la misma forma que la respuesta de `requestNotification`.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| MESSAGE_NOT_FOUND | 404 | No tienes ningún mensaje con esa clave: la petición no se aceptó. | No reintentar la consulta; si querías el correo, pídelo. |
| VALIDATION_ERROR | 400 | La clave no cumple su formato. | Corregir input. |
| UNAUTHENTICATED | 401 | Falta el token o no es válido. | Obtener un token y reintentar. |
| ACCESS_DENIED | 403 | El token no tiene `notification:read` (por ejemplo, `dispatch-only`) o es de otra audiencia. | No reintentar. |

### reportEmailBounce

El proveedor de correo notifica que un destinatario de un mensaje rebotó. Lo consume un **adaptador de
despliegue** que traduce el webhook real del proveedor a este contrato: el `messageId` es la parte
local (antes de la `@`) del `Message-ID` del correo rebotado.

| | |
|---|---|
| Endpoint | `POST /api/v1/messages/{messageId}/bounces` |
| Acceso | `service` — scopes `bounce:report` |
| Idempotencia | natural — repetir un rebote `permanent` sobre una dirección ya suprimida no cambia nada |

**Request** — path `messageId: uuid` (requerido).

| Campo | Tipo | Notas |
|---|---|---|
| recipient | string | requerido; la dirección que rebotó, uno de los `recipients` del mensaje. |
| bounceType | `permanent \| transient` | requerido. `permanent` suprime la dirección (motivo `hard-bounce`), aunque una persona la hubiera liberado; `transient` solo se registra en el log. |
| diagnostic | string | opcional; ≤1000. Se guarda en las notas de la supresión. |

```json
{ "recipient": "luis@example.com", "bounceType": "permanent", "diagnostic": "550 5.1.1 user unknown" }
```

**Response** `204`, sin cuerpo. El rebote no cambia el estado del mensaje.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| MESSAGE_NOT_FOUND | 404 | No existe un mensaje con ese `messageId`. | No reintentar. |
| RECIPIENT_NOT_IN_MESSAGE | 422 | La dirección no está entre los destinatarios del mensaje (también si sus datos ya se purgaron). | No reintentar; descarta el rebote. |
| SUPPRESSED_ADDRESS_ALREADY_EXISTS | 409 | Otra supresión de la misma dirección se creó a la vez. | Reintentar: el segundo intento encuentra la supresión activa y responde `204`. |
| CONCURRENT_MODIFICATION | 409 | Otra escritura modificó la supresión a la vez. | Reintentar. |
| VALIDATION_ERROR | 400 | Cuerpo mal formado (`bounceType` desconocido, dirección inválida). | Corregir input. |
| UNAUTHENTICATED | 401 | Falta el token o no es válido. | Obtener un token y reintentar. |
| ACCESS_DENIED | 403 | El token no tiene `bounce:report` o es de otra audiencia. | No reintentar. |

## Eventos

El contrato formal de los eventos está en [`asyncapi.yaml`](asyncapi.yaml) (visor:
[`asyncapi.html`](asyncapi.html)). El broker concreto y los nombres físicos de topic/cola se deciden al
desplegar; aquí solo aparecen los canales lógicos.

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "EmailSent",
    "eventVersion": 1,
    "occurredAt": "2026-10-05T09:22:03.114Z",
    "source": "notifications",
    "correlationId": null,
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": {
    "messageId": "6b1e2c9a-4f0d-4c7e-9a51-2d8f3e7b1c40",
    "requestedBy": "billing",
    "idempotencyKey": "inv-2026-0001",
    "templateCode": "invoice-issued",
    "templateVersionNumber": 1,
    "sentAt": "2026-10-05T09:22:03.114Z"
  }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`EmailSent`). Discriminador si el canal transporta varios tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| metadata.source | string | Servicio emisor: `notifications`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición (los desenlaces los produce el despacho programado, así que normalmente es `null`). |
| metadata.traceparent | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

Ningún evento lleva destinatarios ni variables: para correlacionarlo con tu petición usa
`requestedBy` + `idempotencyKey` (o `messageId`).

### EmailSent

El relay de correo aceptó el correo de un mensaje para al menos un destinatario. **No implica que
llegara al buzón**: los destinatarios que rechace el relay o que reboten después llegan como rebotes, no
cambian este evento.

| | |
|---|---|
| Canal | `notificationEvents` |
| Garantía | outbox — ningún evento se pierde si la transacción confirma |
| Emitido por | `sendQueuedMessage` (despacho interno) |

| Campo | Tipo | Notas |
|---|---|---|
| messageId | uuid | requerido |
| requestedBy | string | requerido; el llamante que pidió el correo |
| idempotencyKey | string | requerido |
| templateCode | string | requerido |
| templateVersionNumber | int | requerido |
| sentAt | timestamp | requerido |

```json
{ "messageId": "6b1e2c9a-4f0d-4c7e-9a51-2d8f3e7b1c40", "requestedBy": "billing", "idempotencyKey": "inv-2026-0001", "templateCode": "invoice-issued", "templateVersionNumber": 1, "sentAt": "2026-10-05T09:22:03.114Z" }
```

### EmailDeliveryFailed

Un mensaje acabó en `failed` y **no se reintentará**. Para volver a intentarlo, pide el correo con una
`idempotencyKey` nueva.

| | |
|---|---|
| Canal | `notificationEvents` |
| Garantía | outbox — ningún evento se pierde si la transacción confirma |
| Emitido por | `sendQueuedMessage`, `dispatchQueuedMessages` (rescate) |

| Campo | Tipo | Notas |
|---|---|---|
| messageId | uuid | requerido |
| requestedBy | string | requerido |
| idempotencyKey | string | requerido |
| templateCode | string | requerido |
| templateVersionNumber | int | requerido |
| failureReason | `delivery-rejected \| delivery-error \| stuck-in-sending \| recipient-suppressed` | requerido. `delivery-rejected`: el relay rechazó a todos los destinatarios. `delivery-error`: el intento falló sin respuesta concluyente. `stuck-in-sending`: el envío quedó a medias y se cerró sin reenviar (puede que el correo sí saliera). `recipient-suppressed`: al ir a enviarlo, un destinatario estaba suprimido y no se envió nada. |
| failedAt | timestamp | requerido |

```json
{ "messageId": "6b1e2c9a-4f0d-4c7e-9a51-2d8f3e7b1c40", "requestedBy": "billing", "idempotencyKey": "inv-2026-0001", "templateCode": "invoice-issued", "templateVersionNumber": 1, "failureReason": "recipient-suppressed", "failedAt": "2026-10-05T09:22:03.114Z" }
```

### Suscripciones

**Puertas de entrada (`nature: request`).** Son contrato público: las publica quien quiera encargar
trabajo a este servicio, y su payload no cambia sin subir la versión del contrato. Este servicio no
tiene suscripciones `nature: fact`.

### NotificationRequested

Pide el envío de un correo por el canal de eventos. Es la puerta hermana de
`POST /api/v1/notifications`: mismo contrato, misma `idempotencyKey` y mismo efecto, así que
**la misma clave por las dos puertas es el mismo envío**.

| | |
|---|---|
| Origen | `any-registered-system` — cualquier sistema registrado como cliente máquina con el scope `notification:send` (hoy `billing`, `shipping`, `dispatch-only`) |
| Canal | `notificationRequests` |
| Envoltura | `keel` (ver §Publicados → *Forma del mensaje*: la misma envoltura, ahora la publicas tú) |
| Identidad | `metadata.source` = tu nombre de cliente máquina (`billing`); es tu `requestedBy`. Un `source` que no resuelve a un cliente registrado con `notification:send` va a la deadLetter. |
| Deduplicación | `metadata.eventId` (una reentrega del mismo evento no tiene efecto) y, además, la `idempotencyKey` del payload, que es permanente |
| Operación disparada | `requestNotification` |
| onFailure | 3 intentos con espera exponencial (1000 ms iniciales, tope 30000 ms) y después deadLetter |

**Payload esperado** (`data`) — la firma que hay que cumplir:

| Campo | Tipo | Notas |
|---|---|---|
| idempotencyKey | string | requerido; `^[A-Za-z0-9._:-]{8,128}$` |
| templateCode | string | requerido |
| recipients | string[] | requerido; 1–20 |
| variables | `{ name, value }[]` | opcional; ≤50 |

```json
{
  "metadata": {
    "eventId": "3a7c1e0b-5d2f-4b8a-9c6e-1f0d2e3a4b5c",
    "eventType": "NotificationRequested",
    "eventVersion": 1,
    "occurredAt": "2026-10-05T09:21:07.482Z",
    "source": "billing",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": null
  },
  "data": {
    "idempotencyKey": "inv-2026-0001",
    "templateCode": "invoice-issued",
    "recipients": ["ana@example.com", "luis@example.com"],
    "variables": [
      { "name": "invoiceNumber", "value": "F-0001" },
      { "name": "customerName", "value": "Ana" },
      { "name": "amount", "value": "10,00 €" }
    ]
  }
}
```

**Qué hace este servicio al recibirlo:** acepta la petición como un mensaje `queued` y lo envía en el
siguiente ciclo del despacho; el desenlace sale como `EmailSent` o `EmailDeliveryFailed`.

**No hay respuesta por esta puerta.** Un rechazo de negocio (cualquier error declarado de
`requestNotification`: plantilla inexistente o sin publicar, variables que faltan, destinatario
suprimido…) es permanente y va a la deadLetter **sin gastar reintentos**; solo los fallos transitorios
se reintentan. Para saber si tu petición se aceptó, reconcilia con `findMessageByIdempotencyKey`: un
`404 MESSAGE_NOT_FOUND` significa que no se aceptó.

**Confianza.** La identidad por evento es política de desarrollo, no control técnico: el broker
autentica a quien publica, pero nada impide que un emisor registrado escriba el nombre de otro. Se
acepta mientras todos los emisores sean sistemas propios de la organización; un emisor externo exigiría
pasar a un destino por emisor con ACL.
