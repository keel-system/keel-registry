# Manual de uso por actor de payment

> specs/payment v0.2.3. Generado con `keel describe payment --manual`: no se edita, se cambia el diseño y se vuelve a generar.

## Actores

| Actor | Quién es | Operaciones |
|---|---|---|
| Cliente máquina checkout | Backend del checkout de la aplicación: cobra, captura o anula y gestiona los medios del titular. No devuelve dinero. | 10 |
| Cliente máquina backoffice | Backend de atención al cliente: consulta cobros y hace devoluciones. | 4 |

## Cliente máquina checkout

Backend del checkout de la aplicación: cobra, captura o anula y gestiona los medios del titular. No devuelve dinero.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [requestCharge](#requestcharge) | `POST /api/v1/payments` | Pide un cobro (autorización) con un token de un solo uso o con un medio guardado del titular. Registra el cobro en pending antes de llamar a la pasarela y responde con el cobro en el estado que haya alcanzado: authorized, actionRequired, failed, o pending si la pasarela no contestó. | — | — |
| [capturePayment](#capturepayment) | `POST /api/v1/payments/{id}/capture` | Captura entero el importe de un cobro autorizado. | Payment: authorized → capturing; Payment: capturing → authorized | — |
| [cancelPayment](#cancelpayment) | `POST /api/v1/payments/{id}/cancel` | Anula un cobro autorizado o que espera la autenticación del cliente, sin capturarlo. | Payment: authorized, actionRequired → canceling; Payment: canceling → authorized; Payment: canceling → actionRequired | — |
| [getPayment](#getpayment) | `GET /api/v1/payments/{id}` | Recupera un cobro por su id, con sus devoluciones. | — | — |
| [getPaymentByChargeRequest](#getpaymentbychargerequest) | `GET /api/v1/payments/by-charge-request/{chargeRequestId}` | Recupera un cobro por la clave de negocio con la que lo pidió el consumidor. | — | — |
| [listPaymentsByCustomer](#listpaymentsbycustomer) | `GET /api/v1/payments` | Lista los cobros de un titular, del más reciente al más antiguo. | — | — |
| [savePaymentMethod](#savepaymentmethod) | `POST /api/v1/payment-methods` | Guarda en el Wallet del titular el medio de pago que representa un token de la pasarela, para cobrarle después sin que esté presente. | — | — |
| [listPaymentMethods](#listpaymentmethods) | `GET /api/v1/payment-methods` | Lista los medios active de un titular, primero el que está por defecto. | — | — |
| [setDefaultPaymentMethod](#setdefaultpaymentmethod) | `POST /api/v1/payment-methods/{paymentMethodId}/default` | Marca un medio del titular como el preferente. | — | — |
| [removePaymentMethod](#removepaymentmethod) | `DELETE /api/v1/payment-methods/{paymentMethodId}` | Retira un medio guardado del titular; deja de poder cobrarse. | — | — |

## Cliente máquina backoffice

Backend de atención al cliente: consulta cobros y hace devoluciones.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [refundPayment](#refundpayment) | `POST /api/v1/payments/{id}/refunds` | Devuelve todo o parte de lo que queda por devolver de un cobro capturado. | Payment: captured → refunding; Payment: refunding → captured | — |
| [getPayment](#getpayment) | `GET /api/v1/payments/{id}` | Recupera un cobro por su id, con sus devoluciones. | — | — |
| [getPaymentByChargeRequest](#getpaymentbychargerequest) | `GET /api/v1/payments/by-charge-request/{chargeRequestId}` | Recupera un cobro por la clave de negocio con la que lo pidió el consumidor. | — | — |
| [listPaymentsByCustomer](#listpaymentsbycustomer) | `GET /api/v1/payments` | Lista los cobros de un titular, del más reciente al más antiguo. | — | — |

## Lo que hace el servicio sin que nadie lo pida

| Operación | La dispara | Cambia de estado |
|---|---|---|
| [markAuthorized](#markauthorized) | el desenlace authorized del cobro | Payment: pending, actionRequired → authorized |
| [markActionRequired](#markactionrequired) | el desenlace actionRequired del cobro | Payment: pending → actionRequired |
| [markCaptured](#markcaptured) | el desenlace captured del cobro | Payment: capturing → captured |
| [markFailed](#markfailed) | el desenlace failed del cobro | Payment: pending, actionRequired, canceling → failed |
| [markCanceled](#markcanceled) | el desenlace canceled del cobro | Payment: pending, actionRequired, authorized, capturing, canceling → canceled |
| [markRefunded](#markrefunded) | el desenlace refunded del cobro | Payment: refunding → captured; Payment: refunding → refunded |
| [sweepPendingPayments](#sweeppendingpayments) | reloj `*/5 * * * *` | Payment: capturing → authorized; Payment: canceling → authorized; Payment: canceling → actionRequired; Payment: refunding → captured; Payment: actionRequired → canceling |

## Operaciones

### requestCharge

Pide un cobro (autorización) con un token de un solo uso o con un medio guardado del titular. Registra el cobro en pending antes de llamar a la pasarela y responde con el cobro en el estado que haya alcanzado: authorized, actionRequired, failed, o pending si la pasarela no contestó.

- Tipo: command · `POST /api/v1/payments`
- Precondición: Se indica exactamente una fuente de pago: paymentToken o paymentMethodId.
- Precondición: Si se indica paymentMethodId, el medio existe, está active y pertenece al Wallet de customerRef.
- Regla: La clave se comprueba antes que cualquier otra guarda: si ya existe un cobro con ese chargeRequestId y los mismos customerRef, amount, currency y fuente de pago, se devuelve ese cobro en su estado actual sin volver a llamar a la pasarela, aunque el medio se haya retirado después; si existe con otro contenido, CHARGE_REQUEST_CONFLICT. La fuente se compara por su clase cuando es un token (el token no se guarda) y por paymentMethodId cuando es un medio guardado.
- Regla: Dos peticiones con el mismo chargeRequestId a la vez: la que choca con la clave natural al registrar el cobro relee el ganador y aplica la regla anterior — con el mismo contenido responde ese cobro (201); con otro, CHARGE_REQUEST_CONFLICT. Nunca hay un segundo cobro ni una segunda llamada a la pasarela.
- Regla: Responde 201 tanto al crear el cobro como al repetirlo, sea cual sea el estado en que esté.
- Regla: currency debe ser un código vigente de ISO 4217 (la tabla vigente del estándar), y amount no puede tener más decimales que los de esa moneda.
- Regla: El importe lo fija el servidor consumidor, que es de confianza; el servicio no le pone tope ni lo contrasta con nada.
- Regla: El cobro se registra en pending, con awaitingSince estampado, antes de llamar a la pasarela.
- Regla: La respuesta de la pasarela se aplica con su operación de desenlace (markAuthorized, markActionRequired o markFailed); si no llega, el cobro se queda en pending y lo resuelve el barrido.
- Regla: Un rechazo de la pasarela no es un error de la petición — el cobro se devuelve en failed con su failureReason (una moneda que la pasarela no admite falla con processingError).
- Regla: Si al aplicar el desenlace el cobro ya salió de pending por otra vía (el aviso de la pasarela), se relee y se responde el cobro tal como esté, sin error.
- Regla: El cobro en pending se confirma en su propia transacción antes de llamar a la pasarela, y el desenlace en otra: la frontera por operación vale para cada una de las dos escrituras.
- Regla: Si se cobra con un medio guardado, el cobro anota su paymentMethodId. Un medio retirado mientras se cobra no se detecta; vale tal como estaba al validar la petición.
- Regla: El paymentToken no se guarda; solo viaja a la pasarela.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_SOURCE_INVALID` | 400 | No se indica ninguna fuente de pago o se indican las dos. |
| `CURRENCY_UNKNOWN` | 400 | currency no es un código vigente de ISO 4217. |
| `AMOUNT_SCALE_INVALID` | 400 | amount tiene más decimales que los de la moneda. |
| `PAYMENT_METHOD_UNAVAILABLE` | 422 | El medio indicado no existe, está retirado o no es del titular (no se distingue cuál). |
| `CHARGE_REQUEST_CONFLICT` | 409 | Ya existe un cobro con ese chargeRequestId y con otro contenido. |

### capturePayment

Captura entero el importe de un cobro autorizado.

- Tipo: command · `POST /api/v1/payments/{id}/capture`
- Cambia de estado: Payment: authorized → capturing; Payment: capturing → authorized
- Regla: La repetición se comprueba antes que cualquier otra guarda: si el cobro ya está en capturing, captured, refunding o refunded, se devuelve tal cual (200) sin volver a llamar a la pasarela.
- Regla: El cobro pasa a capturing, con awaitingSince estampado, antes de llamar a la pasarela.
- Regla: Si la pasarela captura, se aplica markCaptured; si no contesta, el cobro se queda en capturing y lo resuelve el barrido.
- Regla: Si la pasarela rechaza porque la autorización ya caducó, se aplica markCanceled y responde AUTHORIZATION_EXPIRED.
- Regla: Si la pasarela rechaza por otro motivo, el cobro vuelve a authorized, se vacía awaitingSince, se publica PaymentActionRejected y responde CAPTURE_REJECTED.
- Regla: Si al aplicar el desenlace el cobro ya salió de su estado en vuelo por otra vía (el aviso de la pasarela o el barrido), se relee y se responde el cobro tal como esté, sin error; CONCURRENT_MODIFICATION queda para dos peticiones del consumidor que se cruzan.
- Regla: Un rechazo de la pasarela confirma el cambio de estado y el evento aunque la operación responda con error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese id. |
| `PAYMENT_NOT_CAPTURABLE` | 409 | El cobro no está en authorized ni en un estado de la repetición (capturing, captured, refunding, refunded). |
| `AUTHORIZATION_EXPIRED` | 409 | La pasarela ya había cancelado la autorización. |
| `CAPTURE_REJECTED` | 422 | La pasarela rechazó la captura por otro motivo. |
| `CONCURRENT_MODIFICATION` | 409 | El cobro cambió mientras se procesaba la petición. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en authorized, capturing. (lo pone el generador) |

### cancelPayment

Anula un cobro autorizado o que espera la autenticación del cliente, sin capturarlo.

- Tipo: command · `POST /api/v1/payments/{id}/cancel`
- Cambia de estado: Payment: authorized, actionRequired → canceling; Payment: canceling → authorized; Payment: canceling → actionRequired
- Regla: La repetición se comprueba antes que cualquier otra guarda: si el cobro ya está en canceling o canceled, se devuelve tal cual (200) sin volver a llamar a la pasarela.
- Regla: El cobro pasa a canceling antes de llamar a la pasarela: estampa awaitingSince, anota en cancelOrigin el estado del que sale y conserva customerAction y actionRequiredSince si venía de actionRequired.
- Regla: Si la pasarela anula, se aplica markCanceled; si no contesta, el cobro se queda en canceling y lo resuelve el barrido.
- Regla: Si la pasarela responde que el cobro ya terminó fallido (p. ej. el cliente falló la autenticación), se aplica markFailed con su motivo y la operación responde 200 con el cobro en failed.
- Regla: Si la pasarela rechaza la anulación, el cobro vuelve al estado que dice cancelOrigin (con awaitingSince renovado y el mismo actionRequiredSince de antes si vuelve a actionRequired, vacío si vuelve a authorized), se vacía cancelOrigin, se publica PaymentActionRejected y responde CANCEL_REJECTED.
- Regla: Si al aplicar el desenlace el cobro ya salió de su estado en vuelo por otra vía (el aviso de la pasarela o el barrido), se relee y se responde el cobro tal como esté, sin error; CONCURRENT_MODIFICATION queda para dos peticiones del consumidor que se cruzan.
- Regla: Un rechazo de la pasarela confirma el cambio de estado y el evento aunque la operación responda con error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese id. |
| `PAYMENT_NOT_CANCELABLE` | 409 | El cobro no está en authorized, actionRequired, canceling ni canceled. |
| `CANCEL_REJECTED` | 422 | La pasarela rechazó la anulación. |
| `CONCURRENT_MODIFICATION` | 409 | El cobro cambió mientras se procesaba la petición. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en authorized, actionRequired, canceling. (lo pone el generador) |

### refundPayment

Devuelve todo o parte de lo que queda por devolver de un cobro capturado.

- Tipo: command · `POST /api/v1/payments/{id}/refunds`
- Cambia de estado: Payment: captured → refunding; Payment: refunding → captured
- Regla: La clave se comprueba justo después de que el cobro exista y antes que cualquier otra guarda: si ya existe una devolución con ese refundRequestId sobre este cobro y la petición trae un amount igual a su requestedAmount o lo omite, se devuelve el cobro en su estado actual (200) sin volver a llamar a la pasarela, esté la devolución pending, succeeded o failed; si existe con otro amount o sobre otro cobro, REFUND_REQUEST_CONFLICT. La comparación es contra requestedAmount, no contra el amount que haya informado la pasarela al completarla: repetir la petición original tras un ajuste del importe no es un conflicto. Una devolución failed no se reintenta con la misma clave: hace falta otro refundRequestId.
- Regla: Dos devoluciones con el mismo refundRequestId a la vez: la que pierde, choque con la clave natural o con la versión del cobro al pasarlo a refunding, relee el cobro; si ya tiene una devolución con ese refundRequestId, aplica la regla anterior (el cobro tal cual, o REFUND_REQUEST_CONFLICT si trae un amount distinto del requestedAmount de la ganadora), y solo si no la tiene responde CONCURRENT_MODIFICATION. Nunca hay una segunda devolución ni una segunda llamada a la pasarela.
- Regla: amount no puede tener más decimales que los de la moneda del cobro.
- Regla: Se crea la devolución en pending, con amount y requestedAmount iguales al importe resuelto (el pedido o, si se omitió, lo que queda por devolver), y el cobro pasa a refunding, con awaitingSince estampado, antes de llamar a la pasarela.
- Regla: Si la pasarela devuelve, se aplica markRefunded; si no contesta, el cobro se queda en refunding y lo resuelve el barrido.
- Regla: Si la pasarela rechaza, la devolución pasa a failed, el cobro vuelve a captured con awaitingSince vacío, se publica PaymentActionRejected y responde REFUND_REJECTED.
- Regla: Si al aplicar el desenlace el cobro ya salió de su estado en vuelo por otra vía (el aviso de la pasarela o el barrido), se relee y se responde el cobro tal como esté, sin error; CONCURRENT_MODIFICATION queda para dos peticiones del consumidor que se cruzan.
- Regla: Un rechazo de la pasarela confirma el cambio de estado y el evento aunque la operación responda con error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese id. |
| `PAYMENT_NOT_REFUNDABLE` | 409 | El cobro no está en captured (incluido el que ya tiene una devolución en curso), salvo que ya exista una devolución con ese refundRequestId: esa petición se resuelve antes por la regla de la clave (el cobro tal cual o REFUND_REQUEST_CONFLICT). |
| `AMOUNT_SCALE_INVALID` | 400 | amount tiene más decimales que los de la moneda del cobro. |
| `REFUND_AMOUNT_EXCEEDED` | 422 | amount supera lo que queda por devolver. |
| `REFUND_REQUEST_CONFLICT` | 409 | Ya existe una devolución con ese refundRequestId con otro importe pedido (requestedAmount) o sobre otro cobro. |
| `REFUND_REJECTED` | 422 | La pasarela rechazó la devolución. |
| `CONCURRENT_MODIFICATION` | 409 | El cobro cambió mientras se procesaba la petición. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en captured, refunding. (lo pone el generador) |

### getPayment

Recupera un cobro por su id, con sus devoluciones.

- Tipo: query · `GET /api/v1/payments/{id}`
- Regla: Las devoluciones viajan dentro del cobro, de la más antigua a la más reciente (createdAt y, a igualdad, id).

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese id. |

### getPaymentByChargeRequest

Recupera un cobro por la clave de negocio con la que lo pidió el consumidor.

- Tipo: query · `GET /api/v1/payments/by-charge-request/{chargeRequestId}`
- Regla: Las devoluciones viajan dentro del cobro, de la más antigua a la más reciente (createdAt y, a igualdad, id).

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |

### listPaymentsByCustomer

Lista los cobros de un titular, del más reciente al más antiguo.

- Tipo: query · `GET /api/v1/payments`
- Regla: Un titular sin cobros devuelve una página vacía, no un error.

### markAuthorized

Registra que la pasarela autorizó el cobro.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: pending, actionRequired → authorized
- Regla: Guarda gatewayPaymentId y vacía awaitingSince, customerAction y actionRequiredSince.
- Regla: Si el cobro ya no está en pending ni actionRequired, no hace nada y no es un error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en pending, actionRequired. (lo pone el generador) |

### markActionRequired

Registra que el cobro necesita una acción del cliente (3DS) para completarse.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: pending → actionRequired
- Regla: Guarda gatewayPaymentId y customerAction, y estampa awaitingSince y actionRequiredSince.
- Regla: Si el cobro ya no está en pending, no hace nada y no es un error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en pending. (lo pone el generador) |

### markCaptured

Registra que la pasarela capturó el cobro.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: capturing → captured
- Regla: Vacía awaitingSince.
- Regla: Si el cobro ya no está en capturing, no hace nada y no es un error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en capturing. (lo pone el generador) |

### markFailed

Registra que el cobro falló, con el motivo en el vocabulario neutro.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: pending, actionRequired, canceling → failed
- Regla: Guarda failureReason y gatewayPaymentId si llega, y vacía awaitingSince, customerAction, actionRequiredSince y cancelOrigin.
- Regla: Si el cobro ya no está en pending, actionRequired ni canceling, no hace nada y no es un error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en pending, actionRequired, canceling. (lo pone el generador) |

### markCanceled

Registra que la autorización quedó anulada, a petición nuestra o porque la pasarela la expiró.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: pending, actionRequired, authorized, capturing, canceling → canceled
- Regla: Vacía awaitingSince, customerAction, actionRequiredSince y cancelOrigin.
- Regla: El evento lleva requested a true si el cobro venía de canceling (lo pidió el consumidor o venció el plazo del 3DS) y a false si lo cerró la pasarela por su cuenta.
- Regla: Si el cobro ya está en canceled o en un estado que no admite la transición, no hace nada y no es un error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en pending, actionRequired, authorized, capturing, canceling. (lo pone el generador) |

### markRefunded

Registra que la pasarela completó la devolución en curso del cobro.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: refunding → captured; Payment: refunding → refunded
- Regla: La devolución en pending del cobro pasa a succeeded con el importe que informa la pasarela (que manda si difiere del pedido), y se vacía awaitingSince.
- Regla: amount llega siempre como lo devuelto en esta devolución, nunca como el acumulado devuelto sobre el cobro: si la pasarela solo informa el acumulado, convertirlo es de quien traduce su respuesta o su aviso a este desenlace (el importe del propio objeto devolución de la pasarela), no de esta operación, que no recibe nada que le diga cuál de los dos es.
- Regla: Si el importe informado supera lo que quedaba por devolver (amount menos refundedAmount antes de esta devolución), la devolución se registra por lo que quedaba: lo devuelto nunca supera lo cobrado.
- Regla: El cobro pasa a refunded si refundedAmount llega a amount; si no, vuelve a captured.
- Regla: Si el cobro ya no está en refunding, no hace nada y no es un error.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_NOT_FOUND` | 404 | No existe un cobro con ese chargeRequestId. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en refunding. (lo pone el generador) |

### sweepPendingPayments

Consulta a la pasarela los cobros que llevan más de 15 minutos esperando un desenlace sin respuesta y aplica lo que diga.

- Tipo: command · no se llama por HTTP
- Cambia de estado: Payment: capturing → authorized; Payment: canceling → authorized; Payment: canceling → actionRequired; Payment: refunding → captured; Payment: actionRequired → canceling
- Regla: Selecciona los cobros en pending, actionRequired, capturing, canceling o refunding cuyo awaitingSince es anterior a hace 15 minutos, los más antiguos primero, hasta 100 por ejecución.
- Regla: Antes de consultar cada cobro renueva su awaitingSince; así otra réplica que corra a la vez no lo toma en la misma pasada.
- Regla: Tras una parada recupera todo lo pendiente, sin límite de antigüedad.
- Regla: Aplica el desenlace que diga la pasarela con su operación (markAuthorized, markActionRequired, markCaptured, markFailed, markCanceled, markRefunded).
- Regla: Un cobro en pending que la pasarela no conoce se da por fallido con failureReason notReceived.
- Regla: Si la pasarela dice que una acción en vuelo fue rechazada, o que nunca la recibió, el cobro vuelve al estado del que salió (cancelOrigin en una anulación; una devolución pasa a failed) y se publica PaymentActionRejected. Al salir de canceling se vacía cancelOrigin. Al volver a authorized o captured se vacía awaitingSince; al volver a actionRequired se renueva, y actionRequiredSince conserva el de antes.
- Regla: Un cobro en actionRequired cuyo actionRequiredSince es anterior a hace actionRequiredTimeoutHours, y que la pasarela sigue dando por pendiente del cliente, se anula como en cancelPayment: pasa a canceling con cancelOrigin actionRequired y se pide la anulación a la pasarela. El plazo se cuenta con actionRequiredSince, no con awaitingSince, que el barrido renueva en cada consulta.
- Regla: Cada cobro se procesa aislado: un fallo con uno no aborta la pasada ni deshace los demás.
- Regla: Si la pasarela no contesta, el cobro se queda como estaba y se vuelve a consultar en la siguiente pasada en que vuelva a superar el umbral, sin techo de intentos: un cobro cuyo estado real tiene la pasarela no se da por fallido a ciegas.

| Error | HTTP | Cuándo |
|---|---|---|
| `CONCURRENT_MODIFICATION` | 409 | Un desenlace se aplicó a la vez por otra vía; se descarta y lo resuelve la siguiente pasada. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en capturing, canceling, refunding, actionRequired. (lo pone el generador) |

### savePaymentMethod

Guarda en el Wallet del titular el medio de pago que representa un token de la pasarela, para cobrarle después sin que esté presente.

- Tipo: command · `POST /api/v1/payment-methods`
- Regla: El tope de 10 medios active se comprueba antes de llamar a la pasarela.
- Regla: Solo un guardado con éxito queda registrado bajo la Idempotency-Key. Si la operación termina en error (tope alcanzado, la pasarela rechaza, exige autenticar o no contesta), la clave queda libre: una petición posterior con la misma clave vuelve a ejecutarse, con el mismo paymentToken o con otro, y no recibe IDEMPOTENCY_KEY_IN_PROGRESS ni IDEMPOTENCY_KEY_REUSED.
- Regla: La repetición de un guardado registrado bajo la misma Idempotency-Key responde 201 con el medio tal como está ahora, aunque se haya retirado después (status removed, isDefault false): no guarda otro medio ni llama a la pasarela.
- Regla: Si el titular no tiene Wallet, se crea con este primer medio.
- Regla: El medio se guarda solo si la pasarela lo acepta; guarda su referencia opaca y el resumen de la tarjeta.
- Regla: Si la pasarela acepta el medio pero su resumen de tarjeta no trae brand, last4, expMonth o expYear (o alguno no cumple sus cotas), aquí no se guarda nada y se responde PAYMENT_METHOD_REJECTED: un medio sin ese resumen no se puede mostrar al titular ni saber si caducó. El medio puede quedar guardado en la pasarela sin registro aquí, uno por intento, como tras un timeout; nunca se cobra.
- Regla: El medio queda por defecto si makeDefault es true o si el titular no tenía ningún medio active; el anterior por defecto deja de serlo.
- Regla: El paymentToken no se guarda; solo viaja a la pasarela.
- Regla: Guardar dos veces la misma tarjeta crea dos medios: el servicio no reconoce tarjetas iguales. Un timeout de la pasarela puede dejar el medio guardado allí sin registro aquí; nunca se cobra.

| Error | HTTP | Cuándo |
|---|---|---|
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | La petición no trae la cabecera Idempotency-Key. |
| `PAYMENT_METHOD_LIMIT_REACHED` | 422 | El titular ya tiene 10 medios active. |
| `PAYMENT_METHOD_AUTHENTICATION_REQUIRED` | 422 | La pasarela exige autenticar al cliente (3DS) para guardar el medio; el frontend debe completarla al tokenizar y volver a pedir el guardado. |
| `PAYMENT_METHOD_REJECTED` | 422 | La pasarela no acepta el token (inválido, caducado o ya usado), rechaza el medio o lo acepta sin un resumen de tarjeta completo. Es un solo código a propósito: en los dos casos el consumidor hace lo mismo, pedir otra tarjeta; el resumen incompleto es una anomalía de la pasarela que reintentar con la misma tarjeta no arregla. |
| `GATEWAY_UNAVAILABLE` | 503 | La pasarela no contestó; aquí no se guardó nada y se puede reintentar con un token nuevo. |
| `CONCURRENT_MODIFICATION` | 409 | El Wallet cambió mientras se procesaba la petición. |

### listPaymentMethods

Lista los medios active de un titular, primero el que está por defecto.

- Tipo: query · `GET /api/v1/payment-methods`
- Regla: Solo devuelve medios active, caducados incluidos (con expired a true); como mucho son 10, así que no se pagina.
- Regla: A igualdad de isDefault, el orden es por id: es total.
- Regla: Un titular sin Wallet devuelve una lista vacía, no un error.

### setDefaultPaymentMethod

Marca un medio del titular como el preferente.

- Tipo: command · `POST /api/v1/payment-methods/{paymentMethodId}/default`
- Regla: El medio anterior por defecto deja de serlo; si el medio ya era el preferente, no cambia nada.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_METHOD_NOT_FOUND` | 404 | El medio no existe, está retirado o no es del titular (no se distingue cuál). |
| `CONCURRENT_MODIFICATION` | 409 | El Wallet cambió mientras se procesaba la petición. |

### removePaymentMethod

Retira un medio guardado del titular; deja de poder cobrarse.

- Tipo: command · `DELETE /api/v1/payment-methods/{paymentMethodId}`
- Regla: El medio pasa a removed y, si era el preferente, el titular se queda sin medio por defecto.
- Regla: Retirar un medio del titular que ya estaba removed no hace nada y no es un error.
- Regla: Los cobros ya hechos con el medio conservan su paymentMethodId.

| Error | HTTP | Cuándo |
|---|---|---|
| `PAYMENT_METHOD_NOT_FOUND` | 404 | El medio no existe o no es del titular (no se distingue cuál). |
| `CONCURRENT_MODIFICATION` | 409 | El Wallet cambió mientras se procesaba la petición. |
