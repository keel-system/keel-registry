# payment — Documento de diseño

> specs/payment v0.1.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`payment` cobra con tarjeta, a través de una pasarela intercambiable, los importes que le piden los
servidores de **una** aplicación: autoriza el cobro y después lo captura o lo anula, admite la
autenticación del cliente (3DS) y devuelve lo cobrado en una o varias veces. Guarda además medios de
pago por titular para cobrarle sin que esté presente.

Es infraestructura: no sabe qué es un pedido ni quién es un usuario. El consumidor le da una clave de
negocio por cobro (`chargeRequestId`) y una referencia opaca del titular (`customerRef`). El
navegador del comprador nunca le llama: produce el token con el componente de la pasarela y se lo
entrega a su propio backend, que es quien pide el cobro.

La pasarela no se nombra en el diseño: se elige al generar, como el broker. Quedan fuera a propósito
los métodos de pago asíncronos, las disputas, los cobros recurrentes programados, los payouts y la
captura parcial (ver § 7).

## 2. Modelo de dominio

**Value types**

| Tipo | Significado |
|---|---|
| `RequestReference` | Clave de negocio de una petición (cobro o devolución); `^[A-Za-z0-9._:-]+$`, 1–100. Es la guarda contra la doble ejecución. |
| `CustomerRef` | Referencia opaca del titular que asigna el consumidor (1–200). Solo se compara. |
| `Amount` | Importe en unidades mayores, positivo, hasta 3 decimales con `scalePolicy: reject`; cuántos admite de verdad lo fija la moneda (ISO 4217). |
| `CurrencyCode` | ISO 4217 en mayúsculas. |
| `FailureReason` | El vocabulario neutro de Keel para el fallo de un cobro: `declined`, `insufficientFunds`, `expiredCard`, `authenticationFailed`, `fraudSuspected`, `invalidPaymentMethod`, `notReceived`, `processingError`. |
| `CardDetails` | Resumen visible de una tarjeta: `brand`, `last4`, `expMonth`, `expYear`. Nunca el número. |

**Agregados**

- **`Payment`** (raíz) **+ `Refund`** (hija): el cobro y sus devoluciones cambian juntos, porque lo
  devuelto se valida contra lo cobrado en la misma transacción.
- **`Wallet`** (raíz, uno por `customerRef`) **+ `PaymentMethod`** (hija): el medio por defecto y el
  tope de 10 medios son invariantes **por titular**, y solo se sostienen si todos sus medios cambian
  en una transacción. Un cobro guarda el `paymentMethodId` como valor, no como relación a una
  entidad interna de otro agregado.

**`Payment`** — `chargeRequestId` (único), `customerRef`, `amount`, `currency`, `status`,
`gatewayPaymentId`, `awaitingSince`, `failureReason`, `customerAction` (json opaco),
`actionRequiredSince`, `cancelOrigin`, `paymentMethodId`, `refundedAmount` (**computed**: suma de las
devoluciones `succeeded`), `createdAt`/`updatedAt` (**generated**, en el contrato).

Ciclo de vida:

| Desde | Hacia |
|---|---|
| `pending` | `authorized`, `actionRequired`, `failed`, `canceled` |
| `actionRequired` | `authorized`, `failed`, `canceling`, `canceled` |
| `authorized` | `capturing`, `canceling`, `canceled` |
| `capturing` | `captured`, `authorized` (rechazo), `canceled` (autorización caducada) |
| `canceling` | `canceled`, `authorized` / `actionRequired` (rechazo, según `cancelOrigin`), `failed` |
| `captured` | `refunding` |
| `refunding` | `captured` (devolución parcial o rechazada), `refunded` (todo devuelto) |
| `refunded`, `canceled`, `failed` | terminales |

`pending`, `capturing`, `canceling` y `refunding` son **estados en vuelo**: la operación deja el
cobro en ellos *antes* de llamar a la pasarela. Así una respuesta perdida no se repite a ciegas, dos
acciones incompatibles no llegan las dos a la pasarela, y quien consulta ve que hay algo en curso.

**`Refund`** — `refundRequestId` (único), `amount`, `reason`, `status` (`pending`, `succeeded`,
`failed`), `createdAt`/`updatedAt`. Su estado lo gobierna la raíz: cambia en el mismo paso en que el
cobro entra o sale de `refunding`.

**`PaymentMethod`** — `gatewayMethodRef` (**sensitive**: nunca sale del servicio), `card`,
`isDefault`, `status` (`active` → `removed`, terminal), `expired` (**computed**: caducidad anterior
al mes en curso, UTC).

## 3. Invariantes y reglas clave

- `awaitingSince` solo tiene valor mientras el cobro espera un desenlace; `customerAction` y
  `actionRequiredSince`, solo en `actionRequired` (y durante la anulación de un cobro que venía de
  ahí); `failureReason`, solo en `failed`; `cancelOrigin`, solo en `canceling`.
- Un cobro con medio guardado solo usa un medio del Wallet de su mismo titular.
- Lo devuelto más lo que está en curso nunca supera el importe; un cobro está `refunded` si y solo si
  se devolvió entero. Como mucho una devolución en curso por cobro.
- Los decimales de cualquier importe no superan los de su moneda (JPY 0, EUR 2, KWD 3). Nunca se
  redondea.
- Un Wallet tiene como mucho 10 medios `active` y uno solo por defecto; un medio retirado no puede
  ser el preferente.
- **La clave se comprueba antes que cualquier otra guarda**: repetir un cobro o una devolución con
  el mismo contenido devuelve lo ya hecho sin volver a la pasarela; con otro contenido,
  `CHARGE_REQUEST_CONFLICT` / `REFUND_REQUEST_CONFLICT`. Dos peticiones idénticas a la vez convergen en
  un solo cobro.
- Un rechazo de la pasarela no es un error de `requestCharge`: el cobro se devuelve en `failed` con su
  motivo. En captura, anulación y devolución el rechazo sí responde error, pero el cambio de estado y
  el evento se confirman igual.
- Si el aviso de la pasarela aplica el desenlace antes que la respuesta síncrona, la operación relee
  y responde el cobro como esté: el consumidor no ve un `409` por una acción que sí se hizo.
- Ni el token ni la referencia de la pasarela de un medio se guardan o se exponen.

## 4. Qué hace

Toda la superficie es **servidor-a-servidor** (`defaultAudience: services`), bajo `/api/v1`.

**Cobros**

| Operación | Endpoint | Qué hace |
|---|---|---|
| `requestCharge` | `POST /payments` → `201` | Autoriza un cobro con un token (cliente presente) o con un medio guardado (cliente ausente). Responde el cobro en `authorized`, `actionRequired`, `failed` o `pending`. Idempotente por `chargeRequestId`. |
| `capturePayment` | `POST /payments/{id}/capture` | Captura el importe entero. Idempotente por estado. |
| `cancelPayment` | `POST /payments/{id}/cancel` | Anula un cobro autorizado o que espera el 3DS. Idempotente por estado. |
| `refundPayment` | `POST /payments/{id}/refunds` | Devuelve una parte o todo lo que queda. Idempotente por `refundRequestId`. |
| `getPayment` | `GET /payments/{id}` | El cobro con sus devoluciones. |
| `getPaymentByChargeRequest` | `GET /payments/by-charge-request/{chargeRequestId}` | Lo mismo, por la clave del consumidor. |
| `listPaymentsByCustomer` | `GET /payments?customerRef=` | Paginado 20/100, del más reciente al más antiguo. |

**Medios guardados**

| Operación | Endpoint | Qué hace |
|---|---|---|
| `savePaymentMethod` | `POST /payment-methods` → `201` | Guarda el medio de un token; el primero queda por defecto. Exige `Idempotency-Key` (24 h). |
| `listPaymentMethods` | `GET /payment-methods?customerRef=` | Los medios `active`, el preferente primero. Sin paginar (≤ 10). |
| `setDefaultPaymentMethod` | `POST /payment-methods/{id}/default` | Marca el preferente. |
| `removePaymentMethod` | `DELETE /payment-methods/{id}?customerRef=` → `204` | Lo retira; no se vuelve a cobrar. |

Toda operación sobre un medio recibe el `customerRef` y responde `PAYMENT_METHOD_NOT_FOUND` si el
medio no existe, está retirado o es de otro titular, sin distinguir cuál.

**Desenlaces** (`internal`, los aplica el servicio al conocerlos por la respuesta, el aviso o el
barrido): `markAuthorized`, `markActionRequired`, `markCaptured`, `markFailed`, `markCanceled`,
`markRefunded`. Si un desenlace llega repetido o tarde, no hace nada.

**Reconciliación** — `sweepPendingPayments`, cada 5 minutos: consulta a la pasarela, hasta 100 por
pasada y los más antiguos primero, los cobros que llevan más de 15 minutos esperando sin respuesta.
Aplica lo que diga la pasarela. Un cobro que la pasarela no conoce pasa a `failed` con
`notReceived`; una acción que nunca recibió vuelve al estado de origen. Anula los cobros que llevan
más de `actionRequiredTimeoutHours` (24 h) esperando el 3DS. Si la pasarela no contesta, sigue
preguntando sin techo, y cada cobro se procesa aislado.

## 5. Fronteras e integraciones

- **Pasarela de pago** (capa `payments`): `authorize-capture`, capacidades `partial-refund`,
  `customer-action` y `off-session`. Cuantas menos capacidades, más pasarelas sirven el diseño: por
  eso no se pide `partial-capture`. El contrato concreto, la firma de los avisos y la traducción de
  códigos los pone el generador.
- **Eventos** (`messaging`, canal `paymentEvents`, **outbox**): `PaymentAuthorized`,
  `PaymentActionRequired` (con `customerAction`), `PaymentCaptured`, `PaymentFailed`,
  `PaymentCanceled` (con `requested`), `PaymentRefunded` y `PaymentActionRejected`. Cada uno lleva el
  estado del hecho para que el consumidor decida sin volver a llamar. No se suscribe a nada.
- **Almacenamiento** (`persistence`): relacional; claves naturales `chargeRequestId`,
  `refundRequestId` y `customerRef` (Wallet); índice `[status, awaitingSince]` para el barrido.
  Frontera `per-operation`: cada acción escribe dos veces, el estado en vuelo antes de llamar y el
  desenlace después. Bloqueo optimista en `Payment` y `Wallet`. Auditoría: fechas en el contrato del
  cobro y autoría de cada fila fuera del contrato.
- **Seguridad**: OIDC con `client-credentials` y audiencia validada. Hay dos clientes máquina de
  ejemplo: `checkout` (cobra, captura, anula, gestiona medios y consulta) y `backoffice` (consulta y
  devuelve). Devolver dinero tiene scope propio.

## 6. Decisiones de diseño (qué / por qué)

Fuente: `specs/payment/decisions.yaml` (estructurales y aceptadas) y `specs/payment/gaps.yaml`.

**Encaje**

- **Solo servidores** frente a una superficie para la SPA del titular. El servicio es infraestructura
  de la aplicación; un público humano obligaría a operaciones de usuario, CORS y autorización por
  titular desde el token.
- **Autorizar y capturar** frente a cobrar en el acto. Sirve a más negocios (capturar al enviar y
  anular si no se sirve), a cambio de dos estados más y de una autorización que caduca sola.
- **Medios guardados en un registro propio** frente a pasar la referencia de la pasarela. Permite
  atar el medio a su titular sin que la referencia salga del servicio, y habilita cobrar sin cliente.
- **Una sola aplicación** frente a multi-aplicación: más simple. La clave de negocio es única global.
- **Devolución con scope propio** (`payment:refund`): el checkout puede cobrar sin poder devolver.

**Repetición (§3.2)**

- `requestCharge`: la **clave natural `chargeRequestId`** es la guarda, permanente, frente a una
  cabecera con TTL. Una carrera de dos peticiones idénticas relee y responde la repetición, en vez
  del `409 IDEMPOTENCY_KEY_IN_PROGRESS` canónico: el consumidor nunca ve un conflicto por su propio
  reintento.
- `capturePayment` / `cancelPayment`: **idempotentes por estado** frente a una cabecera o a un `409`
  en el reintento. La transición irrepetible impide la doble llamada y nada caduca.
- `refundPayment`: una entidad `Refund` con **`refundRequestId` único** frente a un importe
  acumulado, que devolvería dos veces al reintentar.
- `savePaymentMethod`: **`Idempotency-Key` obligatoria, 24 h**, porque el token es de un solo uso.

**Lo que puede llegar rancio (§3.3)** — **sin caché** en ninguna lectura. El estado del cobro cambia
por vías asíncronas y se consulta precisamente para saber si se resolvió.

**Paginación (§3.8)** — cobros del titular 20/100; medios sin paginar porque tienen tope.

**Fiabilidad (§3.1)** — **outbox** frente a best-effort. Son desenlaces de dinero: un despliegue del
broker en mal momento dejaría un pedido esperando pago con el cobro ya capturado, sin traza.

**Transacción y concurrencia (§3.7, §3.9)** — **por operación**, para que una operación futura que
toque dos agregados siga siendo atómica. **Bloqueo optimista**: un aviso y el barrido pueden traer
desenlaces distintos a la vez, y sin versión el último pisaría al primero.

**Auditoría (§3.9b)** — fechas del cobro **en el contrato** (ordenan el historial); autoría **fuera
del contrato**, para responder en una disputa qué cliente devolvió qué.

**Plazos y silencios**

- **El 3DS abandonado se anula a las 24 h** (parámetro), contado con su propio reloj
  (`actionRequiredSince`). No se cuenta con `awaitingSince`, que el barrido renueva en cada consulta.
- **Un cobro al que la pasarela no contesta nunca no se da por fallido**: el estado real lo tiene
  ella, y negarlo a ciegas podría negar un cobro que sí se hizo.
- **La caducidad de una autorización se conoce por el aviso.** Si se pierde, se descubre al intentar
  capturar (`AUTHORIZATION_EXPIRED`). Vigilar `authorized` exigiría un reloj que la capa no tiene.

**Contrato observable**

- Un **rechazo del cobro es `201` con el cobro `failed`**, no un `402`: es un hecho registrado, y el
  reintento devuelve lo mismo.
- **`customerAction` se expone** en la lectura y en el evento, aunque suele llevar un secreto del
  intento: el checkout lo necesita para recuperar un 3DS, y solo sirve para completar ese cobro con
  el cliente delante.
- **Importes como número JSON**, comparados por valor; claves que distinguen mayúsculas; nulos que
  viajan.
- **Moneda por petición** con escala ISO 4217, frente a una moneda fija por despliegue: un diseño de
  registry no debe fijar el mercado.
- **Tarjetas caducadas visibles** (`expired: true`) en vez de ocultas, para no tener un titular que
  ve 8 tarjetas y no puede guardar la novena.

## 7. Ficha de reutilización

### Contrato estable vs adaptable

**Estable** (cambiarlo rompe a un consumidor): los códigos de error, los siete eventos y su payload,
las rutas y status de la API, los scopes `payment:*` y `payment-method:*`, el vocabulario de
`FailureReason` (lo fija Keel) y el sobre de paginación.

**Adaptable sin romper a nadie**: el tope de medios por titular, el plazo del 3DS (parámetro), el
lote y la frecuencia del barrido, los tamaños de página, los clientes máquina de ejemplo (`checkout`,
`backoffice`, que el adoptante renombra), y las capacidades exigidas a la pasarela (quitar una
estrecha el diseño; añadir una es minor).

El spec versiona según `docs/methodology.md`: patch para prosa, minor para añadidos compatibles y
major para cualquier cambio de lo estable.

### Puntos de extensión típicos

- **Captura parcial**: añadir `partial-capture` y `capture.amount`.
- **Multi-aplicación**: `callerIdentity` desde el cliente máquina y la clave de negocio acotada por
  aplicación.
- **Cobro por evento** (off-session): la capa ya lo admite. Basta con una suscripción a
  `requestCharge`, con `payload-field` como idempotencia.
- **Patrones reutilizables** en otros servicios: los estados en vuelo con barrido que reclama, la
  clave natural como guarda permanente y el `cancelOrigin` para volver al estado de origen.

### Supuestos y limitaciones

- **Una sola aplicación.** Sin inquilinos: `checkout` y `backoffice` son credenciales de la misma
  aplicación y ven todos los cobros.
- **Consumidor de confianza.** El importe y el titular los pone el backend consumidor; el servicio no
  los contrasta con un pedido ni con un usuario, y no pone tope al importe.
- **Solo tarjeta, sin disputas.** Fuera: métodos asíncronos (PIX, boleto, transferencia),
  contracargos, cobros recurrentes programados, payouts y captura parcial. Las devoluciones hechas
  desde la consola de la pasarela no se reflejan.
- **Volumen moderado.** 100 cobros en duda por pasada cada 5 minutos y ≤ 10 medios por titular; para
  más volumen se ajusta el lote.
- **Límites heredados de la capa `payments` del DSL.** Retirar un medio no lo desvincula en la
  pasarela, y guardar un medio no admite 3DS (`PAYMENT_METHOD_AUTHENTICATION_REQUIRED`: el frontend
  autentica al tokenizar). Un timeout al guardar puede dejar el medio en la pasarela sin registro
  aquí; nunca se cobra.
- **Sin retención.** Un cobro es un registro contable y vive indefinidamente.

### Cómo reutilizarlo

`keel describe payment` da el resumen mecánico. Si sirve **tal cual**, se adopta con
`keel registry get payment`: llega con sus derivados al día y se va directo a `keel-<tech> build`. Si
hay que **cambiarlo**, se deriva con `keel new <nuevo> --from registry:payment`, y `/keel-design`
entrevista solo lo que cambia.

Lo esperable es **adoptarlo** cuando el caso es una aplicación con su checkout, y **derivarlo**
cuando hace falta multi-aplicación, cobro por evento o captura parcial.

## Cobertura de comportamiento

`specs/payment/validation-scenarios.md`: 31 flujos que cubren las 18 operaciones, todos los códigos
de error y todos los estados. Incluyen las carreras de idempotencia, el outbox con el canal caído y
con el relay rendido, el rescate del barrido, el clúster de dos réplicas y la autorización por scope.
