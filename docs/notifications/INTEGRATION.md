---
service: notifications
version: 1.6.0
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
    access: service message:read
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
  - code: APPLICATION_FORBIDDEN
    http: 403
  - code: APPLICATION_NOT_FOUND
    http: 404
  - code: TEMPLATE_NOT_FOUND
    http: 404
  - code: MESSAGE_NOT_FOUND
    http: 404
  - code: APPLICATION_SUSPENDED
    http: 409
  - code: TEMPLATE_NOT_PUBLISHED
    http: 409
  - code: IDEMPOTENCY_KEY_REUSED
    http: 409
  - code: IDEMPOTENCY_KEY_IN_PROGRESS
    http: 409
  - code: APPLICATION_ADDRESS_ALREADY_EXISTS
    http: 409
  - code: CONCURRENT_MODIFICATION
    http: 409
  - code: RECIPIENT_SUPPRESSED
    http: 422
  - code: MISSING_TEMPLATE_VARIABLES
    http: 422
---

# Integración con notifications

## Resumen

`notifications` emite el correo transaccional de las aplicaciones de la organización. Un servidor que
necesita avisar a una persona —una factura emitida, un envío despachado— no compone el correo ni
habla con el proveedor de correo: nombra una plantilla y aporta sus valores, y este servicio resuelve
la versión activa, valida las variables, comprueba la lista de supresión, renderiza y entrega. Ofrece
a otros servidores **dos vías equivalentes para pedir un envío** —un endpoint HTTP y un canal de
eventos, con el mismo contrato de entrada—, un endpoint para **consultar el desenlace** de lo que
pidieron, un endpoint para que el proveedor de correo **notifique un rebote asíncrono**, y dos
**eventos informativos** con el resultado de cada correo.

Dos cosas que conviene saber antes de integrarse, porque condicionan el código del consumidor. La
primera: **la aplicación en cuyo nombre se pide un envío nunca viaja en el cuerpo** — se resuelve de
la credencial en HTTP y de `metadata.source` en el canal —, así que no hay ningún campo que rellenar
para identificarse. La segunda: **el fallo de envío es terminal**; este servicio no reintenta nunca,
y quien quiera recuperar un aviso fallido debe volver a pedirlo con otra clave de idempotencia.

Contratos formales: [`openapi.yaml`](openapi.yaml) (HTTP) y [`asyncapi.yaml`](asyncapi.yaml)
(eventos). Este documento es la prosa de integración; no los duplica.

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno. **Tu `clientId`
   se corresponde con el `code` de una `Application` registrada en el servicio**: es de ahí de donde
   se resuelve en nombre de quién se pide cada envío, así que antes de integrarte tiene que existir
   esa aplicación con su remitente verificado. Pídesela también al dueño del servicio.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los `scopes` que tu
   cliente tiene concedidos (ver tabla). Fija la audiencia `aud: notifications`, **que se valida**:
   un token cuya audiencia no nombre a este servicio recibe `403`, no `401`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=billing&client_secret=...&scope=notification:send%20message:read&audience=notifications
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| `billing` | `notification:send`, `message:read` | Facturación: pide envíos y consulta el desenlace de los suyos. |
| `shipping` | `notification:send`, `message:read` | Logística: pide envíos y consulta el desenlace de los suyos. |
| `dispatch-only` | `notification:send` | Consumidor que solo dispara notificaciones y nunca las consulta. |
| `mail-provider` | `bounce:report` | El relay de correo, que notifica de forma asíncrona un rebote. |

**Mínimo privilegio, y no es decorativo.** Pedir un envío y leer los mensajes son capacidades
distintas: `message:read` expone los destinatarios y el asunto renderizado, que son datos personales.
Si tu servicio solo dispara avisos y no reacciona a su desenlace, pide únicamente `notification:send`.

### requestNotification

| | |
|---|---|
| Endpoint | `POST /api/v1/notifications` |
| Éxito | `202 Accepted` |
| Acceso | `service` — scopes `notification:send` |
| Idempotencia | sí — campo `idempotencyKey` del cuerpo (no cabecera). **Permanente: no caduca.** |
| Alternativa por eventos | publicar `NotificationRequested` (ver §Suscripciones); mismo contrato de entrada |

Acepta la petición, la valida entera y deja el mensaje **encolado**. El `202` significa que el
encargo se aceptó, **no que el correo se haya entregado**: el envío real lo hace un ciclo interno
que corre cada minuto. Para saber qué pasó, consulta `findMessageByIdempotencyKey` o escucha
`EmailSent` / `EmailDeliveryFailed`.

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| `templateCode` | `TemplateCode` | requerido; código de la plantilla dentro de tu aplicación. Se normaliza a minúsculas. |
| `idempotencyKey` | `IdempotencyKey` | requerido; 8–128 caracteres. La eliges tú (ver abajo). |
| `recipients` | `EmailAddress[]` | requerido; entre 1 y 20. Se normalizan a minúsculas y se deduplican. |
| `variables` | `TemplateVariableValue[]` | opcional; hasta 50. Cada elemento `{ name, value }`, `value` hasta 2000 caracteres. |

**`applicationCode` no viaja en el cuerpo.** No es que puedas omitirlo: **no forma parte de la
petición**. Lo estampa el servidor desde el `serviceClient` de tu credencial —cada cliente máquina se
corresponde con una `Application` registrada—, así que no puedes pedir un envío en nombre de otra
aplicación ni desde su remitente verificado. Si lo mandas de todos modos, se **ignora en silencio**:
no hay error que devolver, porque no hay nada que comprobar. (Hasta la versión 1.3.0 el campo sí se
aceptaba y un valor ajeno daba `403 APPLICATION_FORBIDDEN`; ese código ya no lo emite esta
operación.)

Las variables que la versión de plantilla no declara **se descartan sin error**; las que declara como
`required` deben venir con valor no vacío o la petición se rechaza.

```json
{
  "templateCode": "invoiceready",
  "idempotencyKey": "inv-2026-000123",
  "recipients": ["cliente@example.com"],
  "variables": [
    { "name": "customerName", "value": "Ana" },
    { "name": "invoiceNumber", "value": "F-000123" }
  ]
}
```

**Response** — `202` con el registro del mensaje aceptado.

| Campo | Tipo | Notas |
|---|---|---|
| `id` | `uuid` | Identificador del mensaje; es lo que necesita `reportEmailBounce`. |
| `applicationId` | `uuid` | Referencia por id a tu `Application`. |
| `idempotencyKey` | `IdempotencyKey` | La que enviaste. |
| `recipients` | `EmailAddress[]` | Ya normalizados y deduplicados: pueden ser menos que los que mandaste. |
| `renderedSubject` | `string` | El asunto **ya resuelto** con tus variables, tal como lo verá el destinatario. |
| `senderAddress` | `EmailAddress` | Remitente efectivo, copiado de tu aplicación en este momento. |
| `senderName` | `string \| null` | Nombre visible que acompaña al remitente, copiado a la vez que `senderAddress`. `null` si tu aplicación no declara nombre visible: el correo sale solo con la dirección. Las dos mitades se congelan juntas, así que cambiar el remitente de tu aplicación **no** reescribe las de un mensaje ya aceptado. |
| `templateCode` | `TemplateCode` | Código de la plantilla resuelta. |
| `templateVersionNumber` | `int` | Número de la versión activa en el momento de aceptar. |
| `templateVersionId` | `uuid` | Id de esa versión. |
| `status` | `MessageStatus` | `queued` \| `sending` \| `sent` \| `failed`. Aquí siempre `queued`. |
| `failureReason` | `string \| null` | Motivo del fallo; `null` mientras no lo haya. |
| `requestedAt` | `timestamp` | Instante UTC en que se aceptó. |
| `sendingSince` | `timestamp \| null` | Instante en que un ciclo de despacho tomó el mensaje; `null` mientras siga en `queued`. |
| `sentAt` | `timestamp \| null` | Instante en que el relay aceptó el mensaje. |
| `personalDataPurgedAt` | `timestamp \| null` | Sellado cuando se borran sus datos personales (18 meses). |

El campo `variableValues` **nunca viaja en ninguna respuesta**: es sensible porque puede contener
datos personales.

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "applicationId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "idempotencyKey": "inv-2026-000123",
  "recipients": ["cliente@example.com"],
  "renderedSubject": "Tu factura F-000123 está lista",
  "senderAddress": "facturas@acme.com",
  "senderName": "Acme Facturación",
  "templateCode": "invoiceready",
  "templateVersionNumber": 1,
  "templateVersionId": "b1f4c2d8-5e3a-4c17-9d02-8a6b3e5f1c40",
  "status": "queued",
  "failureReason": null,
  "requestedAt": "2026-08-20T09:21:07.482Z",
  "sendingSince": null,
  "sentAt": null,
  "personalDataPurgedAt": null
}
```

**Cómo elegir la `idempotencyKey`, y qué promete.** Es tu guarda contra mandarle dos correos a una
persona real, así que derívala del hecho de negocio que motiva el aviso (`inv-2026-000123`), nunca de
un aleatorio por intento: un reintento tuyo tiene que reusar exactamente la misma clave. La guarda es
**permanente** —no caduca nunca— y vive por aplicación, así que la misma cadena en dos aplicaciones
distintas son dos claves distintas.

Repetir una petición con la misma clave y **el mismo contenido** devuelve `202` con el mismo cuerpo,
sin encolar un segundo correo. «Mismo contenido» son exactamente tres cosas: el `templateCode`, el
**conjunto** de destinatarios ya normalizados y deduplicados, y el **conjunto** de pares
nombre/valor de las variables declaradas — comparados por igualdad de valores y **sin importar el
orden**. Reordenar los destinatarios no es contenido distinto. Con la misma clave y contenido
distinto recibes `409 IDEMPOTENCY_KEY_REUSED`.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | La petición no cumple las cotas del input (0 o más de 20 destinatarios, clave de menos de 8 caracteres, dirección mal formada). | Corregir el input; no reintentar igual. |
| `APPLICATION_NOT_FOUND` | 404 | Tu identidad no corresponde a ninguna `Application` registrada. | No reintentar: pide al dueño del servicio que registre tu aplicación. |
| `TEMPLATE_NOT_FOUND` | 404 | Tu aplicación no tiene ninguna plantilla con ese código. | No reintentar; revisa el `templateCode`. |
| `APPLICATION_SUSPENDED` | 409 | Tu aplicación está suspendida y no puede originar envíos. | No reintentar; es una decisión administrativa, habla con el dueño del servicio. |
| `TEMPLATE_NOT_PUBLISHED` | 409 | La plantilla existe pero no tiene versión activa con la que componer el mensaje. | No reintentar; alguien debe publicar o reactivar una versión. |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Esa clave ya se usó en tu aplicación para una petición con contenido distinto. | No reintentar: usa una clave nueva, o corrige el contenido para que coincida. |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | 409 | Otra petición con la misma clave está confirmando en este mismo instante. | **Reintentable** tras una espera breve: la ganadora aún no había confirmado. |
| `RECIPIENT_SUPPRESSED` | 422 | Alguno de los destinatarios está en la lista de supresión de tu aplicación por rebote duro o queja. | No reintentar. Basta un destinatario suprimido para rechazar la petición entera; quita esa dirección o pide que la liberen. |
| `MISSING_TEMPLATE_VARIABLES` | 422 | Falta el valor de alguna variable que la versión activa declara como `required`. | Corregir el input añadiendo la variable. |

Cualquier `5xx` o timeout **es reintentable**: reusa la misma `idempotencyKey` y la guarda impide el
doble envío.

### findMessageByIdempotencyKey

| | |
|---|---|
| Endpoint | `GET /api/v1/notifications/{applicationCode}/{idempotencyKey}` |
| Éxito | `200 OK` |
| Acceso | `service` — scopes `message:read` |
| Idempotencia | no aplica (query) |

**Es la fuente de verdad del desenlace de un envío.** Los eventos `EmailSent` y
`EmailDeliveryFailed` son informativos y con `best-effort` pueden perderse; esta consulta no.

**Request** — path `applicationCode: ApplicationCode` (requerido) e `idempotencyKey: IdempotencyKey`
(requerido). El `applicationCode` del path debe ser el de la aplicación que tu credencial representa:
pedir el de otra devuelve `403`, **no** una respuesta vacía.

```
GET /api/v1/notifications/billing/inv-2026-000123
Authorization: Bearer <access_token>
```

**Response** — `200` con el mismo cuerpo de `EmailMessage` documentado en
[`requestNotification`](#requestnotification), con el `status` y el `sentAt`/`failureReason` al día.

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "applicationId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "idempotencyKey": "inv-2026-000123",
  "recipients": ["cliente@example.com"],
  "renderedSubject": "Tu factura F-000123 está lista",
  "senderAddress": "facturas@acme.com",
  "senderName": "Acme Facturación",
  "templateCode": "invoiceready",
  "templateVersionNumber": 1,
  "templateVersionId": "b1f4c2d8-5e3a-4c17-9d02-8a6b3e5f1c40",
  "status": "sent",
  "failureReason": null,
  "requestedAt": "2026-08-20T09:21:07.482Z",
  "sendingSince": "2026-08-20T09:22:01.004Z",
  "sentAt": "2026-08-20T09:22:03.117Z",
  "personalDataPurgedAt": null
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `APPLICATION_FORBIDDEN` | 403 | Tu credencial no representa a la aplicación del path. | No reintentar: solo puedes consultar los mensajes de tu propia aplicación. |
| `APPLICATION_NOT_FOUND` | 404 | No existe una aplicación con ese código. | No reintentar; revisa el `applicationCode`. |
| `MESSAGE_NOT_FOUND` | 404 | Esa aplicación no ha originado ningún mensaje con esa clave. | No reintentar como error: significa que tu petición **no llegó a aceptarse**. Vuelve a pedirla. |

**Nota sobre la retención.** A los 18 meses se borran los datos personales del mensaje y se conserva
la fila: la consulta seguirá respondiendo `200`, pero con `recipients` vacío, `renderedSubject` vacío
y `personalDataPurgedAt` informado. El `status` y el desenlace se conservan siempre.

### reportEmailBounce

| | |
|---|---|
| Endpoint | `POST /api/v1/messages/{messageId}/bounces` |
| Éxito | `204 No Content` |
| Acceso | `service` — scopes `bounce:report` |
| Idempotencia | sin clave; el efecto es convergente (no duplica el registro de supresión) |

**Este endpoint lo consume el proveedor de correo, no un servicio de negocio.** Es la puerta por la
que entra el rebote que llega **después** de que el relay aceptara el mensaje —minutos u horas más
tarde—, distinto del rechazo síncrono que el servicio ya detecta al enviar. **Es el camino
determinista del rebote duro**: el rechazo síncrono solo alimenta la lista si el relay dice *qué
dirección* rebotó, cosa que no todos hacen, así que sin este canal la regla de suprimir por rebote
duro no sería implementable ni verificable.

**Request** — path `messageId: uuid` (requerido).

| Campo | Tipo | Notas |
|---|---|---|
| `address` | `EmailAddress` | requerido; la dirección que rebotó. Se normaliza a minúsculas. |
| `permanent` | `boolean` | requerido. `true` = rebote duro; `false` = rebote blando. |
| `reason` | `string` | opcional; hasta 500 caracteres. El motivo tal como lo reporta el proveedor. |

```json
{
  "address": "cliente@example.com",
  "permanent": true,
  "reason": "550 5.1.1 mailbox does not exist"
}
```

**Response** — `204` sin cuerpo.

Qué provoca: un rebote **permanente** añade la dirección a la lista de supresión de la aplicación del
mensaje con motivo `hard-bounce`, y **reactiva** un registro que estuviera liberado. Un rebote
**blando NO cambia ningún estado persistente**: no suprime la dirección, no toca el `status` del
mensaje y no escribe su motivo en ningún campo —`failureReason` pertenece al desenlace del envío, no
a un rebote posterior—; queda solo en el log del servicio, así que no esperes observarlo por la API.
En ningún caso cambia el `status` del mensaje: el rebote es un hecho posterior a que el relay lo
aceptara, así que un mensaje `sent` sigue siendo `sent`.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `VALIDATION_ERROR` | 400 | Falta `permanent`, o la dirección está mal formada. | Corregir el input. |
| `MESSAGE_NOT_FOUND` | 404 | No existe un mensaje con ese identificador. | No reintentar; el `messageId` es incorrecto o el mensaje nunca existió. |
| `APPLICATION_ADDRESS_ALREADY_EXISTS` | 409 | Dos rebotes simultáneos sobre la misma dirección de la misma aplicación, y el segundo chocó con la clave natural. | **Reintentable** una vez: el efecto es convergente y el registro ya existe, así que el reintento resuelve o confirma el estado. |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura sobre la misma raíz de agregado confirmó entre su lectura y este guardado. | **Reintentable** tras una espera breve. |

## Eventos

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "EmailSent",
    "eventVersion": 1,
    "occurredAt": "2026-08-20T09:22:03.117Z",
    "source": "notifications",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": {
    "messageId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
    "applicationCode": "billing",
    "templateCode": "invoiceready",
    "idempotencyKey": "inv-2026-000123",
    "recipientCount": 1,
    "sentAt": "2026-08-20T09:22:03.117Z"
  }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `metadata.eventId` | `uuid` | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| `metadata.eventType` | `string` | Nombre del evento (`EmailSent`). Discriminador si el canal transporta varios tipos. |
| `metadata.eventVersion` | `int` | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| `metadata.occurredAt` | `timestamp` | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| `metadata.source` | `string` | Servicio emisor: `notifications`. |
| `metadata.correlationId` | `string \| null` | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| `data` | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Garantía de entrega de los dos eventos: `best-effort`.** No hay outbox. Si la transacción confirma
con el broker caído, el correo salió y el evento no existe — y nadie se entera nunca. **No los uses
como fuente de verdad**: son un empujón para no sondear. Si tu lógica de negocio depende de saber con
certeza qué pasó con un envío, consulta
[`findMessageByIdempotencyKey`](#findmessagebyidempotencykey). Diséñalos también como
**at-least-once**: deduplica por `metadata.eventId`.

### EmailSent

| | |
|---|---|
| Canal lógico | `notificationEvents` |
| Garantía | `best-effort` — puede perderse aunque el correo sí haya salido |
| Emitido por | `sendQueuedMessage` |

El relay aceptó el mensaje para su entrega. Ojo con lo que **no** significa: que el relay lo acepte no
garantiza que el destinatario lo reciba — un rebote posterior llega por `reportEmailBounce`, no por
este evento.

`data`:

| Campo | Tipo | Notas |
|---|---|---|
| `messageId` | `uuid` | requerido |
| `applicationCode` | `ApplicationCode` | requerido |
| `templateCode` | `TemplateCode` | requerido |
| `idempotencyKey` | `IdempotencyKey` | requerido; es la clave con la que tú pediste el envío |
| `recipientCount` | `int` | requerido; mínimo 1. Es el **número** de destinatarios, no las direcciones: el evento no lleva datos personales |
| `sentAt` | `timestamp` | requerido |

```json
{
  "messageId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "applicationCode": "billing",
  "templateCode": "invoiceready",
  "idempotencyKey": "inv-2026-000123",
  "recipientCount": 1,
  "sentAt": "2026-08-20T09:22:03.117Z"
}
```

### EmailDeliveryFailed

| | |
|---|---|
| Canal lógico | `notificationEvents` |
| Garantía | `best-effort` — tampoco está garantizado |
| Emitido por | `sendQueuedMessage`, `dispatchQueuedMessages` |

El mensaje no llegó a entregarse. **El fallo es terminal por diseño**: este servicio no reintenta.
Si el aviso importa, quien lo pidió debe volver a pedirlo **con otra clave de idempotencia** —
reusar la misma devolvería el mensaje fallido sin enviar nada.

**Lo publican dos vías, y no afirman lo mismo.** Léelo en `failureReason` antes de reaccionar:

| Vía | Qué afirma | Qué puedes concluir |
|---|---|---|
| `sendQueuedMessage` — el relay rechazó el mensaje o no respondió | el correo **no salió** | re-pedir el envío es seguro |
| `dispatchQueuedMessages` — **rescate** de un mensaje atascado en `sending` más de 15 minutos | **no se sabe** si salió: el despachador pudo morir después de que el relay aceptara el mensaje | re-pedir el envío **puede duplicar un correo ya entregado a una persona real** |

Cada evento sale únicamente si su transición a `failed` se aplicó, así que un mensaje no se anuncia
dos veces. Aun así, `best-effort` sobre un canal at-least-once no garantiza entrega única: tu
consumidor debe deduplicar por `metadata.eventId`. Si necesitas certeza del desenlace antes de
actuar, no reacciones a este evento: consulta `findMessageByIdempotencyKey`.

`data`:

| Campo | Tipo | Notas |
|---|---|---|
| `messageId` | `uuid` | requerido |
| `applicationCode` | `ApplicationCode` | requerido |
| `templateCode` | `TemplateCode` | requerido |
| `idempotencyKey` | `IdempotencyKey` | requerido |
| `failureReason` | `string` | requerido; hasta 500 caracteres |
| `failedAt` | `timestamp` | requerido |

```json
{
  "messageId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "applicationCode": "billing",
  "templateCode": "invoiceready",
  "idempotencyKey": "inv-2026-000123",
  "failureReason": "550 5.1.1 mailbox does not exist",
  "failedAt": "2026-08-20T09:22:03.117Z"
}
```

### Suscripciones

Este servicio tiene **una** suscripción, y es de naturaleza `request`: una **puerta de entrada**. Su
payload es contrato público que alguien cumple para encargarnos trabajo, así que puedes apoyarte en
él para integrarte. No hay ninguna suscripción `nature: fact` — este servicio no reacciona a hechos
ajenos.

### NotificationRequested

| | |
|---|---|
| Naturaleza | `request` — puerta de entrada; publicar esto **nos encarga** un envío |
| Canal lógico | `notificationRequests` |
| Origen (`source`) | `any-registered-system` — cualquier servidor con una `Application` registrada |
| Dispara | `requestNotification` |
| Envelope | `keel` — la misma envoltura de §Publicados, ahora en sentido contrario |
| Formato | `json`; campos desconocidos se ignoran |
| Clave de deduplicación | `metadata.eventId` del sobre |

**Es la vía asíncrona equivalente a `POST /notifications`**, con el mismo contrato de entrada.
Cuándo preferirla al endpoint HTTP: cuando tengas **volumen** (el canal absorbe picos y ya trae
reintentos y cola de descartes) o cuando no necesites respuesta inmediata. Cuándo preferir HTTP:
cuando necesites saber en el acto si la petición se aceptó, y con qué error si no.

**Cómo te identificas.** No hay campo de aplicación en el payload: la aplicación en cuyo nombre se
pide el envío se resuelve de **`metadata.source`** del sobre, que debe ser el `code` de una
`Application` registrada. Un mensaje cuyo `source` no corresponda a ninguna **va a la cola de
descartes**, no se descarta en silencio.

> **Lo que esto asume, dicho en voz alta.** El broker autentica a los emisores al publicar, pero cada
> uno estampa su propio nombre en `metadata.source`: es política de desarrollo, no un control
> técnico. Un emisor registrado podría pedir un envío en nombre de otra aplicación y usar su
> remitente verificado. Se acepta a sabiendas mientras todos los emisores sean sistemas propios. Si
> tu integración es de un tercero, o necesitas no-repudio del origen, dilo al dueño del servicio
> antes de integrarte: la respuesta es cambiar la identidad a una cabecera que garantice el broker.

**Payload esperado** — es el contenido de `data`. Es la firma que hay que cumplir para que hagamos el
trabajo:

| Campo | Tipo | Obligatorio | Si falta |
|---|---|---|---|
| `templateCode` | `TemplateCode` | sí | El mensaje va a la cola de descartes sin reintentar. |
| `idempotencyKey` | `IdempotencyKey` | sí | Íd. |
| `recipients` | `EmailAddress[]` | sí, entre 1 y 20 | Íd. |
| `variables` | `TemplateVariableValue[]` | no, hasta 50 | Si falta una que la plantilla declara `required`, el mensaje va a la cola de descartes. |

```json
{
  "metadata": {
    "eventId": "c4a1e9d7-0b32-4f88-9a10-2e6d4b7c8f31",
    "eventType": "NotificationRequested",
    "eventVersion": 1,
    "occurredAt": "2026-08-20T09:21:07.482Z",
    "source": "billing",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": {
    "templateCode": "invoiceready",
    "idempotencyKey": "inv-2026-000123",
    "recipients": ["cliente@example.com"],
    "variables": [
      { "name": "customerName", "value": "Ana" },
      { "name": "invoiceNumber", "value": "F-000123" }
    ]
  }
}
```

**Qué hace este servicio al recibirlo.** Resuelve tu aplicación desde `metadata.source`, comprueba la
repetición por la pareja aplicación + `idempotencyKey`, valida que la aplicación no esté suspendida,
resuelve la versión activa de la plantilla, comprueba que ningún destinatario esté suprimido, valida
las variables `required`, renderiza el asunto y **deja el mensaje encolado**. El correo real sale en
el ciclo de despacho siguiente.

**Política de fallo — los reintentos son solo para lo transitorio.**

| | |
|---|---|
| Reintentos | 3 intentos, backoff exponencial (1000 ms inicial, 10000 ms máximo) |
| Cola de descartes | sí |

Un fallo de negocio **no se reintenta**: la plantilla seguirá sin existir, el destinatario seguirá
suprimido y la variable seguirá faltando. Van **directos a la cola de descartes**, sin consumir
ningún intento: `TEMPLATE_NOT_FOUND`, `TEMPLATE_NOT_PUBLISHED`, `RECIPIENT_SUPPRESSED`,
`MISSING_TEMPLATE_VARIABLES`, `APPLICATION_SUSPENDED`, `IDEMPOTENCY_KEY_REUSED` y
`APPLICATION_NOT_FOUND` (emisor sin aplicación registrada). Se reintenta todo lo demás:
indisponibilidad del almacén, `IDEMPOTENCY_KEY_IN_PROGRESS` y cualquier fallo de infraestructura.

**Deduplicación, que necesitas por partida doble.** El canal es at-least-once por definición. La
primera barrera es `metadata.eventId`, que corta la reentrega del broker antes de tocar el dominio;
la segunda es tu `idempotencyKey`, que es permanente y cubre el caso que la primera no cubre: que
republiques el mismo encargo con un sobre nuevo. Por eso **la clave debe derivarse del hecho de
negocio**, no generarse en cada publicación.

**Confirmar que el encargo llegó.** Publicar en un canal no te da respuesta. Si necesitas saber que
la petición se aceptó, consulta
[`findMessageByIdempotencyKey`](#findmessagebyidempotencykey) con la misma clave: un `404
MESSAGE_NOT_FOUND` significa que no se aceptó (probablemente esté en la cola de descartes).
