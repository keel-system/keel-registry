# notifications — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/notifications v0.1.0. Contrato de validación para la fase de generación.

## Convenciones de determinación

- **Rutas**: todas bajo `/api/v1`.
- **Ausencia vs nulo** (declarado en el manifiesto, `conventions.nulls: include`): un campo sin valor
  viaja como `null`, en respuestas y en eventos (`senderName: null`, `sentAt: null`,
  `failureReason: null`). Una colección vacía viaja como `[]`.
- **Fechas**: todo instante es ISO-8601 en UTC. Se verifica por forma y por relación
  (`requestedAt` ≤ `sendingSince` ≤ `sentAt`; un valor que «no cambia» es igual al leído antes),
  nunca por valor exacto.
- **Identificadores generados**: forma uuid, verificados por reutilización simbólica (`m1`, `v1`…),
  nunca por valor literal.
- **Mayúsculas** (declarado en el YAML): toda dirección de correo se normaliza a minúsculas antes de
  guardarla, compararla o deduplicarla; `SuppressedAddress.address` es única sin distinguir
  mayúsculas y el filtro `recipient` de `listMessages` casa igual (`compare: ignore-case`). Los
  `templateCode` ya son minúsculas por patrón.
- **Proyecciones** (derivadas del YAML; toda enumeración de este documento las respeta). La
  auditoría es `all`: `createdAt`, `updatedAt`, `createdBy` y `updatedBy` **no** viajan en ninguna
  respuesta.
  - `SenderSettings`: `{ id, key, senderAddress, senderName, replyToAddress }`; `key` vale siempre
    `"default"`.
  - `EmailTemplate`: `{ id, code, name, description, hasActiveVersion }`. Nunca `versions`.
  - `EmailTemplateVersion` (publicar, activar, leer): `{ id, versionNumber, subject, htmlBody,
    textBody, variables, status, publishedAt, emailTemplateId }`. En `listTemplateVersions`, la misma sin
    `htmlBody` ni `textBody`. `variables` es una lista de `{ name, description, required }` en el
    orden en que se publicó.
  - `EmailMessage`: `{ id, requestedBy, idempotencyKey, recipients, renderedSubject, sender: {
    address, name, replyToAddress }, templateCode, templateVersionNumber, templateVersionId, status,
    requestedAt, sendingSince, sentAt, failedAt, failureReason, personalDataPurgedAt }`. **Nunca**
    `variableValues` (sensible, excluido de toda salida).
  - `SuppressedAddress`: `{ id, address, reason, notes, status, suppressedAt, releasedAt }`.
- **Usuarios de prueba**: `admin` (rol `notifications-admin`), `editor` (rol `template-editor`),
  `operator` (rol `notifications-operator`) y `auditor` (rol `notifications-auditor`).
- **Clientes máquina de prueba**: credencial de máquina del cliente `billing`, del cliente
  `shipping`, del cliente `dispatch-only` y del cliente `mail-provider`, todas emitidas con audiencia
  `notifications` salvo donde el escenario diga otra cosa. `requestedBy` vale el nombre del cliente
  (`"billing"`).
- **Cuerpo de error**: forma fija del generador —`{timestamp, status, error, code, message,
  details}` más `correlationId`—. Los escenarios fijan solo `code` y status; `message` no es
  contrato.
- **Errores del mecanismo** (canónicos de `framework-errors.md`): entrada fuera de cotas → `400
  VALIDATION_ERROR`; sin credencial → `401 UNAUTHENTICATED`; sin permiso, sin scope o token de otra
  audiencia → `403 ACCESS_DENIED`. Los 409 de idempotencia y de concurrencia están declarados en el
  diseño (`IDEMPOTENCY_KEY_IN_PROGRESS`, `IDEMPOTENCY_KEY_REUSED`, `CONCURRENT_MODIFICATION`).
- **Idempotencia**: en `requestNotification` la clave es el campo `idempotencyKey` del cuerpo (igual
  por HTTP y por evento), permanente y propia de cada llamante. Ninguna otra operación usa clave.
- **Paginación** (sobre canónico): `{ items, page, size, totalElements, totalPages }`, `page` base 0,
  `size` 20 por defecto, tope 100 (un `size` mayor se recorta a 100). Página vacía o fuera de rango:
  `items: []` y, si no hay ningún elemento, `totalElements: 0` y `totalPages: 0`.
- **Correo**: se observa en el **buzón de pruebas** (el relay SMTP de pruebas captura lo que acepta).
  Un correo esperado se afirma por: `From` (dirección y nombre), `To` (lista exacta, todos los
  destinatarios), `Reply-To` (presente o ausente), `Subject`, cuerpo `multipart/alternative` con una
  parte `text/html` y otra `text/plain`, y la **parte local del `Message-ID`** igual al id del
  mensaje. «No sale correo» es: el buzón no recibe ningún correo cuyo `Message-ID` tenga esa parte
  local en el plazo del escenario. El dominio del `Message-ID` es despliegue y no se afirma.
- **Relay de pruebas**: rechaza de forma definitiva toda dirección del dominio reservado
  `rejected.invalid` y acepta las demás. Los escenarios que lo necesitan **indisponible** lo dicen en
  su `Given`; cómo se programa es del arnés.
- **Despacho**: `dispatchQueuedMessages` corre cada minuto. «Pasa un ciclo del despacho» significa
  esperar a lo sumo **70 s**. Los escenarios no fuerzan el cron ni lo aceleran.
- **Parámetros** (manifiesto): `sendingTimeoutMinutes` vale `15` en pruebas y
  `personalDataRetentionMonths` vale `0`: en pruebas **todo mensaje en sent o failed es purgable**.
- **Eventos**: envoltura Keel `{ metadata, data }` en el canal `notificationEvents`. Se afirma
  `metadata.eventType` y el `data` completo; `eventId` y `occurredAt` por forma. `EmailSent.data` =
  `{ messageId, requestedBy, idempotencyKey, templateCode, templateVersionNumber, sentAt }`;
  `EmailDeliveryFailed.data` = `{ messageId, requestedBy, idempotencyKey, templateCode,
  templateVersionNumber, failureReason, failedAt }`. Ninguno lleva destinatarios.
- **Plantillas**: el marcador es `{{name}}`, con espacios opcionales dentro de las llaves. En la parte
  HTML los valores escapan solo `& < > " '` (`&amp; &lt; &gt; &quot; &#39;`); el resto (`€`, acentos)
  va literal. En el asunto, un salto de línea de un valor pasa a un espacio. Una variable opcional
  sin valor se sustituye por cadena vacía.
- **Destinatarios**: `recipients` y el `To` conservan el orden de llegada, en minúsculas y con la
  primera aparición de cada duplicado.

### Preparaciones

Cada flujo arranca desde el estado vacío y ejecuta por la API las preparaciones que su `Given` nombra.

- **P-SND** — `admin` ejecuta `configureSenderSettings` (`PUT /api/v1/sender-settings`):
  `{ "senderAddress": "Avisos@Notify.Example.com", "senderName": "Acme Avisos", "replyToAddress": "soporte@example.com" }`.
- **P-TPL** — P-SND y después `editor` crea la plantilla `invoice-issued`
  (`POST /api/v1/templates` con `{ "code": "invoice-issued", "name": "Factura emitida" }`) y publica su
  versión 1 (`POST /api/v1/templates/invoice-issued/versions`):
  ```json
  {
    "subject": "Factura {{invoiceNumber}}",
    "htmlBody": "<p>Hola {{ customerName }}, total {{amount}}.</p><p>{{note}}</p>",
    "textBody": "Hola {{customerName}}, total {{amount}}. {{note}}",
    "variables": [
      { "name": "invoiceNumber", "description": "Número de factura", "required": true },
      { "name": "customerName",  "description": "Nombre del cliente", "required": true },
      { "name": "amount",        "description": "Importe con moneda", "required": true },
      { "name": "note",          "description": "Nota opcional",     "required": false }
    ]
  }
  ```
- **Petición estándar R1** — cuerpo de `requestNotification`:
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

## Matriz de cobertura

| Operación | Flujos | Superficie |
|-----------|--------|------------|
| configureSenderSettings | FL-SND-001, FL-SND-002, FL-SND-003 | usuarios |
| getSenderSettings | FL-SND-001, FL-SND-002 | usuarios |
| createTemplate | FL-TPL-001 | usuarios |
| publishTemplateVersion | FL-TPL-010, FL-TPL-011, FL-TPL-020 | usuarios |
| activateTemplateVersion | FL-TPL-020 | usuarios |
| retireTemplate | FL-TPL-020, FL-NTF-020 | usuarios |
| getTemplate | FL-TPL-001, FL-TPL-020 | usuarios |
| listTemplates | FL-TPL-030 | usuarios |
| getTemplateVersion | FL-TPL-010, FL-TPL-020 | usuarios |
| listTemplateVersions | FL-TPL-010, FL-TPL-011 | usuarios |
| requestNotification | FL-NTF-001, FL-NTF-002, FL-NTF-003, FL-NTF-010, FL-NTF-020, FL-NTF-030, FL-NTF-040, FL-EVT-001, FL-EVT-002, FL-EVT-003 | **servidores (M2M)** + eventos |
| dispatchQueuedMessages | FL-DSP-001, FL-DSP-040, FL-RSC-001, FL-RSC-002 | reloj |
| sendQueuedMessage | FL-NTF-001, FL-DSP-010, FL-DSP-020, FL-DSP-021, FL-DSP-022, FL-DSP-030, FL-DSP-040, FL-OBX-001, FL-OBX-002 | interna |
| getMessage | FL-NTF-001, FL-MSG-001 | usuarios |
| listMessages | FL-MSG-001, FL-MSG-002 | usuarios |
| findMessageByIdempotencyKey | FL-NTF-001, FL-NTF-030, FL-EVT-002 | **servidores (M2M)** |
| suppressAddress | FL-SUP-001, FL-SUP-002, FL-NTF-010 | usuarios |
| releaseAddress | FL-SUP-001, FL-BNC-001 | usuarios |
| getSuppressedAddress | FL-SUP-001, FL-SUP-002, FL-BNC-001, FL-BNC-002 | usuarios |
| listSuppressedAddresses | FL-SUP-001, FL-SUP-003 | usuarios |
| reportEmailBounce | FL-BNC-001, FL-BNC-002 | **servidores (M2M)** |
| purgeMessagePersonalData | FL-PRG-001 | usuarios |

## Remitente

### FL-SND-001: sin remitente no se envía; configurarlo y reemplazarlo

**Given**: estado vacío. Existe la plantilla `invoice-issued` creada y publicada por `editor` como en
P-TPL, **pero sin** ejecutar P-SND.

**When** (1): `admin` ejecuta `getSenderSettings` — `GET /api/v1/sender-settings`.
**Then**:
1. Status `404`, code `SENDER_SETTINGS_NOT_CONFIGURED`.

**When** (2): `requestNotification` con R1 y credencial de máquina del cliente `billing` —
`POST /api/v1/notifications`.
**Then**:
2. Status `404`, code `SENDER_SETTINGS_NOT_CONFIGURED`.
3. `findMessageByIdempotencyKey` (`GET /api/v1/notifications/inv-2026-0001`, credencial de máquina
   del cliente `billing`) responde `404` `MESSAGE_NOT_FOUND`.
4. Tras un ciclo del despacho, el buzón no ha recibido ningún correo.

**When** (3): `admin` ejecuta `configureSenderSettings` con el cuerpo de P-SND.
**Then**:
5. Status `200`. Cuerpo: `{ id: <uuid s1>, key: "default", senderAddress: "avisos@notify.example.com",
   senderName: "Acme Avisos", replyToAddress: "soporte@example.com" }` y ningún campo más.
6. `getSenderSettings` responde `200` con el mismo cuerpo.

**When** (4): `admin` ejecuta `configureSenderSettings` con `{ "senderAddress": "Avisos2@Notify.Example.com" }`.
**Then**:
7. Status `200`. Cuerpo: `{ id: s1, key: "default", senderAddress: "avisos2@notify.example.com",
   senderName: null, replyToAddress: null }` — el `id` no cambia y los campos omitidos quedan vacíos.

**Casos borde**:
- `configureSenderSettings` sin `senderAddress` → `400 VALIDATION_ERROR`.
- `senderAddress: "no-es-correo"` → `400 VALIDATION_ERROR`.

**Notas de determinación**: el `404` del paso 2 precede a cualquier comprobación de plantilla,
variables o supresiones (ver FL-NTF-010).

### FL-SND-002: dos primeras configuraciones a la vez dejan un solo remitente

**Given**: estado vacío; no existe remitente.
**When**: `admin` lanza **a la vez** dos `configureSenderSettings`: A con
`{ "senderAddress": "a@notify.example.com" }` y B con `{ "senderAddress": "b@notify.example.com" }`.
**Then**:
1. Desenlace admisible: las dos responden `200`, o una responde `200` y la otra `409`
   `CONCURRENT_MODIFICATION`.
2. `getSenderSettings` responde `200` con `key: "default"` y un `senderAddress` que es
   `"a@notify.example.com"` o `"b@notify.example.com"`; si alguna respondió `409`, es el de la otra.
3. Dos lecturas consecutivas de `getSenderSettings` devuelven el mismo `id`: existe exactamente un
   remitente.

### FL-SND-003: el mensaje conserva el remitente congelado al aceptarlo

**Given**: P-TPL. `billing` pide R1 → `202` con `m1`, cuyo `sender` es `{ address:
"avisos@notify.example.com", name: "Acme Avisos", replyToAddress: "soporte@example.com" }`.
**When**: `admin` ejecuta `configureSenderSettings` con `{ "senderAddress": "nuevo@notify.example.com" }`;
`billing` pide R1 cambiando solo `idempotencyKey: "inv-2026-0002"` → `m2`; pasan dos ciclos del
despacho.
**Then**:
1. `m1` y `m2` quedan `sent`.
2. `getMessage` de `m1` devuelve el `sender` congelado original, y el correo de `m1` sale con `From`
   `"Acme Avisos" <avisos@notify.example.com>` y `Reply-To` `soporte@example.com`, tanto si el
   despacho lo envió antes del cambio de remitente como después.
3. `m2` tiene `sender: { address: "nuevo@notify.example.com", name: null, replyToAddress: null }` y su
   correo sale con `From` `nuevo@notify.example.com` y **sin** `Reply-To`.

## Plantillas

### FL-TPL-001: alta de plantilla, lectura y colisión de code

**Given**: estado vacío.
**When** (1): `editor` ejecuta `createTemplate` — `POST /api/v1/templates` con
`{ "code": "invoice-issued", "name": "Factura emitida", "description": "Aviso de factura" }`.
**Then**:
1. Status `200` (no `201`). Cuerpo: `{ id: <uuid t1>, code: "invoice-issued", name: "Factura
   emitida", description: "Aviso de factura", hasActiveVersion: false }` y ningún campo más.
2. `getTemplate` (`GET /api/v1/templates/invoice-issued`) responde `200` con el mismo cuerpo.

**When** (2): `editor` repite `createTemplate` con `code: "invoice-issued"` y otro `name`.
**Then**:
3. Status `409`, code `TEMPLATE_ALREADY_EXISTS`. `getTemplate` sigue devolviendo el `name` original.

**Casos borde**:
- `getTemplate` de `no-existe` → `404 TEMPLATE_NOT_FOUND`.
- `code: "Invoice"` (mayúscula) o `code: "ab"` (menos de 3) → `400 VALIDATION_ERROR`.
- Sin `name` → `400 VALIDATION_ERROR`.

### FL-TPL-010: publicar versiones, consultarlas y rechazos de contenido

**Given**: `editor` creó `invoice-issued` (sin versiones).
**When** (1): `editor` publica la versión de P-TPL — `publishTemplateVersion`,
`POST /api/v1/templates/invoice-issued/versions`.
**Then**:
1. Status `200`. Cuerpo: `{ id: <uuid v1>, versionNumber: 1, subject, htmlBody, textBody, variables
   (las cuatro, en el orden enviado), status: "active", publishedAt: <instante>, emailTemplateId: t1 }`.
2. `getTemplate` devuelve `hasActiveVersion: true`.

**When** (2): `editor` publica una segunda versión con `subject: "Tu factura {{invoiceNumber}}"` y el
resto igual.
**Then**:
3. Status `200`, `versionNumber: 2`, `status: "active"`.
4. `getTemplateVersion` de la versión 1 (`GET /api/v1/templates/invoice-issued/versions/1`) responde
   `200` con `status: "archived"` y el resto de campos idénticos a los del paso 1.
5. `listTemplateVersions` (`GET /api/v1/templates/invoice-issued/versions`) responde `200` con
   `items` = [versión 2 (`active`), versión 1 (`archived`)] en ese orden, cada una **sin** `htmlBody`
   ni `textBody`; `totalElements: 2`.

**When** (3): `editor` publica con `subject: "Factura {{invoiceNumber}} {{ dueDate }}"` sin declarar
`dueDate`.
**Then**:
6. Status `422`, code `UNDECLARED_TEMPLATE_VARIABLES`. `listTemplateVersions` sigue con 2 versiones.

**Orden de evaluación** (`publishTemplateVersion`):
1. Forma del cuerpo (cotas de `subject`, cuerpos, `variables`) → `400 VALIDATION_ERROR`.
2. La plantilla existe → `TEMPLATE_NOT_FOUND` (`404`).
3. Ningún `name` repetido en `variables` → `DUPLICATE_TEMPLATE_VARIABLES` (`422`).
4. Todo marcador está declarado → `UNDECLARED_TEMPLATE_VARIABLES` (`422`).
5. Escritura concurrente sobre la plantilla → `CONCURRENT_MODIFICATION` (`409`, ver FL-TPL-011).

**Casos borde**:
- `variables` con dos entradas `name: "amount"` → `422 DUPLICATE_TEMPLATE_VARIABLES`.
- Publicar en `no-existe` con variables repetidas → `404 TEMPLATE_NOT_FOUND` (la guarda 2 precede a la 3).
- Variables repetidas **y** marcador sin declarar → `422 DUPLICATE_TEMPLATE_VARIABLES` (la 3 precede a la 4).
- Sin `textBody` → `400 VALIDATION_ERROR`; `subject` de 301 caracteres → `400 VALIDATION_ERROR`;
  51 variables → `400 VALIDATION_ERROR`.
- `getTemplateVersion` de la versión 9 → `404 TEMPLATE_VERSION_NOT_FOUND`.
- `listTemplateVersions` de `no-existe` → `404 TEMPLATE_NOT_FOUND`.

### FL-TPL-011: dos publicaciones a la vez no dejan dos versiones activas

**Given**: P-TPL (versión 1 active).
**When**: `editor` lanza **a la vez** dos `publishTemplateVersion` con contenidos válidos distintos.
**Then**:
1. Desenlace admisible: las dos responden `200` con `versionNumber` distintos (2 y 3), o una responde
   `200` y la otra `409` `CONCURRENT_MODIFICATION`.
2. `listTemplateVersions` devuelve **exactamente una** versión `active` —la de mayor `versionNumber`—
   y ningún `versionNumber` repetido.
3. `getTemplate` devuelve `hasActiveVersion: true`.

### FL-TPL-020: revertir, retirar y volver a publicar

**Given**: P-TPL y `editor` publicó la versión 2 (la 1 queda `archived`).
**When** (1): `editor` ejecuta `activateTemplateVersion` sobre la versión 1 —
`POST /api/v1/templates/invoice-issued/versions/1/activate`.
**Then**:
1. Status `200` con la versión 1 completa y `status: "active"`.
2. `getTemplateVersion` de la 2 devuelve `status: "archived"`.

**When** (2): `editor` repite `activateTemplateVersion` sobre la versión 1.
**Then**:
3. Status `200` con el mismo cuerpo; la 2 sigue `archived` (no-op).

**When** (3): `editor` ejecuta `retireTemplate` — `POST /api/v1/templates/invoice-issued/retire`.
**Then**:
4. Status `200` con `{ id: t1, code: "invoice-issued", name: "Factura emitida", description: null,
   hasActiveVersion: false }`.
5. `listTemplateVersions` devuelve las dos versiones `archived`: el historial se conserva.

**When** (4): `editor` repite `retireTemplate`.
**Then**:
6. Status `409`, code `TEMPLATE_NOT_PUBLISHED` (no hay versión active que archivar).

**When** (5): `editor` publica contenido nuevo.
**Then**:
7. Status `200`, `versionNumber: 3` (no se reutiliza ningún número), `status: "active"`; `getTemplate`
   devuelve `hasActiveVersion: true`.

**Casos borde**:
- `activateTemplateVersion` de la versión 7 → `404 TEMPLATE_VERSION_NOT_FOUND`.
- `activateTemplateVersion` o `retireTemplate` sobre `no-existe` → `404 TEMPLATE_NOT_FOUND`.

### FL-TPL-030: listado de plantillas, filtro y paginación

**Given**: P-SND; `editor` crea `b-receipt`, `a-welcome` y `c-reminder` y publica una versión solo en
`a-welcome` y `c-reminder`.
**When**: `listTemplates` con distintos parámetros — `GET /api/v1/templates`.
**Then**:
1. Sin parámetros: `200`, `items` en orden `code` ascendente = [`a-welcome`, `b-receipt`,
   `c-reminder`], `page: 0`, `size: 20`, `totalElements: 3`, `totalPages: 1`.
2. `?hasActiveVersion=false`: `items` = [`b-receipt`], `totalElements: 1`.
3. `?size=2`: `items` = [`a-welcome`, `b-receipt`], `totalPages: 2`; `?size=2&page=1`: `items` =
   [`c-reminder`].
4. `?page=5`: `items: []`, `page: 5`, `size: 20`, `totalElements: 3`, `totalPages: 1`.
5. `?size=500`: `size: 100` en el sobre.
6. Tras publicar una versión en `b-receipt`, `?hasActiveVersion=false` devuelve `items: []`,
   `totalElements: 0`, `totalPages: 0`.

## Envíos servidor a servidor

### FL-NTF-001: pedir un correo, recibirlo y consultarlo

**Given**: P-TPL.
**When**: `requestNotification` con credencial de máquina del cliente `billing` —
`POST /api/v1/notifications` con R1 más la variable no declarada
`{ "name": "coupon", "value": "X" }` y el destinatario repetido `"ana@example.com"`.
**Then**:
1. Status `202`. Cuerpo: `{ id: <uuid m1>, requestedBy: "billing", idempotencyKey:
   "inv-2026-0001", recipients: ["ana@example.com", "luis@example.com"], renderedSubject: "Factura
   F-0001", sender: { address: "avisos@notify.example.com", name: "Acme Avisos", replyToAddress:
   "soporte@example.com" }, templateCode: "invoice-issued", templateVersionNumber: 1,
   templateVersionId: v1, status: "queued", requestedAt: <instante>, sendingSince: null, sentAt:
   null, failedAt: null, failureReason: null, personalDataPurgedAt: null }`. Sin `variableValues`.
2. `findMessageByIdempotencyKey` (`GET /api/v1/notifications/inv-2026-0001`, credencial de máquina del
   cliente `billing`) responde `200` con el mismo cuerpo, salvo `status` y los instantes si el despacho
   ya pasó.
3. Tras un ciclo del despacho, el buzón recibe **un** correo: `From` `"Acme Avisos"
   <avisos@notify.example.com>`, `To` [`ana@example.com`, `luis@example.com`], `Reply-To`
   `soporte@example.com`, `Subject` `Factura F-0001`, parte `text/plain` `Hola Ana, total 10,00 €. `
   y parte `text/html` `<p>Hola Ana, total 10,00 €.</p><p></p>`, `Message-ID` con parte local `m1`.
   Ninguna parte contiene `X`.
4. `getMessage` (`admin`, `GET /api/v1/messages/m1`) devuelve `status: "sent"`, `sendingSince` y
   `sentAt` con valor (`requestedAt` ≤ `sendingSince` ≤ `sentAt`) y `failedAt: null`.
5. El canal `notificationEvents` recibe **un** `EmailSent` con `data: { messageId: m1, requestedBy:
   "billing", idempotencyKey: "inv-2026-0001", templateCode: "invoice-issued",
   templateVersionNumber: 1, sentAt: <el de getMessage> }`.

### FL-NTF-002: repetir la misma petición no manda un segundo correo

**Given**: P-TPL; `billing` pidió R1 → `m1`, y tras un ciclo del despacho `m1` está `sent` (un correo
en el buzón).
**When** (1): `billing` repite R1 con `recipients` y `variables` en otro orden y
`"LUIS@example.com"` en mayúsculas.
**Then**:
1. Status `202` con el cuerpo de `m1` tal como está ahora (`status: "sent"`).
2. Tras otro ciclo del despacho, el buzón sigue con **un** solo correo de `m1` y no hay otro mensaje:
   `listMessages` (`admin`) devuelve `totalElements: 1`.

**When** (2): `billing` repite `idempotencyKey: "inv-2026-0001"` con `amount: "11,00 €"`.
**Then**:
3. Status `409`, code `IDEMPOTENCY_KEY_REUSED`. No sale correo.

**When** (3): credencial de máquina del cliente `shipping` envía R1 tal cual.
**Then**:
4. Status `202` con un mensaje **nuevo** `m2` (`id` ≠ `m1`, `requestedBy: "shipping"`): cada llamante
   tiene su propio espacio de claves. Tras un ciclo, el buzón recibe el correo de `m2`.

### FL-NTF-003: dos peticiones con la misma clave a la vez

**Given**: P-TPL.
**When**: `requestNotification` — `billing` envía R1 **a la vez** dos veces (carrera de dos peticiones con la misma clave).
**Then**:
1. Desenlace admisible: las dos responden `202` con el mismo `id`, o una responde `202` y la otra `409`
   `IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `listMessages` (`admin`) devuelve `totalElements: 1`, sea cual sea el ganador.
3. Tras dos ciclos del despacho, el buzón tiene **exactamente un** correo con esa parte local de
   `Message-ID`.

### FL-NTF-010: rechazos de la petición, su precedencia y que no salga correo

**Given**: P-TPL; además `editor` crea `draft-tpl` sin publicar, y `operator` suprime
`blocked@example.com` (`PUT /api/v1/suppressions/blocked@example.com` con
`{ "reason": "manual" }`).
**When**: `billing` envía variantes de R1, cada una con su propia `idempotencyKey`.
**Then**:
1. `templateCode: "no-existe"` → `404 TEMPLATE_NOT_FOUND`.
2. `templateCode: "draft-tpl"` → `409 TEMPLATE_NOT_PUBLISHED`.
3. Sin la variable `amount` → `422 MISSING_TEMPLATE_VARIABLES`.
4. `recipients: ["ana@example.com", "Blocked@Example.com"]` → `422 RECIPIENT_SUPPRESSED`.
5. Dos valores para `amount` → `422 DUPLICATE_TEMPLATE_VARIABLES`.
6. Para cada una de las cinco, `findMessageByIdempotencyKey` responde `404` y, tras un ciclo del
   despacho, el buzón no recibe ningún correo.

**Orden de evaluación** (`requestNotification`):
1. Forma de la petición → `400 VALIDATION_ERROR`; variables repetidas → `DUPLICATE_TEMPLATE_VARIABLES` (`422`).
2. Repetición: misma clave del mismo llamante → mensaje original (`202`), `IDEMPOTENCY_KEY_REUSED`
   (`409`) o `IDEMPOTENCY_KEY_IN_PROGRESS` (`409`).
3. Remitente configurado → `SENDER_SETTINGS_NOT_CONFIGURED` (`404`, ver FL-SND-001).
4. La plantilla existe → `TEMPLATE_NOT_FOUND` (`404`).
5. Tiene versión active → `TEMPLATE_NOT_PUBLISHED` (`409`).
6. Variables required presentes → `MISSING_TEMPLATE_VARIABLES` (`422`).
7. Ningún destinatario suprimido → `RECIPIENT_SUPPRESSED` (`422`).

**Casos borde**:
- `templateCode: "no-existe"` **y** sin `amount` **y** destinatario suprimido → `404 TEMPLATE_NOT_FOUND`.
- `templateCode: "draft-tpl"` con variables repetidas → `422 DUPLICATE_TEMPLATE_VARIABLES` (la 1 precede a todo).
- Sin `amount` **y** destinatario suprimido → `422 MISSING_TEMPLATE_VARIABLES` (la 6 precede a la 7).
- `recipients: []` o 21 destinatarios → `400 VALIDATION_ERROR`; `idempotencyKey: "abc"` (menos de 8)
  → `400 VALIDATION_ERROR`; dirección mal formada → `400 VALIDATION_ERROR`.
- **La repetición precede a la supresión**: `billing` pide R1 (`202`, `m1`); `operator` suprime
  `ana@example.com`; `billing` repite R1 → `202` con `m1`, no `RECIPIENT_SUPPRESSED`.

### FL-NTF-020: la repetición se honra aunque la plantilla se haya retirado

**Given**: P-TPL; `billing` pidió R1 → `m1`.
**When**: `editor` ejecuta `retireTemplate` sobre `invoice-issued`; `billing` repite R1.
**Then**:
1. Status `202` con `m1`: no `TEMPLATE_NOT_PUBLISHED`.
2. Una petición **nueva** (`idempotencyKey: "inv-2026-0009"`) responde `409 TEMPLATE_NOT_PUBLISHED`.
3. `m1` se envía igual en el siguiente ciclo (congeló su versión): el buzón recibe su correo.

### FL-NTF-030: la superficie servidor a servidor y sus credenciales

**Given**: P-TPL; `billing` pidió R1 → `m1`.
**When**: se llama con distintas credenciales.
**Then**:
1. `requestNotification` sin credencial → `401 UNAUTHENTICATED`.
2. Credencial de máquina del cliente `dispatch-only` con R1 y `idempotencyKey: "dsp-0001"` → `202`,
   `requestedBy: "dispatch-only"`.
3. Credencial de máquina del cliente `dispatch-only` en `findMessageByIdempotencyKey` de `dsp-0001`
   → `403 ACCESS_DENIED` (no tiene `notification:read`).
4. Credencial de máquina del cliente `mail-provider` en `requestNotification` → `403 ACCESS_DENIED`.
5. Credencial de máquina del cliente `billing` emitida para otra audiencia → `403 ACCESS_DENIED`.
6. Token del usuario `admin` en `requestNotification` → `403 ACCESS_DENIED` (no tiene el scope).
7. Credencial de máquina del cliente `shipping` en `findMessageByIdempotencyKey` de `inv-2026-0001`
   → `404 MESSAGE_NOT_FOUND`: solo encuentra mensajes del propio llamante.

### FL-NTF-040: el coste de comprobar supresiones no crece con los destinatarios

**Given**: P-TPL.
**When**: `billing` pide una notificación con 1 destinatario y otra con 20 destinatarios distintos.
**Then**:
1. Las dos responden `202`.
2. El trabajo de la operación con 20 destinatarios **no crece** respecto al de 1: la comprobación de
   supresión se hace en lote, no una consulta por destinatario. Cómo se mide es del generador.

## Eventos entrantes

### FL-EVT-001: pedir un correo por el canal de eventos, y su reentrega

**Given**: P-TPL.
**When** (1): llega a `notificationRequests` un `NotificationRequested` con envoltura Keel,
`metadata.source: "billing"`, `metadata.eventId: e1` y `data` = R1.
**Then**:
1. `findMessageByIdempotencyKey` de `inv-2026-0001` con credencial de máquina del cliente `billing`
   responde `200` con `requestedBy: "billing"`, `status` `queued` o posterior.
2. Tras un ciclo del despacho, el buzón recibe un correo con ese mensaje y `notificationEvents` un
   `EmailSent`.

**When** (2): **reentrega** del mismo mensaje (mismo `metadata.eventId: e1`).
**Then**:
3. No se crea otro mensaje (`listMessages` devuelve `totalElements: 1`) y no sale un segundo correo.

**When** (3): `billing` envía R1 por HTTP.
**Then**:
4. `202` con el mismo mensaje: la misma clave por las dos puertas es el mismo envío.

### FL-EVT-002: lo que no se puede aceptar va a la deadLetter

**Given**: P-TPL.
**When**: llegan a `notificationRequests`:
(a) `metadata.source: "billing"` con `templateCode: "no-existe"` e `idempotencyKey: "evt-0001"`;
(b) `metadata.source: "unknown-system"` con R1 e `idempotencyKey: "evt-0002"`;
(c) `metadata.source: "mail-provider"` con R1 e `idempotencyKey: "evt-0003"`.
**Then**:
1. Los tres acaban en la deadLetter de la suscripción (el (a) por un rechazo de negocio; (b) y (c)
   porque su identidad no resuelve a un llamante registrado).
2. `findMessageByIdempotencyKey` de `evt-0001` con credencial de máquina del cliente `billing` → `404
   MESSAGE_NOT_FOUND`: así reconcilia el emisor que su petición no se aceptó.
3. `listMessages` (`admin`) devuelve `totalElements: 0` y el buzón no recibe nada.

### FL-EVT-003: un fallo transitorio se reintenta y, reinyectado, envía una sola vez

**Given**: P-TPL; el almacén del servicio indisponible durante los reintentos de la suscripción.
**When**: llega `NotificationRequested` de `billing` con R1; se agotan los 3 intentos; se restablece
el almacén y el mensaje se **reinyecta** desde la deadLetter dos veces.
**Then**:
1. Agotados los reintentos de la suscripción, el mensaje está en la deadLetter. Restablecido el
   almacén y **antes** de reinyectar, `findMessageByIdempotencyKey` de `inv-2026-0001` → `404
   MESSAGE_NOT_FOUND`.
2. Tras reinyectarlo dos veces, existe **exactamente un** mensaje de `inv-2026-0001` y el buzón recibe
   **un** correo.

## Despacho

### FL-DSP-001: el despacho vacía la cola

**Given**: P-TPL; `billing` pide tres notificaciones (`k-001`, `k-002`, `k-003`).
**When**: pasa un ciclo del despacho.
**Then**:
1. Los tres mensajes están `sent`; el buzón tiene tres correos y `notificationEvents` tres `EmailSent`.
2. Otro ciclo no envía nada nuevo.

### FL-DSP-010: una dirección suprimida después de aceptar no recibe el correo

**Given**: P-TPL. `billing` pide R1 → `m1` (`queued`) y, antes del siguiente ciclo del despacho,
`operator` suprime `luis@example.com`.
**When**: pasa un ciclo del despacho.
**Then**:
1. `m1` queda `failed` con `failureReason: "recipient-suppressed"`, `failedAt` con valor y
   `sendingSince: null`.
2. El buzón no recibe ningún correo de `m1` (ni a `ana@example.com`).
3. `notificationEvents` recibe **un** `EmailDeliveryFailed` con `data: { messageId: m1, requestedBy:
   "billing", idempotencyKey: "inv-2026-0001", templateCode: "invoice-issued",
   templateVersionNumber: 1, failureReason: "recipient-suppressed", failedAt }`.

**Notas de determinación**: el arnés hace la supresión antes de que corra el ciclo; si el ciclo se
adelanta, el flujo se repite.

### FL-DSP-020: el relay rechaza a todos los destinatarios

**Given**: P-TPL; `billing` pide R1 con `recipients: ["ana@rejected.invalid", "luis@rejected.invalid"]` → `m1`.
**When**: pasa un ciclo del despacho.
**Then**:
1. `m1` queda `failed` con `failureReason: "delivery-rejected"`, `sendingSince` y `failedAt` con valor.
2. `notificationEvents` recibe un `EmailDeliveryFailed` con `failureReason: "delivery-rejected"`.
3. Dos ciclos más tarde sigue `failed` y no hay un segundo intento ni un segundo evento.

### FL-DSP-021: el relay rechaza a uno y acepta al otro

**Given**: P-TPL; `billing` pide R1 con `recipients: ["ana@example.com", "luis@rejected.invalid"]` → `m1`.
**When**: pasa un ciclo del despacho.
**Then**:
1. `m1` queda `sent`: el correo salió para al menos un destinatario.
2. El buzón recibe el correo para `ana@example.com`; `notificationEvents` un `EmailSent`.

### FL-DSP-022: con el relay caído el mensaje falla y no se reintenta

**Given**: P-TPL; el relay **indisponible**; `billing` pide R1 → `m1`.
**When**: pasa un ciclo del despacho; después el relay vuelve y pasan dos ciclos más.
**Then**:
1. `m1` queda `failed` con `failureReason: "delivery-error"` y se publica `EmailDeliveryFailed`.
2. Con el relay de vuelta, `m1` sigue `failed` y el buzón no recibe su correo: no hay reintento.

### FL-DSP-030: render de los cuerpos, escapado y cabeceras

**Given**: P-TPL.
**When**: `billing` pide con `customerName: "Ana <b>&</b>"`, `invoiceNumber: "F-1\nX"`, `amount:
"5 €"`, `note: "Gracias"`, `recipients: ["ana@example.com"]` → `m1`; pasa un ciclo.
**Then**:
1. `renderedSubject` de `m1` es `"Factura F-1 X"` y el `Subject` del correo es el mismo.
2. Parte `text/html`: `<p>Hola Ana &lt;b&gt;&amp;&lt;/b&gt;, total 5 €.</p><p>Gracias</p>`.
3. Parte `text/plain`: `Hola Ana <b>&</b>, total 5 €. Gracias`.
4. `To` = [`ana@example.com`], `Reply-To` `soporte@example.com`, parte local del `Message-ID` = `m1`.

### FL-DSP-040: dos réplicas del despacho no mandan dos correos

**Given**: **dos instancias** del servicio vivas (clúster de 2 réplicas). P-TPL; `billing` pide cinco
notificaciones (`c-001`…`c-005`), y un sexto mensaje `c-000` pedido y ya `sent` antes de arrancar la
segunda instancia (fila de control).
**When**: pasan dos ciclos de `dispatchQueuedMessages` en las dos instancias.
**Then**:
1. El buzón recibe **exactamente un** correo por cada uno de `c-001`…`c-005` (cinco en total).
2. El buzón **no** recibe un segundo correo de `c-000`.
3. `notificationEvents` recibe exactamente cinco `EmailSent` nuevos.

**Notas de determinación**: la instancia que pierde la transición `queued → sending` recibe
internamente `MESSAGE_NOT_QUEUED` y salta el mensaje sin enviar; no es observable salvo por el
conteo de correos.

### FL-RSC-001: dispatchQueuedMessages rescata lo que otra réplica dejó a medias en sending

**Given**: P-TPL; `billing` pide R1 → `m1`, y el arnés lo deja en `sending` con su `sendingSince`
más atrás que `sendingTimeoutMinutes` — el estado en que queda una réplica que murió con él en la
mano.
**When**: pasa un ciclo de `dispatchQueuedMessages`.
**Then**:
1. `m1` queda `failed` con `failureReason: "stuck-in-sending"` y `failedAt` con valor.
2. `notificationEvents` recibe **un** `EmailDeliveryFailed` con `failureReason: "stuck-in-sending"`.
3. El buzón **no** recibe ningún correo de `m1`: el rescate nunca reenvía.
4. Ningún mensaje queda en `sending` con `sendingSince: null`.

**Notas de determinación**: el arnés fija `m1` en `sending` antes de que corra ningún ciclo del
despacho; si un ciclo se adelanta y envía `m1`, el flujo se repite.

### FL-RSC-002: dispatchQueuedMessages no rescata lo que acaba de entrar en sending

**Given**: P-TPL; `billing` pide R1 → `m1`, y el arnés lo deja en `sending` con `sendingSince` = ahora.
**When**: pasan dos ciclos de `dispatchQueuedMessages`; otra réplica podría estar enviándolo.
**Then**:
1. `m1` sigue `sending` con el mismo `sendingSince`; no se publica ningún evento ni sale correo.

**Notas de determinación**: igual que FL-RSC-001, el arnés fija `m1` en `sending` antes del primer
ciclo; si un ciclo se adelanta y envía `m1`, el flujo se repite.

### FL-OBX-001: el evento sobrevive a un canal indisponible

**Given**: P-TPL; el canal `notificationEvents` sin mensajes y el canal de eventos **indisponible**.
**When**: `billing` pide R1 → `m1`; pasa un ciclo del despacho.
**Then**:
1. `m1` queda `sent` y el buzón recibe su correo: la indisponibilidad del canal no afecta al envío.
2. `notificationEvents` no ha recibido ningún mensaje todavía.

**When**: el canal vuelve a estar disponible.
**Then**:
3. En ≤ 10 s `notificationEvents` recibe **exactamente un** `EmailSent` de `m1`, con su `data` completo.
4. El servidor no informa de ningún evento abandonado.

### FL-OBX-002: el evento que el relay de eventos abandona no se pierde en silencio

**Given**: el canal de eventos indisponible; `m1` ya `sent` con su `EmailSent` pendiente de salir.
**When**: se agota el presupuesto de reintentos de ese evento.
**Then**:
1. El servidor informa de **un** evento abandonado.
2. Restablecido el canal, ese `EmailSent` **no** se publica.
3. Y el canal no recibe ninguna otra cosa.

## Rebotes

### FL-BNC-001: un rebote permanente suprime la dirección

**Given**: P-TPL; `billing` pidió R1 → `m1`, ya `sent`.
**When** (1): credencial de máquina del cliente `mail-provider` ejecuta `reportEmailBounce` —
`POST /api/v1/messages/m1/bounces` con
`{ "recipient": "Luis@Example.com", "bounceType": "transient", "diagnostic": "mailbox full" }`.
**Then**:
1. Status `204`, sin cuerpo. `getSuppressedAddress` de `luis@example.com` → `404
   SUPPRESSED_ADDRESS_NOT_FOUND`: un rebote blando no cambia nada.

**When** (2): el mismo con `bounceType: "permanent"` y `diagnostic: "550 user unknown"`.
**Then**:
2. Status `204`.
3. `getSuppressedAddress` (`operator`, `GET /api/v1/suppressions/luis@example.com`) → `200` con `{ id:
   <uuid a1>, address: "luis@example.com", reason: "hard-bounce", notes: <contiene m1 y "550 user
   unknown">, status: "active", suppressedAt: <instante>, releasedAt: null }`.
4. `getMessage` de `m1` sigue `sent`: el rebote no cambia el mensaje.

**When** (3): `mail-provider` repite el rebote permanente.
**Then**:
5. `204`; el registro es idéntico al del paso 3 (no-op).

**When** (4): `operator` libera la dirección (`POST /api/v1/suppressions/luis@example.com/release`)
y `mail-provider` vuelve a reportar el rebote permanente.
**Then**:
6. El registro vuelve a `active` con el mismo `id`, `reason: "hard-bounce"`, `releasedAt: null` y un
   `suppressedAt` posterior: el rebote reactiva aunque la hubiera liberado una persona.

**Casos borde**:
- `reportEmailBounce` sobre un id inexistente → `404 MESSAGE_NOT_FOUND`.
- `recipient: "otra@example.com"` (no está en `m1`) → `422 RECIPIENT_NOT_IN_MESSAGE`.
- Credencial de máquina del cliente `billing` → `403 ACCESS_DENIED`; sin credencial → `401`.
- `bounceType: "soft"` → `400 VALIDATION_ERROR`.

### FL-BNC-002: dos rebotes de la misma dirección a la vez

**Given**: P-TPL; `m1` `sent` a `luis@example.com`; no hay supresiones.
**When**: `mail-provider` envía **a la vez** dos rebotes permanentes de `luis@example.com` sobre `m1`.
**Then**:
1. Desenlace admisible: los dos responden `204`, o uno `204` y el otro `409`
   `SUPPRESSED_ADDRESS_ALREADY_EXISTS`.
2. `getSuppressedAddress` devuelve un único registro `active`, y `listSuppressedAddresses` tiene
   `totalElements: 1`.

## Supresiones

### FL-SUP-001: suprimir, consultar, liberar y volver a suprimir

**Given**: P-SND.
**When** (1): `operator` ejecuta `suppressAddress` — `PUT /api/v1/suppressions/Pepe@Example.com` con
`{ "reason": "complaint", "notes": "pidió no recibir" }`.
**Then**:
1. Status `200` con `{ id: <uuid a1>, address: "pepe@example.com", reason: "complaint", notes: "pidió
   no recibir", status: "active", suppressedAt: <instante>, releasedAt: null }`.
2. Repetirlo con `reason: "manual"` → `200` con el registro sin cambios (no-op).
3. `listSuppressedAddresses` devuelve `items` = [a1], `totalElements: 1`.

**When** (2): `operator` ejecuta `releaseAddress` dos veces —
`POST /api/v1/suppressions/pepe@example.com/release`.
**Then**:
4. Las dos responden `200` con `status: "released"` y el mismo `releasedAt` (la segunda es no-op).
5. `getSuppressedAddress` devuelve el registro `released`; `listSuppressedAddresses` devuelve
   `items: []`, `totalElements: 0`, `totalPages: 0`.

**When** (3): `operator` suprime de nuevo con `{ "reason": "manual" }` (sin `notes`).
**Then**:
6. `200` con el mismo `id`, `status: "active"`, `reason: "manual"`, `notes: null`, `releasedAt: null`
   y `suppressedAt` posterior al del paso 1.

**Casos borde**:
- `releaseAddress` de `nunca@example.com` → `404 SUPPRESSED_ADDRESS_NOT_FOUND`.
- `getSuppressedAddress` de `nunca@example.com` → `404 SUPPRESSED_ADDRESS_NOT_FOUND`.
- `reason: "bounced"` → `400 VALIDATION_ERROR`; dirección mal formada → `400 VALIDATION_ERROR`.

### FL-SUP-002: dos supresiones de la misma dirección a la vez

**Given**: P-SND; no hay supresiones.
**When**: `operator` y `admin` suprimen **a la vez** `x@example.com`.
**Then**:
1. Desenlace admisible: las dos `200`, o una `200` y la otra `409`
   `SUPPRESSED_ADDRESS_ALREADY_EXISTS`; si una escritura choca con la versión ya modificada, `409`
   `CONCURRENT_MODIFICATION`.
2. `getSuppressedAddress` devuelve un único registro `active`; `listSuppressedAddresses` tiene
   `totalElements: 1`.

### FL-SUP-003: listado de supresiones, orden y paginación

**Given**: P-SND; `operator` suprime en este orden `a@example.com`, `b@example.com` y `c@example.com`.
**When**: `listSuppressedAddresses` — `GET /api/v1/suppressions`.
**Then**:
1. `items` en orden `suppressedAt` descendente = [`c`, `b`, `a`], `totalElements: 3`.
2. `?size=2` → [`c`, `b`], `totalPages: 2`; `?size=2&page=1` → [`a`].
3. `?page=4` → `items: []`, `totalElements: 3`; `?size=500` → `size: 100`.

## Consulta de correos

### FL-MSG-001: leer y listar correos con filtros

**Given**: P-TPL; en este orden, `billing` pide `k-001` a `ana@example.com`, `billing` pide `k-002` a
`luis@example.com` y `shipping` pide `k-003` a `zed@rejected.invalid`. Tras un ciclo del despacho,
`k-001` y `k-002` están `sent` y `k-003` `failed` (`delivery-rejected`).
**When**: `operator` consulta.
**Then**:
1. `getMessage` de `k-001` → `200` con la proyección completa de `EmailMessage`.
2. `getMessage` de un id inexistente → `404 MESSAGE_NOT_FOUND`.
3. `listMessages` sin filtros → `items` = [`k-003`, `k-002`, `k-001`] (`requestedAt` descendente,
   desempate por `id` ascendente), `totalElements: 3`.
4. `?recipient=ANA@example.com` → [`k-001`].
5. `?requestedBy=billing` → [`k-002`, `k-001`]; `?requestedBy=shipping` → [`k-003`];
   `?status=failed` → [`k-003`]; `?templateCode=invoice-issued&status=sent` → [`k-002`, `k-001`].
6. `?requestedFrom=<requestedAt de k-002>` → [`k-003`, `k-002`] (inclusivo);
   `?requestedTo=<requestedAt de k-002>` → [`k-001`] (exclusivo).
7. `?requestedFrom=2026-10-05T10:00:00Z&requestedTo=2026-10-05T09:00:00Z` → `422 INVALID_TIME_WINDOW`.
8. Ningún elemento trae `variableValues`.

### FL-MSG-002: paginación del listado de correos

**Given**: P-TPL; `billing` pide 3 notificaciones.
**When**: `listMessages` con paginación.
**Then**:
1. `?size=2` → 2 elementos, `totalElements: 3`, `totalPages: 2`; `?size=2&page=1` → el más antiguo.
2. `?page=9` → `items: []`; `?size=500` → `size: 100`.
3. `?templateCode=no-such-tpl` → `items: []`, `totalElements: 0`, `totalPages: 0`.

## Purga de datos personales

### FL-PRG-001: purga manual y repetición tras la purga

**Given**: P-TPL; `billing` pidió R1 → `m1`, y tras un ciclo del despacho `m1` está `sent`.
**When** (1): `admin` ejecuta `purgeMessagePersonalData` —
`POST /api/v1/maintenance/purge-personal-data`.
**Then**:
1. Status `200` con `{ purgedCount: 1 }` (con retención 0 en pruebas, todo mensaje terminal es purgable).
2. `getMessage` de `m1`: `recipients: []`, `renderedSubject: null`, `personalDataPurgedAt` con valor;
   el resto de campos sin cambios (`idempotencyKey`, `status: "sent"`, `sender`…).
3. `listMessages?recipient=ana@example.com` no devuelve `m1`.

**When** (2): `admin` repite la purga.
**Then**:
4. `200` con `{ purgedCount: 0 }`.

**When** (3): `billing` repite R1.
**Then**:
5. `202` con `m1` purgado: tras la purga basta el mismo `templateCode`. No sale correo.
6. `billing` con `idempotencyKey: "inv-2026-0001"` y `templateCode: "other-tpl"` → `409
   IDEMPOTENCY_KEY_REUSED`.
7. `mail-provider` reporta un rebote de `ana@example.com` sobre `m1` → `422 RECIPIENT_NOT_IN_MESSAGE`.

**Casos borde**:
- `operator` lanza la purga → `403 ACCESS_DENIED`; sin credencial → `401`.

**Notas de determinación**: la ejecución programada de las 03:00 UTC no se espera; su política es la
misma que la manual y se verifica a través de esta. Que un mensaje `queued` o `sending` no se purgue
es política declarada y se verifica en estático: fijar uno en ese estado mientras corre la purga
depende del reloj del despacho.

## Transversal

### FL-SEC-001: CORS — preflight

**Given**: la SPA de back-office corre en un origen web distinto.
**When**: `OPTIONS /api/v1/templates` sin credencial, con `Access-Control-Request-Method: POST` y
`Access-Control-Request-Headers: Authorization, Content-Type`.
**Then**:
1. Respuesta de preflight aceptada: el método `POST` y las cabeceras `Authorization` y
   `Content-Type` permitidas; `Access-Control-Max-Age: 3600`; sin credenciales permitidas.

### FL-SEC-002: CORS — petición normal cross-origin

**Given**: P-SND; la SPA en un origen web permitido.
**When**: `admin` hace `GET /api/v1/sender-settings` desde ese origen.
**Then**:
1. `200` con la cabecera `Access-Control-Expose-Headers` que incluye `X-Correlation-Id`, y esa
   cabecera presente en la respuesta.

### FL-SEC-003: autorización por operación

**Given**: P-TPL; `m1` pedido por `billing`; `pepe@example.com` suprimida.
**When**: se llama a cada operación protegida sin credencial y con un usuario sin el permiso.
**Then**: sin credencial, todas responden `401 UNAUTHENTICATED`. Sin permiso, `403 ACCESS_DENIED`:
1. `configureSenderSettings`: `editor`, `operator`, `auditor`.
2. `getSenderSettings`: `editor`, `operator`.
3. `createTemplate`, `publishTemplateVersion`, `activateTemplateVersion`, `retireTemplate`:
   `operator`, `auditor`.
4. `getTemplate`, `listTemplates`, `getTemplateVersion`, `listTemplateVersions`: credencial de
   máquina del cliente `billing`.
5. `getMessage`, `listMessages`: `editor`.
6. `suppressAddress`, `releaseAddress`: `editor`, `auditor`.
7. `getSuppressedAddress`, `listSuppressedAddresses`: `editor`.
8. `purgeMessagePersonalData`: `operator`, `auditor`.
9. Con permiso: `auditor` lee `getSenderSettings`, `getTemplate`, `getMessage` y
   `listSuppressedAddresses` (`200`); `operator` suprime y libera (`200`).

## Lo que no tiene escenario

- **La guarda de `sendQueuedMessage` por sí sola** (`OBL-GUARD-UNOBSERVABLE`, aceptada en
  `decisions.yaml`): es interna y solo la alcanza el despacho. FL-DSP-040 cubre su efecto a través
  del barrido; su verificación propia es la comprobación de idempotencia de infraestructura del
  generador (dos ejecuciones concurrentes sobre el mismo mensaje producen un solo correo).
- **La ejecución programada de la purga**: su condición de entrada es el paso del tiempo. Se verifica
  por el disparo manual con `personalDataRetentionMonths: 0` (FL-PRG-001).
- **La aceptación tardía del relay tras el rescate** (el correo sale con el mensaje ya `failed`): no se
  puede provocar en caja negra sin controlar el reloj del relay; es un desajuste aceptado en el diseño.
