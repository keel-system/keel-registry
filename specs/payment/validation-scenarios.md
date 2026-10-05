# payment — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/payment v0.1.0. Contrato de validación para la fase de generación.

> **Un único diseño, cualquier pasarela.** Estos escenarios no nombran ninguna pasarela y tienen que
> pasar igual con todas las que el generador ofrezca. Hablan de la **pasarela de prueba**: el doble
> que el generador levanta para la pasarela elegida, que habla su protocolo real y al que el arnés le
> dice qué contestar. Si un escenario necesitara saber cuál es, el hueco sería del diseño.

## Convenciones de determinación

- **Formato temporal**: instante en UTC ISO-8601 con milisegundos (`2026-10-05T10:30:00.000Z`).
  `createdAt`, `updatedAt`, `awaitingSince` y el `occurredAt` de los eventos se verifican **por
  forma** (y, cuando importa, por orden relativo), nunca por valor.
- **Identificadores generados**: `uuid` en forma canónica en minúsculas, verificados por forma y por
  reutilización simbólica dentro del flujo (`<p1>`, `<m1>`…): el id que devuelve un escenario es el
  que usa el siguiente. Cuando un orden desempata por id, compara la forma canónica en minúsculas
  como texto.
- **Claves de negocio**: `chargeRequestId`, `refundRequestId` y `customerRef` los elige el escenario
  (`ch-001`, `rf-001`, `cus-1`) y se comparan **tal cual**, distinguiendo mayúsculas (`compare:
  exact`, el defecto del DSL): `CH-001` y `ch-001` son claves distintas.
- **Ausencia vs nulo**: un campo sin valor **viaja como nulo**, en las respuestas y en los payloads de
  evento; nunca se omite (`conventions.nulls: include`). Una colección vacía viaja como `[]`.
- **Importes**: número JSON en unidades mayores de la moneda, comparados **por valor** (`12.5` y
  `12.50` son el mismo importe). Un importe con más decimales de los que admite su moneda según
  ISO 4217 se **rechaza** con `400 AMOUNT_SCALE_INVALID`, nunca se redondea (`scalePolicy: reject`
  sobre la escala 3 del tipo, y la regla por moneda de cada operación). Las pruebas cobran en `EUR`
  (2 decimales) salvo que el escenario diga `JPY` (0) o `KWD` (3).
- **La acción del cliente**: `customerAction` es **opaca**: su contenido lo define la pasarela. Los
  escenarios solo afirman si es nula o no, que viaja como **objeto JSON** (en la respuesta y en el
  evento, nunca como cadena con JSON escapado) y, cuando el escenario lo dice, que es **la misma**
  que se leyó antes.
- **Proyección de un cobro** (`output: { entity: Payment }`, derivada del artefacto). Toda respuesta
  que devuelve un cobro lleva **exactamente** estos campos, y ninguno más:
  `id`, `chargeRequestId`, `customerRef`, `amount`, `currency`, `status`, `gatewayPaymentId`,
  `awaitingSince`, `failureReason`, `customerAction`, `actionRequiredSince`, `cancelOrigin`,
  `paymentMethodId`,
  `refundedAmount`, `createdAt`, `updatedAt` y `refunds`. Cada elemento de `refunds` lleva
  exactamente `id`, `refundRequestId`, `amount`, `reason`, `status`, `createdAt`, `updatedAt` y
  `paymentId`, y la lista va de la devolución más antigua a la más reciente (`createdAt` y, a
  igualdad, `id`). Los escenarios dan los **valores** de esos campos; un campo que un escenario no
  menciona tiene el valor que le corresponde al estado según las invariantes de `Payment`:
  `awaitingSince` no nulo solo en `pending`, `actionRequired`, `capturing`, `canceling` y
  `refunding`; `customerAction` y `actionRequiredSince` no nulos solo en `actionRequired` (y en
  `canceling` con `cancelOrigin: "actionRequired"`); `failureReason` no nulo solo en `failed`; `cancelOrigin` no nulo
  solo en `canceling`; `refundedAmount: 0` sin devoluciones `succeeded`; `refunds: []` sin
  devoluciones.
- **Proyección de un medio** (`output: { entity: PaymentMethod, exclude: [gatewayMethodRef] }`).
  Lleva exactamente `id`, `card` (`{brand, last4, expMonth, expYear}`), `isDefault`, `status`,
  `expired` y `walletId`. `gatewayMethodRef` **no** viaja nunca.
- **Forma del cuerpo de error**: la del generador, `{timestamp, status, error, code, message,
  details}` más `correlationId`. Los escenarios fijan solo el `code` y el status HTTP.
- **Identidad**: cada operación se pide con la credencial del cliente que tiene su scope:
  `refundPayment` con la **credencial de máquina del cliente `backoffice`** (`payment:read`,
  `payment:refund`) y todas las demás —también las que preparan los `Given`— con la **credencial de
  máquina del cliente `checkout`** (`payment:charge`, `payment:read`, `payment-method:write`,
  `payment-method:read`), salvo que el escenario diga otra cosa. Un token emitido para otra audiencia
  responde `403` (lo fija el generador).
- **Cobros de los `Given`**: salvo que el `Given` diga otra cosa, todo cobro que prepara un `Given`
  es de `customerRef: "cus-1"`, de `amount: 10.00` y `currency: "EUR"`, se pide con un token nuevo y
  se lleva a su estado con las operaciones del propio diseño dentro del flujo (`requestCharge`,
  `capturePayment`, configurando la pasarela de prueba cuando hace falta).
- **Rutas**: todas bajo `/api/v1`.
- **Canal**: `paymentEvents` es por donde salen todos los desenlaces, con la envoltura Keel
  (`{metadata, data}`). Los escenarios afirman el `data` completo y `metadata.eventType`.
- **La pasarela de prueba**: autoriza, captura, anula, devuelve y guarda medios por defecto, y
  contesta en el acto. Un token `<tok-ok>` es válido y de un solo uso; cada escenario usa uno nuevo.
  Las tarjetas que guarda son `visa` acabada en `4242`, caducidad `12/2030`, salvo que el escenario
  diga otra. El arnés puede pedirle, antes del `When`, que **rechace** la siguiente operación (con un
  motivo), que **exija autenticación**, que **no conteste** (registrando o no lo que recibió), que
  **avise** de un desenlace de un cobro que ya conoce o que **dé otro importe** al completar una
  devolución. Un aviso de la pasarela va firmado como lo firma ella; el servidor nunca toma el
  desenlace de su contenido: se lo pregunta.
- **El barrido**: `sweepPendingPayments` corre cada 5 minutos. Los escenarios que dependen de él no
  esperan el umbral ni lo bajan: **envejecen el reloj de esa fila**: `awaitingSince` más allá de
  `unansweredAfterSeconds` (900 s), o `actionRequiredSince` más allá de `actionRequiredTimeoutHours`
  (24 h) para el
  plazo del 3DS, y esperan **un ciclo** (≤ 6 minutos).

## Matriz de cobertura

| Operación | Flujos | Superficie |
|-----------|--------|------------|
| requestCharge | FL-CHG-001, FL-CHG-002, FL-CHG-003, FL-CHG-004, FL-CHG-005, FL-CHG-006, FL-CHG-007, FL-OBX-001 | **servidores (M2M)** |
| capturePayment | FL-STL-001, FL-STL-002, FL-STL-007, FL-STL-009 | **servidores (M2M)** |
| cancelPayment | FL-STL-003, FL-STL-004, FL-STL-007 | **servidores (M2M)** |
| refundPayment | FL-STL-005, FL-STL-006 | **servidores (M2M)** |
| getPayment | FL-CHG-001, FL-CHG-003, FL-QRY-002 | **servidores (M2M)** |
| getPaymentByChargeRequest | FL-CHG-001, FL-QRY-002 | **servidores (M2M)** |
| listPaymentsByCustomer | FL-CHG-005, FL-QRY-001 | **servidores (M2M)** |
| savePaymentMethod | FL-MTH-001, FL-MTH-002, FL-MTH-003, FL-MTH-004 | **servidores (M2M)** |
| listPaymentMethods | FL-MTH-001, FL-MTH-002, FL-MTH-003 | **servidores (M2M)** |
| setDefaultPaymentMethod | FL-MTH-003 | **servidores (M2M)** |
| removePaymentMethod | FL-MTH-003, FL-CHG-004 | **servidores (M2M)** |
| markAuthorized | FL-CHG-001, FL-CHG-003, FL-REC-001 | interna; desenlace de la pasarela |
| markActionRequired | FL-CHG-003, FL-STL-004 | interna; desenlace de la pasarela |
| markCaptured | FL-STL-001, FL-STL-009, FL-REC-002 | interna; desenlace de la pasarela |
| markFailed | FL-CHG-002, FL-REC-001, FL-REC-003 | interna; desenlace de la pasarela |
| markCanceled | FL-STL-002, FL-STL-003, FL-STL-004, FL-STL-008, FL-REC-001, FL-REC-003 | interna; desenlace de la pasarela |
| markRefunded | FL-STL-005, FL-REC-002 | interna; desenlace de la pasarela |
| sweepPendingPayments | FL-REC-001, FL-REC-002, FL-REC-003, FL-REC-004, FL-REC-005, FL-CLU-001 | programada; efecto observable en el cobro y en `paymentEvents` |
| **outbox (canal indisponible y relay rendido)** | FL-OBX-001, FL-OBX-002 | los desenlaces |
| **autorización (401 / 403)** | FL-SEC-001 | todas las operaciones expuestas |

La misma matriz leída por **mecanismo**:

| Mecanismo | Camino feliz | Camino caro |
|---|---|---|
| Guarda del doble cargo (`naturalKey` sobre `chargeRequestId`) | FL-CHG-001 | FL-CHG-004 (repetición y conflicto) · FL-CHG-005 (a la vez) |
| Guarda de la doble devolución (`naturalKey` sobre `refundRequestId`) | FL-STL-005 | FL-STL-005 (repetición) · FL-STL-006 (clave de una devolución fallida) |
| Idempotencia de `savePaymentMethod` (`Idempotency-Key`) | FL-MTH-001 | FL-MTH-002 (a la vez) |
| Estados en vuelo | FL-STL-001 | FL-STL-007 (dos acciones a la vez) · FL-STL-009 (el aviso adelanta a la respuesta) · FL-REC-002 |
| Vuelta al origen de una anulación (`cancelOrigin`) | FL-STL-003 | FL-STL-004 (desde `actionRequired`) · FL-REC-002 |
| Aviso de la pasarela | FL-CHG-003 | FL-STL-008 (firma alterada) |
| Reconciliación | FL-REC-001 | FL-REC-004 (no tocar lo recién en vuelo) · FL-REC-005 (sin respuesta) · FL-CLU-001 (dos réplicas) |
| Plazo del 3DS | FL-REC-003 | FL-REC-003 (lo reciente no se anula) |
| Outbox | FL-CHG-001 | FL-OBX-001 (canal indisponible) · FL-OBX-002 (relay rendido) |
| Titularidad del medio | FL-CHG-007 | FL-CHG-006 · FL-MTH-003 (medio de otro titular) |

## Medios de pago

### FL-MTH-001: se guarda el primer medio de un titular, y su repetición

**Given**: ningún medio guardado de `cus-1`.

**When**: `savePaymentMethod` — `POST /api/v1/payment-methods` con cabecera `Idempotency-Key: k-m1`
```json
{ "customerRef": "cus-1", "paymentToken": "<tok-ok>" }
```

**Then**:
1. Status `201`, sin afirmar la cabecera `Location`: el alta no tiene lectura por id (decisión
   `CHK-API-CREATED-NO-READ`).
2. El cuerpo es exactamente `{ id: <m1>, card: { brand: "visa", last4: "4242", expMonth: 12,
   expYear: 2030 }, isDefault: true, status: "active", expired: false, walletId: <w1> }`: es el
   primer medio del titular, así que queda por defecto aunque `makeDefault` no viniera. No aparece
   `gatewayMethodRef`.
3. `listPaymentMethods` — `GET /api/v1/payment-methods?customerRef=cus-1` responde `200` con una
   lista de un elemento, igual al cuerpo del punto 2.

**When**: se repite la misma petición con la misma `Idempotency-Key: k-m1` y el mismo cuerpo.

**Then**:
4. Status `201` con el **mismo** cuerpo del punto 2 (mismo `id <m1>`).
5. La pasarela de prueba ha recibido **un** solo guardado, y `listPaymentMethods` sigue devolviendo
   un único medio.

**When**: se repite con la misma `Idempotency-Key: k-m1` y otro cuerpo (`makeDefault: true`).

**Then**:
6. Status `409` con `IDEMPOTENCY_KEY_REUSED` (canónico) y la pasarela no recibe nada.

**When**: `savePaymentMethod` sin cabecera `Idempotency-Key`.

**Then**:
7. Status `400` con `IDEMPOTENCY_KEY_REQUIRED`, y la pasarela no recibe nada.

**Casos borde**:
- `customerRef` vacío o sin `paymentToken` → `400` (validación de forma).

### FL-MTH-002: el mismo guardado a la vez

**Given**: ningún medio guardado de `cus-2`.

**When**: a la vez, dos `savePaymentMethod` con la misma `Idempotency-Key: k-m2` y el mismo cuerpo
`{ "customerRef": "cus-2", "paymentToken": "<tok-ok>" }`.

**Then**:
1. O las dos responden `201` con el mismo cuerpo (el mismo `id`), o una responde `201` y la otra
   `409` con `IDEMPOTENCY_KEY_IN_PROGRESS`.
2. Sea cual sea la ganadora, `listPaymentMethods` de `cus-2` devuelve **exactamente un** medio, y
   la pasarela de prueba ha recibido **un** guardado.

### FL-MTH-003: varios medios, el preferente y la retirada

**Given**: `cus-1` con el medio `<m1>` por defecto (como en FL-MTH-001).

**When**: `savePaymentMethod` con `Idempotency-Key: k-m3` y `{ "customerRef": "cus-1",
"paymentToken": "<tok-ok>", "makeDefault": true }`, con la pasarela de prueba guardando una tarjeta
`mastercard` acabada en `5454` y caducidad `01/2020`.

**Then**:
1. Status `201` con `{ id: <m2>, card: { brand: "mastercard", last4: "5454", expMonth: 1,
   expYear: 2020 }, isDefault: true, status: "active", expired: true, walletId: <w1> }`: la tarjeta
   está caducada y se guarda igual, marcada.
2. `listPaymentMethods` de `cus-1` responde `[<m2> (isDefault true), <m1> (isDefault false)]`: el
   preferente primero; `<m1>` dejó de serlo.

**When**: `setDefaultPaymentMethod` — `POST /api/v1/payment-methods/<m1>/default` con
`{ "customerRef": "cus-1" }`, dos veces seguidas.

**Then**:
3. Las dos responden `200` con el medio `<m1>` y `isDefault: true`: repetirlo no cambia nada.
4. `listPaymentMethods` responde `[<m1> (isDefault true), <m2> (isDefault false)]`.

**When**: `removePaymentMethod` — `DELETE /api/v1/payment-methods/<m1>?customerRef=cus-1`, dos veces.

**Then**:
5. Las dos responden `204` sin cuerpo: retirar un medio ya retirado no hace nada.
6. `listPaymentMethods` devuelve solo `[<m2> (isDefault false)]`: el titular se queda sin medio por
   defecto, y no se elige otro solo.

**When**: con `<m1>` retirado:
- `setDefaultPaymentMethod` sobre `<m1>` con `customerRef: "cus-1"`;
- `setDefaultPaymentMethod` sobre `<m2>` con `customerRef: "cus-9"`;
- `removePaymentMethod` sobre `<m2>` con `customerRef=cus-9`;
- `setDefaultPaymentMethod` sobre un id que no existe.

**Then**:
7. Las cuatro responden `404` con `PAYMENT_METHOD_NOT_FOUND`: no se distingue si el medio no
   existe, está retirado o es de otro titular.
8. `listPaymentMethods` de `cus-1` sigue devolviendo solo `<m2>` con `isDefault: false`.

**Notas de determinación**: a igualdad de `isDefault`, el orden es por `id` ascendente.
`listPaymentMethods` de un titular sin medios (`cus-9`) responde `200` con `[]`.

### FL-MTH-004: lo que no se puede guardar

**Given**: `cus-3` con 10 medios `active` guardados (cada uno con su propia `Idempotency-Key`).

**When**: `savePaymentMethod` para `cus-3`, con una `Idempotency-Key` nueva (`k-m4a`) y la pasarela de
prueba configurada para rechazar el medio.

**Then**:
1. Status `422` con `PAYMENT_METHOD_LIMIT_REACHED`, y la pasarela **no** recibe nada: el tope se
   comprueba antes de llamarla, así que gana a su rechazo.

**When**: `savePaymentMethod` para `cus-4` (sin medios) en cada caso, cada uno con su clave:
- la pasarela de prueba rechaza el medio;
- la pasarela de prueba exige autenticar al cliente para guardarlo;
- la pasarela de prueba no contesta.

**Then**:
2. Rechazo: status `422` con `PAYMENT_METHOD_REJECTED`.
3. Autenticación: status `422` con `PAYMENT_METHOD_AUTHENTICATION_REQUIRED`.
4. Sin respuesta: status `503` con `GATEWAY_UNAVAILABLE`.
5. `listPaymentMethods` de `cus-4` responde `[]` en los tres casos.

**Orden de evaluación** (`savePaymentMethod`):
1. Cabecera `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Repetición de una clave ya vista: mismo cuerpo → la respuesta original; otro cuerpo →
   `IDEMPOTENCY_KEY_REUSED` (`409`); en curso → `IDEMPOTENCY_KEY_IN_PROGRESS` (`409`).
3. Tope de 10 medios `active` → `PAYMENT_METHOD_LIMIT_REACHED` (`422`).
4. La pasarela → `PAYMENT_METHOD_AUTHENTICATION_REQUIRED` (`422`), `PAYMENT_METHOD_REJECTED` (`422`)
   o `GATEWAY_UNAVAILABLE` (`503`).
5. Escritura del Wallet → `CONCURRENT_MODIFICATION` (`409`) si otra escritura lo cambió a la vez.

## Cobros

### FL-CHG-001: un cobro con el cliente presente se autoriza

**Given**: ningún cobro de `cus-1`.

**When**: `requestCharge` — `POST /api/v1/payments`
```json
{ "chargeRequestId": "ch-001", "customerRef": "cus-1", "amount": 25.90, "currency": "EUR",
  "paymentToken": "<tok-ok>" }
```

**Then**:
1. Status `201` con la cabecera `Location: /api/v1/payments/<p1>`.
2. El cuerpo es el cobro `<p1>` con `chargeRequestId: "ch-001"`, `customerRef: "cus-1"`,
   `amount: 25.90`, `currency: "EUR"`, `status: "authorized"`, `gatewayPaymentId` no nulo,
   `awaitingSince: null`, `failureReason: null`, `customerAction: null`, `cancelOrigin: null`,
   `paymentMethodId: null`, `refundedAmount: 0`, `refunds: []`, y `createdAt`/`updatedAt` con forma
   de instante. El token no aparece en ningún campo.
3. `getPayment` — `GET /api/v1/payments/<p1>` responde `200` con el mismo cuerpo.
4. `getPaymentByChargeRequest` — `GET /api/v1/payments/by-charge-request/ch-001` responde `200` con
   el mismo cuerpo.
5. `paymentEvents` recibe **exactamente un** `PaymentAuthorized` con `data` `{ paymentId: <p1>,
   chargeRequestId: "ch-001", customerRef: "cus-1", amount: 25.90, currency: "EUR",
   gatewayPaymentId: <el del punto 2> }`.

**Notas de determinación**: el cobro nace en `pending` y la respuesta llega ya con el desenlace
porque la pasarela de prueba contesta en el acto; FL-REC-001 es el caso en que no.

### FL-CHG-002: la pasarela rechaza el cobro

**Given**: la pasarela de prueba configurada para rechazar el siguiente cobro por fondos
insuficientes.

**When**: `requestCharge` con `{ "chargeRequestId": "ch-002", "customerRef": "cus-1",
"amount": 40.00, "currency": "EUR", "paymentToken": "<tok-ok>" }`.

**Then**:
1. Status `201` con el cobro en `status: "failed"`, `failureReason: "insufficientFunds"`,
   `gatewayPaymentId` no nulo y `awaitingSince: null`: el rechazo no es un error de la petición, es
   el desenlace del cobro.
2. `paymentEvents` recibe **exactamente un** `PaymentFailed` con `data` `{ paymentId: <p2>,
   chargeRequestId: "ch-002", customerRef: "cus-1", amount: 40.00, currency: "EUR",
   failureReason: "insufficientFunds" }`.
3. `capturePayment` sobre `<p2>` responde `409` con `PAYMENT_NOT_CAPTURABLE`: `failed` es terminal.

**When**: `requestCharge` con `{ "chargeRequestId": "ch-003", "customerRef": "cus-1", "amount": 5000,
"currency": "XAF", "paymentToken": "<tok-ok>" }` (la moneda no tiene decimales), con la pasarela de
prueba configurada para no admitir esa moneda.

**Then**:
4. Status `201` con `status: "failed"` y `failureReason: "processingError"`.

### FL-CHG-003: la pasarela pide autenticar al cliente

**Given**: la pasarela de prueba configurada para exigir autenticación en el siguiente cobro.

**When**: `requestCharge` con `chargeRequestId: "ch-010"`, `customerRef: "cus-1"`, `amount: 60.00`,
`currency: "EUR"` y un token.

**Then**:
1. Status `201` con el cobro `<p10>` en `status: "actionRequired"`, `customerAction` no nula y como
   objeto, `awaitingSince` y `actionRequiredSince` no nulos, `gatewayPaymentId` no nulo y
   `cancelOrigin: null`.
2. `getPayment` sobre `<p10>` responde lo mismo, con la misma `customerAction`.
3. `paymentEvents` recibe **exactamente un** `PaymentActionRequired` con `data` `{ paymentId: <p10>,
   chargeRequestId: "ch-010", customerRef: "cus-1", amount: 60.00, currency: "EUR",
   customerAction: <la misma, como objeto> }`.

**When**: la pasarela de prueba avisa de que el cliente se autenticó y el cobro quedó autorizado.

**Then**:
4. En ≤ 10 s `getPayment` sobre `<p10>` responde `status: "authorized"`, `customerAction: null`,
   `actionRequiredSince: null` y `awaitingSince: null`.
5. `paymentEvents` recibe **exactamente un** `PaymentAuthorized` para `ch-010`.

**When**: la pasarela de prueba vuelve a avisar del mismo desenlace.

**Then**:
6. `<p10>` sigue en `authorized` y `paymentEvents` no recibe ningún `PaymentAuthorized` más: un
   desenlace repetido no hace nada.

### FL-CHG-004: la misma petición de cobro otra vez

**Given**: `cus-1` con un medio guardado `<m1>` (guardado en este flujo como en FL-MTH-001) y el cobro
`ch-020` hecho con él: `requestCharge` con `{ "chargeRequestId": "ch-020", "customerRef": "cus-1",
"amount": 15.00, "currency": "EUR", "paymentMethodId": "<m1>" }` respondió `201` con el cobro `<p20>`
en `authorized` y `paymentMethodId: <m1>`.

**When**: se repite exactamente la misma petición.

**Then**:
1. Status `201` con `Location: /api/v1/payments/<p20>` y el cobro `<p20>` tal como está
   (`authorized`): la repetición no vuelve a llamar a la pasarela, que sigue teniendo **un** cobro
   para `ch-020`.
2. `paymentEvents` no recibe ningún desenlace más para `ch-020`.

**When**: se retira `<m1>` (`removePaymentMethod`, `204`) y se repite otra vez la misma petición.

**Then**:
3. Status `201` con el cobro `<p20>` tal como está: la repetición se comprueba antes que la guarda
   del medio, así que el reintento no recibe `PAYMENT_METHOD_UNAVAILABLE`.

**When**: `requestCharge` con `chargeRequestId: "ch-020"` y `amount: 16.00` (el resto igual).

**Then**:
4. Status `409` con `CHARGE_REQUEST_CONFLICT`, y la pasarela no recibe nada.

**When**: `requestCharge` con `{ "chargeRequestId": "ch-021", "customerRef": "cus-1", "amount": 10.00,
"currency": "EUR", "paymentToken": "<tok-ok>" }`, y se repite con el mismo cuerpo salvo **otro**
`paymentToken`.

**Then**:
5. Las dos responden `201` con el mismo cobro: con token, la fuente se compara por su clase (el
   token no se guarda).

**When**: `getPaymentByChargeRequest` sobre `CH-020`.

**Then**:
6. Status `404` con `PAYMENT_NOT_FOUND`: las claves distinguen mayúsculas.

### FL-CHG-005: la misma petición de cobro a la vez

**Given**: ningún cobro de `cus-5`.

**When**: a la vez, dos `requestCharge` idénticos con `{ "chargeRequestId": "ch-030",
"customerRef": "cus-5", "amount": 9.99, "currency": "EUR", "paymentToken": "<tok-ok>" }`.

**Then**:
1. Las dos responden `201` con el mismo cobro (el mismo `id`): la que choca con la clave natural
   relee el ganador y responde como una repetición. Ninguna recibe un `409`.
2. Sea cual sea la ganadora, `listPaymentsByCustomer` de `cus-5` responde `totalElements: 1`, y la
   pasarela de prueba tiene **un** cobro para `ch-030`.
3. `paymentEvents` recibe **exactamente un** desenlace para `ch-030`.

### FL-CHG-006: las peticiones de cobro que se rechazan

**Given**: `cus-1` con un medio guardado `<m1>` y otro `<m9>` ya retirado.

**When**: `requestCharge` en cada uno de estos casos, cada uno con su propio `chargeRequestId`
(`ch-040` … `ch-049`) y, salvo lo que cambia, `customerRef: "cus-1"`, `amount: 10.00`,
`currency: "EUR"` y un token:
- sin `paymentToken` ni `paymentMethodId`;
- con los dos a la vez;
- con `currency: "ZZZ"`;
- con `amount: 10.555`;
- con `currency: "JPY"` y `amount: 10.5`;
- con `paymentMethodId: <m1>` y `customerRef: "cus-2"`;
- con `paymentMethodId: <m9>`;
- con un `paymentMethodId` que no existe;
- con `currency: "KWD"` y `amount: 1.234`;
- sin `paymentToken` ni `paymentMethodId` **y** con `currency: "ZZZ"`.

**Then**:
1. Los dos primeros: status `400` con `PAYMENT_SOURCE_INVALID`.
2. `ZZZ`: status `400` con `CURRENCY_UNKNOWN`.
3. `10.555` en EUR y `10.5` en JPY: status `400` con `AMOUNT_SCALE_INVALID`.
4. El medio de otro titular, el retirado y el inexistente: status `422` con
   `PAYMENT_METHOD_UNAVAILABLE`, el mismo code y status en los tres: no se revela cuál es.
5. `1.234` en KWD: status `201`: tres decimales son válidos en esa moneda.
6. El último: status `400` con `PAYMENT_SOURCE_INVALID`: la fuente se comprueba antes que la moneda.
7. Salvo el de KWD, la pasarela de prueba no ha recibido ningún cobro, y `getPaymentByChargeRequest`
   sobre esas claves responde `404` con `PAYMENT_NOT_FOUND`.

**Orden de evaluación** (`requestCharge`):
1. La clave: mismo `chargeRequestId` y mismo contenido → el cobro existente (`201`); otro
   contenido → `CHARGE_REQUEST_CONFLICT` (`409`). La carrera con la clave se resuelve igual al
   registrar el cobro.
2. Exactamente una fuente de pago → `PAYMENT_SOURCE_INVALID` (`400`).
3. Moneda vigente en ISO 4217 → `CURRENCY_UNKNOWN` (`400`).
4. Decimales del importe según la moneda → `AMOUNT_SCALE_INVALID` (`400`).
5. Medio existente, `active` y del titular → `PAYMENT_METHOD_UNAVAILABLE` (`422`).
6. Registro en `pending` y llamada a la pasarela: su rechazo es un `201` con el cobro en `failed`.

**Casos borde**:
- `chargeRequestId` con espacios o de más de 100 caracteres, `amount: 0` o negativo, o sin
  `customerRef` → `400` (validación de forma).

### FL-CHG-007: un cobro con un medio guardado

**Given**: `cus-1` con un medio guardado `<m1>`.

**When**: `requestCharge` con `{ "chargeRequestId": "ch-050", "customerRef": "cus-1",
"amount": 30.00, "currency": "EUR", "paymentMethodId": "<m1>" }`.

**Then**:
1. Status `201` con el cobro en `authorized` y `paymentMethodId: <m1>`.
2. `paymentEvents` recibe **exactamente un** `PaymentAuthorized` para `ch-050`.

## Liquidación

### FL-STL-001: se captura un cobro autorizado, y la captura repetida

**Given**: el cobro `<p1>` (`ch-001`, 25.90 EUR) autorizado, creado en este flujo como en FL-CHG-001.

**When**: `capturePayment` — `POST /api/v1/payments/<p1>/capture`, sin cuerpo.

**Then**:
1. Status `200` con el cobro en `status: "captured"`, `awaitingSince: null` y `refundedAmount: 0`:
   pasó por `capturing` y la pasarela confirmó en el acto.
2. `paymentEvents` recibe **exactamente un** `PaymentCaptured` con `data` `{ paymentId: <p1>,
   chargeRequestId: "ch-001", customerRef: "cus-1", amount: 25.90, currency: "EUR" }`.

**When**: se repite `capturePayment` sobre `<p1>`.

**Then**:
3. Status `200` con el cobro tal cual (`captured`): la pasarela de prueba ha recibido **una** sola
   captura y `paymentEvents` no recibe otro `PaymentCaptured`.

### FL-STL-002: lo que no se puede capturar

**Given**: un cobro `ch-060` en `pending` (la pasarela de prueba no contestó al cobro), un cobro
`ch-061` autorizado cuya autorización la pasarela de prueba da por caducada, y un cobro `ch-062`
autorizado cuya captura la pasarela de prueba rechaza.

**When**: `capturePayment` sobre `ch-060`, sobre un id que no existe, sobre `ch-061` y sobre `ch-062`.

**Then**:
1. `ch-060`: status `409` con `PAYMENT_NOT_CAPTURABLE`: `pending` no admite captura.
2. El inexistente: status `404` con `PAYMENT_NOT_FOUND`.
3. `ch-061`: status `409` con `AUTHORIZATION_EXPIRED`, y `getPayment` lo da en `canceled` con
   `awaitingSince: null`. `paymentEvents` recibe **exactamente un** `PaymentCanceled` para `ch-061`
   con `requested: false`: lo cerró la pasarela, no el consumidor.
4. `ch-062`: status `422` con `CAPTURE_REJECTED`, y `getPayment` lo vuelve a dar en `authorized` con
   `awaitingSince: null`. `paymentEvents` recibe **exactamente un** `PaymentActionRejected` con
   `data` `{ paymentId, chargeRequestId: "ch-062", customerRef: "cus-1", action: "capture",
   refundRequestId: null, refundAmount: null, status: "authorized" }`.
5. Se repite `capturePayment` sobre `ch-061`: status `409` con `PAYMENT_NOT_CAPTURABLE`, porque ya
   está `canceled`.

**Orden de evaluación** (`capturePayment`):
1. El cobro existe → `PAYMENT_NOT_FOUND` (`404`).
2. Repetición: en `capturing`, `captured`, `refunding` o `refunded` → el cobro tal cual (`200`).
3. Está en `authorized` → `PAYMENT_NOT_CAPTURABLE` (`409`).
4. Paso a `capturing` → `CONCURRENT_MODIFICATION` (`409`) si otra petición del consumidor lo cambió.
5. La pasarela: caducada → `AUTHORIZATION_EXPIRED` (`409`); otro rechazo → `CAPTURE_REJECTED`
   (`422`); el cambio de estado y el evento se confirman igual.

### FL-STL-003: se anula una autorización

**Given**: un cobro `ch-070` (20.00 EUR) autorizado, un cobro `ch-072` capturado, y otro `ch-071`
autorizado cuya anulación la
pasarela de prueba rechaza.

**When**: `cancelPayment` — `POST /api/v1/payments/<p70>/cancel`, dos veces.

**Then**:
1. La primera: status `200` con el cobro en `status: "canceled"`, `awaitingSince: null` y
   `cancelOrigin: null`.
2. `paymentEvents` recibe **exactamente un** `PaymentCanceled` con `data` `{ paymentId: <p70>,
   chargeRequestId: "ch-070", customerRef: "cus-1", amount: 20.00, currency: "EUR",
   requested: true }`.
3. La segunda: status `200` con el cobro tal cual: la pasarela recibió **una** anulación.

**When**: `cancelPayment` sobre `ch-071`, y después sobre el cobro capturado `ch-072` y sobre un id
que no existe.

**Then**:
4. `ch-071`: status `422` con `CANCEL_REJECTED`; `getPayment` lo da en `authorized` con
   `cancelOrigin: null` y `awaitingSince: null`, y `paymentEvents` recibe **exactamente un**
   `PaymentActionRejected` con `data` `{ paymentId, chargeRequestId: "ch-071", customerRef: "cus-1", action: "cancel", refundRequestId: null, refundAmount: null, status: "authorized" }`.
5. `ch-072` capturado: status `409` con `PAYMENT_NOT_CANCELABLE`.
6. El inexistente: status `404` con `PAYMENT_NOT_FOUND`.

**Orden de evaluación** (`cancelPayment`):
1. El cobro existe → `PAYMENT_NOT_FOUND` (`404`).
2. Repetición: en `canceling` o `canceled` → el cobro tal cual (`200`).
3. Está en `authorized` o `actionRequired` → `PAYMENT_NOT_CANCELABLE` (`409`).
4. Paso a `canceling` → `CONCURRENT_MODIFICATION` (`409`).
5. La pasarela: rechazo → `CANCEL_REJECTED` (`422`), con la vuelta a `cancelOrigin` confirmada.

### FL-STL-004: se anula un cobro que espera la autenticación del cliente

**Given**: dos cobros en `actionRequired`, `ch-080` y `ch-081` (la pasarela de prueba exigió
autenticación), con su `customerAction` leída por `getPayment`. La pasarela de prueba rechazará la
anulación de `ch-081`.

**When**: `cancelPayment` sobre `ch-080`.

**Then**:
1. Status `200` con `status: "canceled"`, `customerAction: null`, `cancelOrigin: null` y
   `awaitingSince: null`.
2. `paymentEvents` recibe **exactamente un** `PaymentCanceled` para `ch-080` con `requested: true`.

**When**: `cancelPayment` sobre `ch-081`.

**Then**:
3. Status `422` con `CANCEL_REJECTED`.
4. `getPayment` sobre `ch-081` responde `status: "actionRequired"`, la **misma** `customerAction` y
   el **mismo** `actionRequiredSince` que antes, `cancelOrigin: null` y `awaitingSince` no nulo y
   posterior al que tenía: volvió al estado del que salió.
5. `paymentEvents` recibe **exactamente un**
   `PaymentActionRejected` con `data` `{ paymentId, chargeRequestId: "ch-081", customerRef: "cus-1", action: "cancel", refundRequestId: null, refundAmount: null, status: "actionRequired" }`.

### FL-STL-005: devoluciones parciales hasta devolverlo todo

**Given**: el cobro `<p1>` (`ch-001`, 25.90 EUR) capturado, creado en este flujo como en FL-STL-001.
La operación la hace la **credencial de máquina del cliente `backoffice`**.

**When**: `refundPayment` — `POST /api/v1/payments/<p1>/refunds`
```json
{ "refundRequestId": "rf-001", "amount": 10.00, "reason": "Línea devuelta" }
```

**Then**:
1. Status `200` con el cobro en `status: "captured"`, `refundedAmount: 10.00`,
   `awaitingSince: null` y `refunds` con un elemento `{ id: <r1>, refundRequestId: "rf-001",
   amount: 10.00, reason: "Línea devuelta", status: "succeeded", createdAt, updatedAt,
   paymentId: <p1> }`.
2. `paymentEvents` recibe **exactamente un** `PaymentRefunded` con `data` `{ paymentId: <p1>,
   chargeRequestId: "ch-001", customerRef: "cus-1", refundRequestId: "rf-001", refundAmount: 10.00,
   refundedAmount: 10.00, currency: "EUR", status: "captured" }`.

**When**: se repite la misma petición `rf-001`, y después otra vez `rf-001` **sin** `amount`.

**Then**:
3. Las dos responden `200` con el cobro tal cual: la pasarela recibió **una** devolución para
   `rf-001` y `paymentEvents` no recibe otro `PaymentRefunded`.

**When**: `refundPayment` con `{ "refundRequestId": "rf-001", "amount": 5.00 }`.

**Then**:
4. Status `409` con `REFUND_REQUEST_CONFLICT`.

**When**: `refundPayment` con `{ "refundRequestId": "rf-002" }`, sin `amount`.

**Then**:
5. Status `200` con el cobro en `status: "refunded"`, `refundedAmount: 25.90` y `refunds` con dos
   elementos en orden: `rf-001` (10.00) y `rf-002` (15.90), los dos `succeeded`.
6. `paymentEvents` recibe **exactamente un** `PaymentRefunded` con `data` `{ paymentId: <p1>,
   chargeRequestId: "ch-001", customerRef: "cus-1", refundRequestId: "rf-002", refundAmount: 15.90,
   refundedAmount: 25.90, currency: "EUR", status: "refunded" }`.

**When**: `refundPayment` con `rf-003` y `amount: 1.00`, y `capturePayment` sobre `<p1>`.

**Then**:
7. La devolución: status `409` con `PAYMENT_NOT_REFUNDABLE`: `refunded` es terminal.
8. La captura, con la credencial de máquina del cliente `checkout`: status `200` con el cobro tal
   cual (`refunded`): ya estaba capturado.

### FL-STL-006: lo que no se puede devolver

**Given**: un cobro `ch-090` capturado por 15.00 EUR, un cobro `ch-091` capturado cuya devolución la
pasarela de prueba rechaza, y un cobro `ch-092` autorizado.

**When**: `refundPayment` sobre `ch-090` con `rf-010` y `amount: 20.00`, con `rf-011` y
`amount: 1.005`, y con `rf-012`, `amount: 20.00` y `amount` mal escalado a la vez (`20.005`).

**Then**:
1. `amount: 20.00`: status `422` con `REFUND_AMOUNT_EXCEEDED`, y la pasarela no recibe nada.
2. `amount: 1.005`: status `400` con `AMOUNT_SCALE_INVALID`.
3. `20.005`: status `400` con `AMOUNT_SCALE_INVALID`: la escala se comprueba antes que el saldo.

**When**: `refundPayment` sobre `ch-091` con `{ "refundRequestId": "rf-013", "amount": 5.00 }`.

**Then**:
4. Status `422` con `REFUND_REJECTED`.
5. `getPayment` sobre `ch-091` responde `status: "captured"`, `refundedAmount: 0`,
   `awaitingSince: null` y `refunds` con un elemento `rf-013` en `status: "failed"`.
6. `paymentEvents` recibe **exactamente un**
   `PaymentActionRejected` con `data` `{ paymentId, chargeRequestId: "ch-091", customerRef: "cus-1", action: "refund", refundRequestId: "rf-013", refundAmount: 5.00, status: "captured" }`.
7. Se repite `rf-013`: status `200` con el cobro tal cual y la pasarela no recibe otra devolución:
   una devolución fallida no se reintenta con la misma clave.

**When**: `refundPayment` sobre `ch-092` (autorizado), sobre un id que no existe, y sobre `ch-090`
con la **credencial de máquina del cliente `checkout`**.

**Then**:
8. `ch-092`: status `409` con `PAYMENT_NOT_REFUNDABLE`.
9. El inexistente: status `404` con `PAYMENT_NOT_FOUND`.
10. Con `checkout`: status `403`: no tiene `payment:refund`.

**Orden de evaluación** (`refundPayment`):
1. El cobro existe → `PAYMENT_NOT_FOUND` (`404`).
2. Repetición: el `refundRequestId` ya existe sobre este cobro con el mismo `amount` o sin él → el
   cobro tal cual (`200`); con otro importe o sobre otro cobro → `REFUND_REQUEST_CONFLICT` (`409`).
3. Está en `captured` → `PAYMENT_NOT_REFUNDABLE` (`409`).
4. Decimales según la moneda del cobro → `AMOUNT_SCALE_INVALID` (`400`).
5. No supera lo que queda por devolver → `REFUND_AMOUNT_EXCEEDED` (`422`).
6. Paso a `refunding` → `CONCURRENT_MODIFICATION` (`409`).
7. La pasarela: rechazo → `REFUND_REJECTED` (`422`), con la devolución `failed` y el cobro en
   `captured` confirmados.

### FL-STL-007: dos acciones sobre el mismo cobro a la vez

**Given**: un cobro `ch-100` autorizado.

**When**: a la vez, `capturePayment` y `cancelPayment` sobre `ch-100`.

**Then**:
1. Gana una: el cobro queda en `captured` o en `canceled`, nunca en un estado mezclado.
2. La otra responde `409` con `PAYMENT_NOT_CAPTURABLE`, `PAYMENT_NOT_CANCELABLE` o
   `CONCURRENT_MODIFICATION`.
3. La pasarela de prueba recibió **una sola** de las dos acciones, y `paymentEvents` recibe **un
   solo** desenlace para `ch-100` (`PaymentCaptured` o `PaymentCanceled`).

### FL-STL-008: el aviso de la pasarela

**Given**: un cobro `ch-110` autorizado que nadie captura, y otro `ch-111` autorizado.

**When**: la pasarela de prueba avisa de que la autorización de `ch-110` caducó.

**Then**:
1. En ≤ 10 s `getPayment` sobre `ch-110` responde `status: "canceled"`.
2. `paymentEvents` recibe **exactamente un** `PaymentCanceled` para `ch-110` con `requested: false`.

**When**: llega un aviso sobre `ch-111` con la firma alterada.

**Then**:
3. El servidor responde al aviso con un `4xx` y no consulta nada a la pasarela: `ch-111` sigue en
   `authorized` y `paymentEvents` no recibe nada para él.

### FL-STL-009: el aviso adelanta a la respuesta de la captura

**Given**: un cobro `ch-120` autorizado, y la pasarela de prueba configurada para avisar de la
captura **antes** de contestar a la petición de captura.

**When**: `capturePayment` sobre `ch-120`.

**Then**:
1. Status `200` con el cobro en `captured`: el aviso aplicó el desenlace primero y la operación
   relee y responde el cobro tal como está, sin `409`.
2. `paymentEvents` recibe **exactamente un** `PaymentCaptured` para `ch-120`.

## Consultas

### FL-QRY-001: los cobros de un titular, paginados

**Given**: `cus-7` con tres cobros pedidos en este orden: `ch-201`, `ch-202`, `ch-203` (10.00 EUR,
con token), y `cus-8` con un cobro `ch-204`.

**When**: `listPaymentsByCustomer` — `GET /api/v1/payments?customerRef=cus-7&page=0&size=2`.

**Then**:
1. Status `200` con `{ items: [ch-203, ch-202], page: 0, size: 2, totalElements: 3,
   totalPages: 2 }`: del más reciente al más antiguo, y cada elemento con la proyección completa del
   cobro.

**When**: `page=1&size=2`, después `page=5&size=2`, después `customerRef=cus-9&page=0&size=2`, y
después `page=0&size=500`.

**Then**:
2. `page=1`: `{ items: [ch-201], page: 1, size: 2, totalElements: 3, totalPages: 2 }`.
3. `page=5`: `{ items: [], page: 5, size: 2, totalElements: 3, totalPages: 2 }`.
4. `cus-9` (sin cobros): `{ items: [], page: 0, size: 2, totalElements: 0, totalPages: 0 }`.
5. `size=500`: `size: 100` en el sobre (se recorta a `maxSize`) y los tres cobros.
6. Sin `page` ni `size`: `size: 20` (`defaultSize`).
7. `ch-204` no aparece en ninguna página de `cus-7`.

**Notas de determinación**: a igualdad de `createdAt`, el orden es por `id` descendente.

### FL-QRY-002: leer un cobro que no existe

**When**: `getPayment` sobre un `uuid` que no existe, y `getPaymentByChargeRequest` sobre `ch-999`.

**Then**:
1. Los dos: status `404` con `PAYMENT_NOT_FOUND`.
2. `getPayment` con un id que no es `uuid`: status `400`.

## Reconciliación

### FL-REC-001: un cobro sin respuesta se resuelve preguntando

**Given**: la pasarela de prueba configurada para no contestar al siguiente cobro, pero
registrándolo.

**When**: `requestCharge` con `{ "chargeRequestId": "ch-300", "customerRef": "cus-1", "amount": 12.00,
"currency": "EUR", "paymentToken": "<tok-ok>" }`.

**Then**:
1. Status `201` con `status: "pending"`, `gatewayPaymentId: null` y `awaitingSince` no nulo: no se
   sabe todavía si se cobró, y no se reintenta.

**When**: la pasarela de prueba vuelve a contestar —sabe que `ch-300` quedó autorizado—, se envejece
la marca de espera de `ch-300` más allá de `unansweredAfterSeconds` y pasa un ciclo de
`sweepPendingPayments`.

**Then**:
2. `getPayment` sobre `ch-300` responde `status: "authorized"`, `gatewayPaymentId` no nulo y
   `awaitingSince: null`.
3. `paymentEvents` recibe **exactamente un** `PaymentAuthorized` para `ch-300`, y la pasarela de
   prueba tiene **un** cobro para `ch-300`: el barrido lo consultó por su referencia, no volvió a
   cobrar.

**When**: lo mismo —`requestCharge` con el mismo cuerpo salvo el `chargeRequestId` y un token nuevo,
sin respuesta de la pasarela, marca envejecida y un ciclo del barrido— con `ch-301`, que la pasarela
de prueba **no** llegó a registrar, y con `ch-302`, que la pasarela registró y luego canceló por su
cuenta.

**Then**:
4. `ch-301`: `status: "failed"`, `failureReason: "notReceived"`, `gatewayPaymentId: null`, y
   `paymentEvents` recibe **exactamente un** `PaymentFailed` con `failureReason: "notReceived"`.
5. `ch-302`: `status: "canceled"`, y `paymentEvents` recibe **exactamente un** `PaymentCanceled`
   con `requested: false`.

### FL-REC-002: acciones cuya respuesta se pierde, o que la pasarela nunca recibió

**Given**: un cobro `ch-310` autorizado y la pasarela de prueba configurada para no contestar a la
siguiente captura, pero haciéndola.

**When**: `capturePayment` sobre `ch-310`.

**Then**:
1. Status `200` con `status: "capturing"` y `awaitingSince` no nulo: no se sabe si capturó, y no
   se repite.
2. Se repite `capturePayment` sobre `ch-310`: status `200` con el cobro aún en `capturing`, y la
   pasarela no recibe otra captura.

**When**: la pasarela vuelve a contestar, se envejece la marca de espera de `ch-310` y pasa un ciclo
del barrido.

**Then**:
3. `getPayment` sobre `ch-310` responde `status: "captured"` y `awaitingSince: null`, y
   `paymentEvents` recibe **exactamente un** `PaymentCaptured` para `ch-310`.

**When**: lo mismo con la anulación de un cobro autorizado `ch-311` que la pasarela **no** llegó a
recibir, y con una devolución `rf-020` de 4.00 sobre un cobro capturado `ch-312` que la pasarela
tampoco recibió.

**Then**:
4. `ch-311`: la anulación responde `200` con el cobro en `canceling`; el barrido lo devuelve a
   `authorized` con
   `cancelOrigin: null` y `awaitingSince: null`; `paymentEvents` recibe **exactamente un**
   `PaymentActionRejected` con `data` `{ paymentId, chargeRequestId: "ch-311", customerRef: "cus-1", action: "cancel", refundRequestId: null, refundAmount: null, status: "authorized" }`. Se puede volver a anular.
5. `ch-312`: el barrido lo devuelve a `captured` con `refundedAmount: 0`, la devolución `rf-020`
   queda en `failed`, y `paymentEvents` recibe **exactamente un**
   `PaymentActionRejected` con `data` `{ paymentId, chargeRequestId: "ch-312", customerRef: "cus-1", action: "refund", refundRequestId: "rf-020", refundAmount: 4.00, status: "captured" }`.

**When**: una devolución `rf-021` de 6.00 sobre un cobro capturado `ch-313` cuya respuesta se pierde,
y que la pasarela completó por **5.00**; se envejece la marca y pasa un ciclo.

**Then**:
6. `ch-313` vuelve a `captured` con `refundedAmount: 5.00`, y la devolución `rf-021` queda en
   `succeeded` con `amount: 5.00`: manda el importe que informa la pasarela.
7. `paymentEvents` recibe **exactamente un** `PaymentRefunded` con `data` `{ paymentId,
   chargeRequestId: "ch-313", customerRef: "cus-1", refundRequestId: "rf-021", refundAmount: 5.00,
   refundedAmount: 5.00, currency: "EUR", status: "captured" }`.

### FL-REC-003: el cobro que espera al cliente demasiado tiempo

**Given**: tres cobros en `actionRequired` (la pasarela de prueba exigió autenticación), que la
pasarela de prueba sigue dando por pendientes del cliente:
- `ch-320`, con su `actionRequiredSince` envejecido más allá de `actionRequiredTimeoutHours` y su
  `awaitingSince` más allá de `unansweredAfterSeconds` (lleva horas sin que nadie lo consulte);
- `ch-321`, recién entrado (su `actionRequiredSince` es de ahora), pero con su `awaitingSince`
  envejecido más allá de `unansweredAfterSeconds`;
- `ch-322`, con `actionRequiredSince` y `awaitingSince` envejecidos igual que `ch-320`, y cuya
  anulación la pasarela de prueba contesta
  diciendo que el cliente **falló** la autenticación.

**When**: pasa un ciclo del barrido.

**Then**:
1. La pasarela de prueba recibe una petición de anulación para `ch-320`, y `getPayment` lo da en
   `canceled` con `customerAction: null` y `actionRequiredSince: null`. `paymentEvents` recibe
   **exactamente un** `PaymentCanceled` para `ch-320` con `requested: true`.
2. `ch-321` se consulta (la pasarela recibe una consulta de estado) pero **no** se anula: sigue en
   `actionRequired` con la misma `customerAction` y el mismo `actionRequiredSince`. Las consultas del
   barrido no cuentan para el plazo del 3DS.
3. `ch-322` termina en `failed` con `failureReason: "authenticationFailed"`, y `paymentEvents`
   recibe **exactamente un** `PaymentFailed` para él.

### FL-REC-004: se rescata lo que otra réplica dejó en vuelo, y lo reciente no se toca

**Given**: dos cobros en `capturing` sin respuesta de la pasarela de prueba, que sí capturó los dos:
`ch-330`, con su reloj (`awaitingSince`) más atrás que el umbral —el estado exacto en que queda una
réplica que murió con él en la mano—, y `ch-331`, con el reloj a ahora.

**When**: pasan dos ciclos de `sweepPendingPayments`.

**Then**:
1. La pasarela de prueba recibe una consulta de estado para `ch-330` y **ninguna** para `ch-331`.
2. `ch-330` avanza a `captured` y `paymentEvents` recibe **exactamente un** `PaymentCaptured` para él.
3. `ch-331` sigue en `capturing` con su `awaitingSince` sin cambiar.
4. Ningún cobro queda en un estado de espera con `awaitingSince` nulo.

### FL-REC-005: la pasarela que no contesta nunca

**Given**: dos cobros en `pending` con la marca envejecida: `ch-340`, a cuyas consultas la pasarela de
prueba no contesta, y `ch-341`, que la pasarela registró como autorizado y cuyas consultas sí contesta.

**When**: pasan dos ciclos del barrido.

**Then**:
1. `ch-340` sigue en `pending` con `failureReason: null`: sin respuesta no se da por fallido, y
   `paymentEvents` no recibe nada para él.
2. La pasarela de prueba recibe al menos una consulta de estado de `ch-340`, y una más en el segundo
   ciclo si su marca vuelve a superar el umbral.
3. `ch-341` queda en `authorized` y `paymentEvents` recibe **exactamente un** `PaymentAuthorized`
   para él: el cobro que no contesta no bloquea al resto de la pasada.

### FL-CLU-001: con dos réplicas, cada cobro en duda se pregunta una vez

`sweepPendingPayments` corre en **todas** las réplicas, y su consulta a la pasarela sale de la
transacción.

**Given**: dos réplicas del servicio vivas contra el mismo almacén; cinco cobros `ch-400` … `ch-404`
en `pending`, registrados por la pasarela de prueba sin contestar y con la marca envejecida; y un
cobro de control `ch-405` en `pending` con la marca a ahora.

**When**: pasan dos ciclos de `sweepPendingPayments` en las dos réplicas.

**Then**:
1. La pasarela de prueba recibe **exactamente una** consulta de estado por cada uno de los cinco, y
   ninguna por `ch-405`.
2. Los cinco quedan en `authorized`, y `paymentEvents` recibe **exactamente un**
   `PaymentAuthorized` por cada uno: cinco, ni uno más.

## Outbox

### FL-OBX-001: el desenlace sobrevive a un canal indisponible

**Given**: el canal `paymentEvents` sin mensajes y el canal de eventos **indisponible**.

**When**: `requestCharge` con `{ "chargeRequestId": "ch-500", "customerRef": "cus-1", "amount": 10.00,
"currency": "EUR", "paymentToken": "<tok-ok>" }`.

**Then**:
1. Status `201` con `status: "authorized"`: la indisponibilidad del canal no llega al consumidor.
2. `getPayment` lo da en `authorized`.
3. `paymentEvents` no ha recibido **ningún** mensaje todavía.

**When**: el canal vuelve a estar disponible.

**Then**:
4. En ≤ 10 s `paymentEvents` recibe **exactamente un** `PaymentAuthorized` para `ch-500`, con el
   `correlationId` de la petición del primer paso.
5. El servidor no informa de ningún evento abandonado.

### FL-OBX-002: el desenlace que el relay abandona no se pierde en silencio

**Given**: el canal indisponible y un cobro ya autorizado (`ch-501`) con su `PaymentAuthorized`
pendiente de salir.

**When**: se agota el presupuesto de reintentos de ese evento.

**Then**:
1. El servidor lo dice: informa de **un** evento abandonado.
2. Restablecido el canal, ese evento **no** se publica: el relay respeta que se rindió.
3. Y el canal no recibe ninguna otra cosa.

## Seguridad

### FL-SEC-001: sin credencial, sin scope y con otra audiencia

**When**: cada operación expuesta, con una petición por lo demás válida, en tres variantes.

**Then**:
1. **Sin credencial**: las once (`requestCharge`, `capturePayment`, `cancelPayment`,
   `refundPayment`, `getPayment`, `getPaymentByChargeRequest`, `listPaymentsByCustomer`,
   `savePaymentMethod`, `listPaymentMethods`, `setDefaultPaymentMethod`, `removePaymentMethod`)
   responden `401`.
2. **Con la credencial de máquina del cliente `backoffice`** (sin `payment:charge` ni
   `payment-method:*`): `requestCharge`, `capturePayment`, `cancelPayment`, `savePaymentMethod`,
   `listPaymentMethods`, `setDefaultPaymentMethod` y `removePaymentMethod` responden `403`.
3. **Con la credencial de máquina del cliente `checkout`** (sin `payment:refund`): `refundPayment`
   responde `403` (también en FL-STL-006).
4. **Con un token emitido para otra audiencia**: `getPayment`, `getPaymentByChargeRequest` y
   `listPaymentsByCustomer` responden `403`.
5. Ninguna de esas peticiones produce efecto: la pasarela de prueba no recibe nada y `paymentEvents`
   no recibe nada.

## Lo que no tiene escenario, y por qué

- **El `403` por scope de las tres lecturas de cobros.** Los dos clientes declarados tienen
  `payment:read`, así que no hay ninguna identidad del diseño a la que le falte: el rechazo por scope
  no es ejercitable sin inventar un cliente. Lo cubre el token de otra audiencia (FL-SEC-001, punto 4).
- **`CONCURRENT_MODIFICATION` en el barrido y en las escrituras del Wallet como respuesta
  observable.** En el barrido no hay respuesta: un desenlace que choca se descarta y lo resuelve la
  siguiente pasada (efecto observable en FL-REC-005, punto 3). En el Wallet, dos escrituras
  concurrentes del consumidor sobre el mismo titular no son reproducibles de forma determinista en
  caja negra sin un `Given` que fabrique la carrera; el desenlace de la carrera de un cobro ya lo
  fija FL-STL-007.
- **El aislamiento de un cobro cuyo desenlace falla al aplicarse.** La rule del barrido lo exige,
  pero provocar en caja negra una excepción al aplicar un desenlace exigiría corromper el almacén.
  Lo observable —que un cobro que no se resuelve no impide resolver otro— lo cubre FL-REC-005.
- **Que la caducidad de una autorización se conozca sin aviso.** Si se pierde el aviso, el cobro
  sigue en `authorized` hasta que alguien lo captura (FL-STL-002, `AUTHORIZATION_EXPIRED`): aceptado
  en el análisis de huecos.
- **Desvincular el medio en la pasarela al retirarlo.** La capa payments no tiene esa acción: no hay
  efecto que afirmar.
