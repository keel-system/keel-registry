---
service: payment
version: 0.1.0
domain: commerce
basePath: /api/v1
m2mAuth:
  protocol: client-credentials
  audience: payment
  validateAudience: true
endpoints:
  - name: requestCharge
    method: POST
    path: /payments
    access: service payment:charge
  - name: getPayment
    method: GET
    path: /payments/{id}
    access: service payment:read
  - name: getPaymentByChargeRequest
    method: GET
    path: /payments/by-charge-request/{chargeRequestId}
    access: service payment:read
  - name: listPaymentsByCustomer
    method: GET
    path: /payments
    access: service payment:read
  - name: capturePayment
    method: POST
    path: /payments/{id}/capture
    access: service payment:charge
  - name: cancelPayment
    method: POST
    path: /payments/{id}/cancel
    access: service payment:charge
  - name: refundPayment
    method: POST
    path: /payments/{id}/refunds
    access: service payment:refund
  - name: savePaymentMethod
    method: POST
    path: /payment-methods
    access: service payment-method:write
  - name: listPaymentMethods
    method: GET
    path: /payment-methods
    access: service payment-method:read
  - name: setDefaultPaymentMethod
    method: POST
    path: /payment-methods/{paymentMethodId}/default
    access: service payment-method:write
  - name: removePaymentMethod
    method: DELETE
    path: /payment-methods/{paymentMethodId}
    access: service payment-method:write
events:
  envelope: keel
  published:
    - name: PaymentAuthorized
      channel: paymentEvents
    - name: PaymentActionRequired
      channel: paymentEvents
    - name: PaymentCaptured
      channel: paymentEvents
    - name: PaymentFailed
      channel: paymentEvents
    - name: PaymentCanceled
      channel: paymentEvents
    - name: PaymentRefunded
      channel: paymentEvents
    - name: PaymentActionRejected
      channel: paymentEvents
  consumed: []
errors:
  - code: PAYMENT_SOURCE_INVALID
    http: 400
  - code: CURRENCY_UNKNOWN
    http: 400
  - code: AMOUNT_SCALE_INVALID
    http: 400
  - code: PAYMENT_METHOD_UNAVAILABLE
    http: 422
  - code: CHARGE_REQUEST_CONFLICT
    http: 409
  - code: PAYMENT_NOT_FOUND
    http: 404
  - code: PAYMENT_NOT_CAPTURABLE
    http: 409
  - code: AUTHORIZATION_EXPIRED
    http: 409
  - code: CAPTURE_REJECTED
    http: 422
  - code: CONCURRENT_MODIFICATION
    http: 409
  - code: PAYMENT_NOT_CANCELABLE
    http: 409
  - code: CANCEL_REJECTED
    http: 422
  - code: PAYMENT_NOT_REFUNDABLE
    http: 409
  - code: REFUND_AMOUNT_EXCEEDED
    http: 422
  - code: REFUND_REQUEST_CONFLICT
    http: 409
  - code: REFUND_REJECTED
    http: 422
  - code: IDEMPOTENCY_KEY_REQUIRED
    http: 400
  - code: PAYMENT_METHOD_LIMIT_REACHED
    http: 422
  - code: PAYMENT_METHOD_AUTHENTICATION_REQUIRED
    http: 422
  - code: PAYMENT_METHOD_REJECTED
    http: 422
  - code: GATEWAY_UNAVAILABLE
    http: 503
  - code: PAYMENT_METHOD_NOT_FOUND
    http: 404
---

# Integración con payment

## Resumen

`payment` (dominio `commerce`) cobra con tarjeta, a través de una pasarela intercambiable, los
importes que le piden los servidores de una aplicación. Autoriza un cobro y lo captura o lo anula
después, admite la autenticación del cliente (3DS) y devuelve lo cobrado en una o varias veces. Guarda
también medios de pago por titular para cobrar sin que esté presente.

Tu servidor le da una clave de negocio por cobro (`chargeRequestId`) y por devolución
(`refundRequestId`), y una referencia opaca del titular (`customerRef`, por ejemplo el `sub` de tu
usuario). El servicio no conoce pedidos ni usuarios.

El navegador del comprador **nunca** llama a este servicio. Produce el token de un solo uso con el
componente de la pasarela y se lo entrega a tu backend, que es quien pide el cobro.

Los desenlaces que llegan después de la respuesta se publican como eventos: un 3DS completado, una
respuesta perdida que resuelve el barrido o una autorización que caduca.

Contratos formales: [`openapi.yaml`](openapi.yaml) (HTTP) y [`asyncapi.yaml`](asyncapi.yaml) (eventos).

**Convenciones que valen para todo el documento:**
- **Importes.** Número JSON en unidades mayores de la moneda (`25.90`), comparado por valor.
  Cuántos decimales admite lo fija la moneda (ISO 4217: JPY 0, EUR 2, KWD 3). Uno de más se rechaza
  con `400 AMOUNT_SCALE_INVALID` y nunca se redondea.
- **Nulos.** Un campo sin valor viaja como `null`; una colección vacía, como `[]`.
- **Instantes.** ISO-8601 en UTC.
- **Claves.** Distinguen mayúsculas: `ch-001` y `CH-001` son claves distintas.
- **Cuerpo de error.** Siempre `{timestamp, status, error, code, message, details, correlationId}`.
  El contrato es el `code`.

## Endpoints expuestos a otros servidores

Todos los endpoints del servicio son de esta sección y se consumen con un **token de cliente
máquina** (OAuth2 client credentials), no con token de usuario. Para obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los scopes que tu cliente
   tiene concedidos. Fija la audiencia `aud: payment`, porque se valida: un token emitido para otro
   servicio responde `403 ACCESS_DENIED`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=payment:charge payment:read&audience=payment
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| checkout | payment:charge, payment:read, payment-method:write, payment-method:read | Backend del checkout: cobra, captura o anula y gestiona los medios del titular. No devuelve dinero. |
| backoffice | payment:read, payment:refund | Backend de atención al cliente: consulta cobros y hace devoluciones. |

Los dos clientes son el ejemplo de mínimo privilegio del diseño; el dueño del servicio los da de alta
con el nombre que corresponda en tu sistema.

Toda operación puede responder además, sin que su tabla lo repita:
- `400 VALIDATION_ERROR` si la petición no cumple la forma (tipos, longitudes, patrones, campos
  obligatorios). Acción: corregir el input.
- `401 UNAUTHENTICATED` sin token válido. Acción: obtener un token y reintentar.
- `403 ACCESS_DENIED` sin el scope exigido o con un token de otra audiencia. Acción: no reintentar;
  pedir el scope.

**El objeto `Payment`** es lo que devuelven las operaciones de cobro y las lecturas:

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | Id del cobro en este servicio. |
| chargeRequestId | string | Tu clave de negocio (1–100, `[A-Za-z0-9._:-]`). Única: la guarda contra el doble cargo. |
| customerRef | string | Tu referencia del titular (1–200). |
| amount | decimal | Importe autorizado; al capturar se cobra entero. |
| currency | string | ISO 4217. |
| status | enum | `pending`, `actionRequired`, `authorized`, `capturing`, `canceling`, `captured`, `refunding`, `refunded`, `canceled`, `failed`. |
| gatewayPaymentId | string \| null | Id de la pasarela; `null` mientras no ha contestado. |
| awaitingSince | timestamp \| null | Desde cuándo espera un desenlace; solo en `pending`, `actionRequired`, `capturing`, `canceling`, `refunding`. |
| failureReason | enum \| null | Solo en `failed`: `declined`, `insufficientFunds`, `expiredCard`, `authenticationFailed`, `fraudSuspected`, `invalidPaymentMethod`, `notReceived`, `processingError`. |
| customerAction | objeto \| null | Acción opaca para el componente de la pasarela en el navegador (3DS). Solo en `actionRequired` y mientras se anula un cobro que venía de ahí. |
| actionRequiredSince | timestamp \| null | Desde cuándo espera el 3DS; el servicio anula el cobro pasadas 24 h (parámetro de despliegue). |
| cancelOrigin | enum \| null | Solo en `canceling`: `authorized` o `actionRequired`, el estado al que vuelve si la pasarela rechaza la anulación. |
| paymentMethodId | uuid \| null | Medio guardado con el que se cobró; `null` si fue con token. |
| refundedAmount | decimal | Suma de las devoluciones `succeeded`. |
| createdAt, updatedAt | timestamp | |
| refunds | lista de `Refund` | De la más antigua a la más reciente. Cada una: `id`, `refundRequestId`, `amount`, `reason`, `status` (`pending`, `succeeded`, `failed`), `createdAt`, `updatedAt`, `paymentId`. |

```json
{
  "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "chargeRequestId": "ch-001",
  "customerRef": "cus-1",
  "amount": 25.90,
  "currency": "EUR",
  "status": "authorized",
  "gatewayPaymentId": "gw_3MtwBwLkdIwHu7ix",
  "awaitingSince": null,
  "failureReason": null,
  "customerAction": null,
  "actionRequiredSince": null,
  "cancelOrigin": null,
  "paymentMethodId": null,
  "refundedAmount": 0,
  "createdAt": "2026-10-05T10:30:00.000Z",
  "updatedAt": "2026-10-05T10:30:00.412Z",
  "refunds": []
}
```

**Cómo leer el `status` de una respuesta.** Un cobro en estado de espera (`pending`, `capturing`,
`canceling`, `refunding`) quiere decir que la pasarela no contestó a tiempo. **No repitas la
acción**: el servicio pregunta él mismo a la pasarela (barrido cada 5 minutos, a partir de los 15
minutos de silencio) y publica el desenlace como evento. Mientras tanto, `getPayment` te da el
estado actual.

### requestCharge

| | |
|---|---|
| Endpoint | `POST /api/v1/payments` |
| Acceso | `service` — scope `payment:charge` |
| Idempotencia | sí — por `chargeRequestId` (clave de negocio, permanente). La misma petición devuelve el mismo cobro sin volver a llamar a la pasarela; dos idénticas a la vez convergen en uno. |
| Éxito | `201` con el cobro y `Location: /api/v1/payments/{id}`, también al repetir |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| chargeRequestId | string | requerido; tu clave de negocio |
| customerRef | string | requerido |
| amount | decimal | requerido; > 0, decimales según la moneda |
| currency | string | requerido; ISO 4217 vigente |
| paymentToken | string | token de un solo uso del componente de la pasarela (cliente presente) |
| paymentMethodId | uuid | medio guardado del titular (cliente ausente) |

Exactamente uno de `paymentToken` o `paymentMethodId`.

```json
{ "chargeRequestId": "ch-001", "customerRef": "cus-1", "amount": 25.90, "currency": "EUR",
  "paymentToken": "tok_1NirD82eZvKYlo2C" }
```

**Response.** El objeto `Payment` en el estado que alcanzó:
- `authorized`: retenido; captúralo o anúlalo.
- `actionRequired`: lleva al cliente al 3DS con `customerAction`.
- `failed`: con `failureReason`. **Un rechazo de la pasarela no es un error HTTP.**
- `pending`: la pasarela no contestó; espera el evento.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_SOURCE_INVALID | 400 | No se indica ninguna fuente de pago o se indican las dos. | Corregir input. |
| CURRENCY_UNKNOWN | 400 | `currency` no es un código vigente de ISO 4217. | Corregir input. |
| AMOUNT_SCALE_INVALID | 400 | `amount` tiene más decimales que los de la moneda. | Corregir input. |
| PAYMENT_METHOD_UNAVAILABLE | 422 | El medio no existe, está retirado o no es del titular (no se distingue cuál). | Corregir input; pedir otro medio o un token. |
| CHARGE_REQUEST_CONFLICT | 409 | Ya existe un cobro con ese `chargeRequestId` y otro contenido. | No reintentar; usa otra clave o consulta el cobro existente. |
| CONCURRENT_MODIFICATION | 409 | El cobro cambió mientras se procesaba la petición. | Consultar el cobro y decidir. |

Orden de evaluación:
1. La clave: una repetición devuelve el cobro existente aunque el medio se haya retirado después.
2. La fuente de pago.
3. La moneda.
4. Los decimales.
5. El medio.

### getPayment

| | |
|---|---|
| Endpoint | `GET /api/v1/payments/{id}` |
| Acceso | `service` — scope `payment:read` |
| Idempotencia | no aplica (query); sin caché, siempre el estado actual |

**Request** — path `id: uuid` (requerido). **Response** — `200` con el objeto `Payment`.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_NOT_FOUND | 404 | No existe un cobro con ese id. | No reintentar. |

### getPaymentByChargeRequest

| | |
|---|---|
| Endpoint | `GET /api/v1/payments/by-charge-request/{chargeRequestId}` |
| Acceso | `service` — scope `payment:read` |
| Idempotencia | no aplica (query) |

**Request** — path `chargeRequestId: string` (requerido). **Response** — `200` con el objeto
`Payment`. Útil para localizar un cobro por tu propia clave tras un timeout.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_NOT_FOUND | 404 | No existe un cobro con ese `chargeRequestId`. | No reintentar; el cobro no se registró, puedes pedirlo. |

### listPaymentsByCustomer

| | |
|---|---|
| Endpoint | `GET /api/v1/payments?customerRef=&page=&size=` |
| Acceso | `service` — scope `payment:read` |
| Idempotencia | no aplica (query) |

**Request** — query `customerRef: string` (requerido), `page` (base 0, default 0) y `size` (default
20, se recorta a 100).

**Response** — `200` con
`{ "items": [Payment…], "page": 0, "size": 20, "totalElements": 3, "totalPages": 1 }`, del más
reciente al más antiguo (`createdAt` desc, `id` desc). Un titular sin cobros devuelve
`items: []`, `totalElements: 0`, `totalPages: 0`. No declara errores propios.

### capturePayment

| | |
|---|---|
| Endpoint | `POST /api/v1/payments/{id}/capture` (sin cuerpo) |
| Acceso | `service` — scope `payment:charge` |
| Idempotencia | sí — por estado: si el cobro ya está en `capturing`, `captured`, `refunding` o `refunded`, devuelve `200` con el cobro tal cual, sin volver a llamar a la pasarela |
| Éxito | `200` con el cobro (`captured`, o `capturing` si la pasarela no contestó) |

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_NOT_FOUND | 404 | No existe un cobro con ese id. | No reintentar. |
| PAYMENT_NOT_CAPTURABLE | 409 | El cobro no está en `authorized` (ni ya capturándose o capturado). | No reintentar; consulta el estado. |
| AUTHORIZATION_EXPIRED | 409 | La pasarela ya había cancelado la autorización; el cobro queda `canceled`. | No reintentar; pide un cobro nuevo. |
| CAPTURE_REJECTED | 422 | La pasarela rechazó la captura; el cobro vuelve a `authorized`. | Se puede volver a intentar más tarde. |
| CONCURRENT_MODIFICATION | 409 | Otra petición tuya cambió el cobro a la vez. | Consultar el cobro y decidir. |

### cancelPayment

| | |
|---|---|
| Endpoint | `POST /api/v1/payments/{id}/cancel` (sin cuerpo) |
| Acceso | `service` — scope `payment:charge` |
| Idempotencia | sí — por estado: en `canceling` o `canceled` devuelve `200` con el cobro tal cual |
| Éxito | `200` con el cobro (`canceled`, `canceling` si la pasarela no contestó, o `failed` si la pasarela dice que el 3DS ya había fallado) |

Anula un cobro `authorized` o uno que espera el 3DS (`actionRequired`).

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_NOT_FOUND | 404 | No existe un cobro con ese id. | No reintentar. |
| PAYMENT_NOT_CANCELABLE | 409 | El cobro no está en `authorized` ni `actionRequired` (ni ya anulándose o anulado). | No reintentar; si está capturado, devuélvelo. |
| CANCEL_REJECTED | 422 | La pasarela rechazó la anulación; el cobro vuelve al estado del que salió. | Se puede volver a intentar más tarde. |
| CONCURRENT_MODIFICATION | 409 | Otra petición tuya cambió el cobro a la vez. | Consultar el cobro y decidir. |

### refundPayment

| | |
|---|---|
| Endpoint | `POST /api/v1/payments/{id}/refunds` |
| Acceso | `service` — scope `payment:refund` |
| Idempotencia | sí — por `refundRequestId` (permanente). La misma devolución, con el mismo `amount` o sin él, devuelve el cobro tal cual. Una devolución `failed` no se reintenta con la misma clave: usa otra. |
| Éxito | `200` con el cobro (`captured` si queda saldo, `refunded` si se devolvió todo, `refunding` si la pasarela no contestó) |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| refundRequestId | string | requerido; tu clave de la devolución |
| amount | decimal | opcional; si se omite, todo lo que queda por devolver |
| reason | string | opcional, ≤ 200; para tu rastro |

```json
{ "refundRequestId": "rf-001", "amount": 10.00, "reason": "Línea devuelta" }
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_NOT_FOUND | 404 | No existe un cobro con ese id. | No reintentar. |
| REFUND_REQUEST_CONFLICT | 409 | El `refundRequestId` ya existe con otro importe o sobre otro cobro. | No reintentar; usa otra clave. |
| PAYMENT_NOT_REFUNDABLE | 409 | El cobro no está en `captured` (incluido el que ya tiene una devolución en curso). | No reintentar; espera al desenlace de la devolución en curso. |
| AMOUNT_SCALE_INVALID | 400 | `amount` tiene más decimales que los de la moneda del cobro. | Corregir input. |
| REFUND_AMOUNT_EXCEEDED | 422 | `amount` supera lo que queda por devolver. | Corregir input. |
| REFUND_REJECTED | 422 | La pasarela rechazó la devolución; queda `failed` y el cobro vuelve a `captured`. | Reintentar con **otro** `refundRequestId`. |
| CONCURRENT_MODIFICATION | 409 | Otra petición cambió el cobro a la vez. | Consultar el cobro y decidir. |

### savePaymentMethod

| | |
|---|---|
| Endpoint | `POST /api/v1/payment-methods` |
| Acceso | `service` — scope `payment-method:write` |
| Idempotencia | sí — header `Idempotency-Key` **obligatorio**, recordado 24 h. La misma clave y el mismo cuerpo devuelven el medio ya guardado. |
| Éxito | `201` con el medio, sin `Location` (no hay lectura de un medio por id) |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| customerRef | string | requerido |
| paymentToken | string | requerido; token de un solo uso de la pasarela |
| makeDefault | boolean | opcional, default `false`; el primer medio del titular queda por defecto igualmente |

**Response — `PaymentMethod`**

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | Es lo que mandas como `paymentMethodId` al cobrar. |
| card | objeto | `brand`, `last4`, `expMonth`, `expYear`. Nunca el número. |
| isDefault | boolean | |
| status | enum | `active`, `removed` |
| expired | boolean | La tarjeta caducó (antes del mes en curso, UTC). |
| walletId | uuid | Wallet del titular. |

```json
{ "id": "3f2504e0-4f89-41d3-9a0c-0305e82c3301",
  "card": { "brand": "visa", "last4": "4242", "expMonth": 12, "expYear": 2030 },
  "isDefault": true, "status": "active", "expired": false,
  "walletId": "9b2f1e7a-1c2d-4e5f-8a9b-0c1d2e3f4a5b" }
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| IDEMPOTENCY_KEY_REQUIRED | 400 | Falta la cabecera `Idempotency-Key`. | Corregir input. |
| IDEMPOTENCY_KEY_REUSED | 409 | La misma clave con otro cuerpo. | No reintentar; usa otra clave. |
| IDEMPOTENCY_KEY_IN_PROGRESS | 409 | Otra petición con la misma clave está en curso. | Reintentar con la misma clave en unos segundos. |
| PAYMENT_METHOD_LIMIT_REACHED | 422 | El titular ya tiene 10 medios `active`. | No reintentar; retira uno antes. |
| PAYMENT_METHOD_AUTHENTICATION_REQUIRED | 422 | La pasarela exige autenticar al cliente (3DS) para guardarlo. | Completa la autenticación al tokenizar y pide el guardado con un token nuevo. |
| PAYMENT_METHOD_REJECTED | 422 | La pasarela no acepta el token (inválido, caducado o ya usado) o rechaza el medio. | Corregir input: token nuevo u otro medio. |
| GATEWAY_UNAVAILABLE | 503 | La pasarela no contestó; aquí no se guardó nada. | Reintentable, con un token nuevo y otra clave. |
| CONCURRENT_MODIFICATION | 409 | El Wallet cambió a la vez. | Reintentar. |

### listPaymentMethods

| | |
|---|---|
| Endpoint | `GET /api/v1/payment-methods?customerRef=` |
| Acceso | `service` — scope `payment-method:read` |
| Idempotencia | no aplica (query) |

**Response** — `200` con la lista de `PaymentMethod` `active` del titular, caducados incluidos
(`expired: true`), el preferente primero y después por `id`. Son como mucho 10, así que no se pagina.
Un titular sin medios devuelve `[]`. No declara errores propios.

### setDefaultPaymentMethod

| | |
|---|---|
| Endpoint | `POST /api/v1/payment-methods/{paymentMethodId}/default` |
| Acceso | `service` — scope `payment-method:write` |
| Idempotencia | sí — naturalmente idempotente: marcar el que ya es preferente no cambia nada |
| Éxito | `200` con el `PaymentMethod` |

**Request** — `{ "customerRef": "cus-1" }`: el medio tiene que ser de ese titular.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_METHOD_NOT_FOUND | 404 | El medio no existe, está retirado o no es del titular (no se distingue cuál). | No reintentar. |
| CONCURRENT_MODIFICATION | 409 | El Wallet cambió a la vez. | Reintentar. |

### removePaymentMethod

| | |
|---|---|
| Endpoint | `DELETE /api/v1/payment-methods/{paymentMethodId}?customerRef=` |
| Acceso | `service` — scope `payment-method:write` |
| Idempotencia | sí — retirar uno ya retirado responde `204` sin efecto |
| Éxito | `204` sin cuerpo |

Si era el preferente, el titular se queda sin medio por defecto. El medio no se vuelve a cobrar;
la pasarela lo sigue guardando, porque el diseño no tiene acción de desvincularlo allí.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PAYMENT_METHOD_NOT_FOUND | 404 | El medio no existe o no es del titular. | No reintentar. |
| CONCURRENT_MODIFICATION | 409 | El Wallet cambió a la vez. | Reintentar. |

## Eventos

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "PaymentAuthorized",
    "eventVersion": 1,
    "occurredAt": "2026-10-05T10:30:00.412Z",
    "source": "payment",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": { "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-001",
            "customerRef": "cus-1", "amount": 25.90, "currency": "EUR",
            "gatewayPaymentId": "gw_3MtwBwLkdIwHu7ix" }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`PaymentAuthorized`). Discriminador: el canal transporta varios tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| metadata.source | string | Servicio emisor: `payment`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición (un desenlace que aplica el barrido). |
| metadata.traceparent | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

Todos salen por el canal lógico **`paymentEvents`** (desenlaces de los cobros y de sus acciones de
seguimiento) y se publican por **outbox**: ningún evento se pierde si la transacción confirma. Un
mismo desenlace puede llegar por la respuesta síncrona, por el aviso de la pasarela o por el barrido,
pero se publica **una sola vez**.

### PaymentAuthorized

La pasarela autorizó el cobro: el importe está retenido y se puede capturar o anular.
Emitido por `markAuthorized`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| amount | decimal | requerido |
| currency | string | requerido |
| gatewayPaymentId | string | requerido |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-001",
  "customerRef": "cus-1", "amount": 25.90, "currency": "EUR", "gatewayPaymentId": "gw_3MtwBwLkdIwHu7ix" }
```

### PaymentActionRequired

El cobro necesita que el cliente complete una acción (3DS). `customerAction` es opaca y la consume el
componente de la pasarela en el navegador. Emitido por `markActionRequired`. Si el cliente no completa
el 3DS en 24 h, el servicio anula el cobro y publica `PaymentCanceled`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| amount | decimal | requerido |
| currency | string | requerido |
| customerAction | objeto | requerido; opaco, viaja como objeto JSON |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-010",
  "customerRef": "cus-1", "amount": 60.00, "currency": "EUR",
  "customerAction": { "type": "redirect", "url": "https://3ds.example/challenge/abc" } }
```

### PaymentCaptured

La pasarela capturó el cobro entero: está cobrado. Emitido por `markCaptured`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| amount | decimal | requerido |
| currency | string | requerido |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-001",
  "customerRef": "cus-1", "amount": 25.90, "currency": "EUR" }
```

### PaymentFailed

El cobro falló. `failureReason` usa el vocabulario neutro; `notReceived` significa que la pasarela
nunca conoció el cobro. Emitido por `markFailed`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| amount | decimal | requerido |
| currency | string | requerido |
| failureReason | enum | requerido; ver el objeto `Payment` |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-002",
  "customerRef": "cus-1", "amount": 40.00, "currency": "EUR", "failureReason": "insufficientFunds" }
```

### PaymentCanceled

La autorización quedó anulada y no se cobrará. Emitido por `markCanceled`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| amount | decimal | requerido |
| currency | string | requerido |
| requested | boolean | requerido; `true` si la pidió tu servidor o venció el plazo del 3DS, `false` si la pasarela la cerró por su cuenta (autorización expirada, 3DS caducado) |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-070",
  "customerRef": "cus-1", "amount": 20.00, "currency": "EUR", "requested": true }
```

### PaymentRefunded

La pasarela completó una devolución. Emitido por `markRefunded`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| refundRequestId | string | requerido |
| refundAmount | decimal | requerido; importe de esta devolución (el que informa la pasarela) |
| refundedAmount | decimal | requerido; total devuelto del cobro tras ella |
| currency | string | requerido |
| status | enum | requerido; `refunded` si se devolvió entero, `captured` si queda saldo |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-001",
  "customerRef": "cus-1", "refundRequestId": "rf-001", "refundAmount": 10.00,
  "refundedAmount": 10.00, "currency": "EUR", "status": "captured" }
```

### PaymentActionRejected

La pasarela rechazó (o nunca recibió) una captura, una anulación o una devolución, y el cobro volvió
al estado del que salió. Emitido por `capturePayment`, `cancelPayment`, `refundPayment` y
`sweepPendingPayments`.

| Campo | Tipo | Notas |
|---|---|---|
| paymentId | uuid | requerido |
| chargeRequestId | string | requerido |
| customerRef | string | requerido |
| action | enum | requerido; `capture`, `cancel` o `refund` |
| refundRequestId | string \| null | solo cuando `action` es `refund` |
| refundAmount | decimal \| null | solo cuando `action` es `refund` |
| status | enum | requerido; estado al que volvió el cobro |

```json
{ "paymentId": "7c9e6679-7425-40de-944b-e07fc1f90ae7", "chargeRequestId": "ch-091",
  "customerRef": "cus-1", "action": "refund", "refundRequestId": "rf-013", "refundAmount": 5.00,
  "status": "captured" }
```

### Suscripciones

Este servicio no se suscribe a ningún evento: los cobros se piden por HTTP. No hay puerta de entrada
asíncrona.
