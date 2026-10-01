# notifications — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/notifications v0.1.1. Contrato de validación para la fase de generación.

## Convenciones de determinación

- **Rutas**: todas bajo `/api/v1`.
- **Ausencia vs nulo** (declarado en el manifiesto, `conventions.nulls: include`): un campo sin valor
  viaja como `null`, en respuestas y en eventos (`senderName: null`, `sentAt: null`,
  `failureReason: null`). Una colección vacía viaja como `[]`.
- **Fechas**: todo instante es ISO-8601 en UTC. Se verifica por forma y por relación
  (`sentAt` ≥ `sendingSince` ≥ `requestedAt`; un valor que «no cambia» es igual al leído antes),
  nunca por valor exacto.
- **Identificadores generados**: forma uuid, verificados por reutilización simbólica (`m1`, `v1`…),
  nunca por valor literal.
- **Mayúsculas** (declarado en el YAML): toda dirección de correo se normaliza a minúsculas antes de
  guardarla, compararla o deduplicarla; el filtro `recipient` de `listMessages` casa sin distinguir
  mayúsculas (`compare: ignore-case`). Los code de aplicación y plantilla ya son minúsculas por patrón.
- **Proyecciones** (derivadas del YAML; toda enumeración de este documento las respeta):
  - `Application`: `{ id, code, name, senderAddress, senderName, replyToAddress, status, createdAt,
    updatedAt, createdBy, updatedBy }`. Nunca `templates`.
  - `EmailTemplate`: `{ id, code, name, description, hasActiveVersion, applicationId, createdAt,
    updatedAt, createdBy, updatedBy }`. Nunca `versions`.
  - `EmailTemplateVersion` (lectura unitaria y publicación): `{ id, versionNumber, subject, htmlBody,
    textBody, variables, status, publishedAt, templateId, applicationId, createdAt, updatedAt,
    createdBy, updatedBy }`. En `listTemplateVersions` la misma sin `htmlBody` ni `textBody`.
    `variables` es una lista de `{ name, description, required }` en el orden en que se publicó.
  - `EmailMessage`: `{ id, idempotencyKey, recipients, renderedSubject, sender: { address, name,
    replyToAddress }, templateCode, templateVersionNumber, templateVersionId, status, failureReason,
    failureDetail, requestedAt, sendingSince, sentAt, failedAt, personalDataPurgedAt, applicationId }`.
    **Nunca** `variableValues` ni `contentFingerprint` (sensibles, excluidos de toda salida). No lleva
    campos de auditoría.
  - `SuppressedAddress`: `{ id, address, reason, notes, status, suppressedAt, releasedAt,
    applicationId, createdAt, updatedAt, createdBy, updatedBy }`.
- **Autoría**: `createdBy` y `updatedBy` son el identificador del principal autenticado que hizo la
  escritura: el `sub` de su token, sea de usuario o de la credencial de un cliente máquina (que no tiene por
  qué coincidir con el nombre del cliente). Las escrituras que no nacen de una
  petición (despacho, rechazo síncrono del relay) llevan el **centinela del sistema** que fija el
  generador. Los escenarios nombran a los usuarios por su rol.
- **Usuarios de prueba**: `admin` (rol `notifications-admin`), `auditor` (rol
  `notifications-auditor`), `editor` (rol `template-editor`, claim `applications: [billing]`) y
  `operator` (rol `application-operator`, claim `applications: [billing]`). Ni `editor` ni `operator`
  alcanzan `shipping`. **Requisito del arnés**: el claim `applications` de `editor` y `operator` se
  siembra con `[billing]`; no puede ser otro valor, porque la identidad HTTP es 1:1 con el cliente
  máquina `billing` y la aplicación de prueba tiene que llamarse igual.
- **Clientes máquina de prueba**: credencial de máquina del cliente `billing`, del cliente
  `shipping`, del cliente `dispatch-only` y del cliente `mail-provider`, todas con audiencia
  `notifications` salvo donde el escenario diga otra cosa.
- **Cuerpo de error**: forma fija del generador —`{timestamp, status, error, code, message,
  details}` más `correlationId`—. Los escenarios fijan solo `code` y status; `message` no es contrato.
- **Errores del mecanismo** (canónicos de `framework-errors.md`, aceptados en `decisions.yaml`):
  entrada fuera de cotas → `400 VALIDATION_ERROR`; sin credencial → `401 UNAUTHENTICATED`; sin
  permiso, sin scope o token de otra audiencia → `403 ACCESS_DENIED`; misma clave a la vez →
  `409 IDEMPOTENCY_KEY_IN_PROGRESS`; misma clave con otro contenido → `409 IDEMPOTENCY_KEY_REUSED`;
  escritura concurrente sobre la misma raíz → `409 CONCURRENT_MODIFICATION`.
- **Idempotencia**: en `requestNotification` la clave es el campo `idempotencyKey` del cuerpo (igual
  por HTTP y por evento), permanente y por aplicación. En `publishTemplateVersion` es la cabecera
  opcional `Idempotency-Key`, acotada a la plantilla, durante 24 h; sin cabecera se publica sin
  deduplicar.
- **Paginación** (sobre canónico): `{ items, page, size, totalElements, totalPages }`, `page` base 0,
  `size` 20 por defecto, tope 100 (un `size` mayor se recorta a 100). Página vacía o fuera de rango:
  `items: []`, `totalElements: 0` si no hay elementos, `totalPages: 0` si no hay elementos.
- **Correo**: se observa en el **buzón de pruebas** (el relay SMTP de pruebas captura lo que recibe).
  Un correo esperado se afirma por: `From` (dirección y nombre), `To` (lista exacta), `Reply-To`
  (presente o ausente), `Subject`, cuerpo `multipart/alternative` con una parte `text/html` y otra
  `text/plain`, y la cabecera `X-Notification-Id` con el id del mensaje. «No sale correo» es: el buzón
  no recibe ningún correo con ese `X-Notification-Id` en el plazo del escenario.
- **Despacho**: `dispatchQueuedMessages` corre cada minuto. «Pasa un ciclo del despacho» significa
  esperar a lo sumo **70 s**. Los escenarios no fuerzan el cron ni lo aceleran.
- **Relay de pruebas**: los escenarios que lo necesitan fijan su comportamiento en el `Given`
  (indisponible; rechaza de forma definitiva una dirección concreta). Cómo se programa es del arnés.
- **Parámetros** (manifiesto): `sendingTimeoutMinutes` vale `15` en pruebas y
  `personalDataRetentionMonths` vale `0`: en pruebas **todo mensaje terminado es purgable**.
- **Eventos**: envoltura Keel `{ metadata, data }` en el canal `notificationEvents`. Se afirma
  `metadata.eventType` y el `data` completo; `eventId` y `occurredAt` por forma. `EmailSent.data` =
  `{ messageId, applicationCode, idempotencyKey, templateCode, templateVersionNumber, sentAt }`;
  `EmailDeliveryFailed.data` = `{ messageId, applicationCode, idempotencyKey, templateCode,
  templateVersionNumber, failureReason, failedAt }`. Ninguno lleva destinatarios.

### Preparaciones

Cada flujo arranca desde el estado vacío y ejecuta por la API las preparaciones que su `Given` nombra.

- **P-APP** — `admin` registra dos aplicaciones con `registerApplication`:
  - `billing`: `{ "code": "billing", "name": "Facturación", "senderAddress": "Facturas@Notify.Example.com", "senderName": "Facturación Acme", "replyToAddress": "soporte@example.com" }`.
  - `shipping`: `{ "code": "shipping", "name": "Envíos", "senderAddress": "envios@notify.example.com" }`.
- **P-TPL** — P-APP y después `admin` crea en `billing` la plantilla `invoice-issued`
  (`{ "code": "invoice-issued", "name": "Factura emitida" }`) y publica su versión 1:
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
| registerApplication | FL-APP-001, FL-SEC-002 | usuarios |
| updateApplication | FL-APP-010, FL-APP-011, FL-SEC-002 | usuarios |
| suspendApplication | FL-APP-020, FL-DSP-010, FL-SEC-002 | usuarios |
| reactivateApplication | FL-APP-020, FL-SEC-002 | usuarios |
| getApplication | FL-APP-001, FL-APP-010, FL-SEC-002 | usuarios |
| listApplications | FL-APP-030, FL-SEC-002 | usuarios |
| createTemplate | FL-TPL-001, FL-TPL-002, FL-SEC-002 | usuarios |
| updateTemplate | FL-TPL-030, FL-SEC-002 | usuarios |
| publishTemplateVersion | FL-TPL-010, FL-TPL-011, FL-TPL-012, FL-SEC-002 | usuarios |
| activateTemplateVersion | FL-TPL-020, FL-SEC-002 | usuarios |
| retireTemplate | FL-TPL-020, FL-SEC-002 | usuarios |
| getTemplate | FL-TPL-001, FL-TPL-020, FL-SEC-002 | usuarios |
| listTemplates | FL-TPL-001, FL-SEC-002 | usuarios |
| getTemplateVersion | FL-TPL-010, FL-SEC-002 | usuarios |
| listTemplateVersions | FL-TPL-010, FL-SEC-002 | usuarios |
| requestNotification | FL-NTF-001, FL-NTF-002, FL-NTF-003, FL-NTF-010, FL-NTF-020, FL-NTF-030, FL-NTF-040 | **servidores (M2M)** |
| findMessageByIdempotencyKey | FL-NTF-001, FL-NTF-030 | **servidores (M2M)** |
| reportEmailBounce | FL-BNC-001, FL-BNC-002, FL-BNC-010 | **servidores (M2M)** |
| suppressHardBouncedAddress | FL-BNC-001, FL-BNC-002, FL-DSP-020, FL-DSP-021 | interna |
| dispatchQueuedMessages | FL-DSP-001, FL-DSP-040, FL-RSC-001, FL-RSC-002 | programada |
| sendQueuedMessage | FL-NTF-001, FL-DSP-001, FL-DSP-010, FL-DSP-011, FL-DSP-020, FL-DSP-021, FL-DSP-022, FL-DSP-030, FL-DSP-040 | interna |
| getMessage | FL-MSG-001, FL-SEC-002 | usuarios |
| listMessages | FL-MSG-001, FL-MSG-002, FL-SEC-002 | usuarios |
| suppressAddress | FL-SUP-001, FL-SUP-002, FL-SEC-002 | usuarios |
| releaseAddress | FL-SUP-001, FL-SEC-002 | usuarios |
| listSuppressedAddresses | FL-SUP-001, FL-SEC-002 | usuarios |
| purgeMessagePersonalData | FL-PRG-001, FL-SEC-002 | usuarios (+ programada) |

Transversales: suscripción `NotificationRequested` (FL-EVT-001, FL-EVT-002, FL-EVT-003) y CORS
(FL-SEC-001).

---

## Aplicaciones

### FL-APP-001: alta de aplicación, lectura y colisión de code

**Given**: el estado vacío. `admin` autenticado.

**When**: `registerApplication` — `POST /api/v1/applications` como `admin`
```json
{ "code": "billing", "name": "Facturación", "senderAddress": "Facturas@Notify.Example.com", "senderName": "Facturación Acme", "replyToAddress": "Soporte@Example.com" }
```

**Then**:
1. Status `200` (no `201`) y sin cabecera `Location` que afirmar.
2. El cuerpo es exactamente `{ id: a1, code: "billing", name: "Facturación", senderAddress:
   "facturas@notify.example.com", senderName: "Facturación Acme", replyToAddress:
   "soporte@example.com", status: "active", createdAt: <instante>, updatedAt: <instante>,
   createdBy: <id de admin>, updatedBy: <id de admin> }`: las direcciones llegan en minúsculas.
3. `getApplication` — `GET /api/v1/applications/billing` como `admin` → `200` con el mismo cuerpo.

**When**: `registerApplication` otra vez con `code: "billing"` y cualquier otro dato válido.

**Then**:
4. `409 APPLICATION_ALREADY_EXISTS`; `getApplication` de `billing` sigue devolviendo el cuerpo del paso 2.

**Orden de evaluación**:
1. Cotas del input (patrón de `code`, `name` 1..120, `EmailAddress`) → `400 VALIDATION_ERROR`.
2. Code libre → `409 APPLICATION_ALREADY_EXISTS`.

**Casos borde**:
- `code: "Billing"` (mayúscula), `code: "ab"` (corto) o `senderAddress: "no-es-correo"` → `400 VALIDATION_ERROR`.
- Sin `senderAddress` → `400 VALIDATION_ERROR`.
- Code duplicado **y** `name` vacío en la misma petición → `400 VALIDATION_ERROR`: la guarda 1 precede a la 2.
- `getApplication` de `nope-app` → `404 APPLICATION_NOT_FOUND`.
- Sin `senderName` ni `replyToAddress`: el cuerpo los trae como `null`.

### FL-APP-010: edición de una aplicación como sustitución completa

**Given**: P-APP.

**When**: `updateApplication` — `PUT /api/v1/applications/billing` como `admin`
```json
{ "name": "Facturación ES", "senderAddress": "facturas-es@notify.example.com" }
```

**Then**:
1. `200`; el cuerpo es la proyección `Application` con `code: "billing"`, `name: "Facturación ES"`,
   `senderAddress: "facturas-es@notify.example.com"`, `senderName: null`, `replyToAddress: null`,
   `status: "active"`, el mismo `id` y `createdAt` que antes, `updatedAt` ≥ `createdAt`,
   `updatedBy: <id de admin>`.
2. `getApplication` de `billing` devuelve ese mismo cuerpo.

**Casos borde**:
- `PUT /api/v1/applications/nope-app` con un cuerpo válido → `404 APPLICATION_NOT_FOUND`.
- Cuerpo sin `senderAddress` → `400 VALIDATION_ERROR` (el cuerpo es obligatorio).

### FL-APP-011: los mensajes aceptados conservan el remitente congelado

**Given**: P-TPL. `billing` pide R1 con la credencial de máquina del cliente `billing` y recibe `m1`
(`202`).

**When**: `updateApplication` de `billing` como `admin` con `{ "name": "Facturación", "senderAddress":
"otra@notify.example.com", "senderName": "Otro", "replyToAddress": "otro@example.com" }`, y pasa un
ciclo del despacho.

**Then**:
1. El buzón recibe un correo con `X-Notification-Id: m1` cuyo `From` es `"Facturación Acme"
   <facturas@notify.example.com>` y cuyo `Reply-To` es `soporte@example.com`: los datos congelados al
   aceptar, no los nuevos.
2. `getMessage` de `m1` como `admin` trae `sender: { address: "facturas@notify.example.com", name:
   "Facturación Acme", replyToAddress: "soporte@example.com" }`.

### FL-APP-020: suspender y reactivar una aplicación

**Given**: P-TPL.

**When**: `suspendApplication` — `POST /api/v1/applications/billing/suspend` como `admin`, sin cuerpo.

**Then**:
1. `200`; cuerpo `Application` de `billing` con `status: "suspended"`.
2. Repetir la suspensión → `409 INVALID_STATE_TRANSITION`; `getApplication` sigue `suspended` con el
   mismo `updatedAt`.
3. `requestNotification` con R1 y la credencial de máquina del cliente `billing` →
   `409 APPLICATION_SUSPENDED`; `findMessageByIdempotencyKey` de `inv-2026-0001` → `404 MESSAGE_NOT_FOUND`.

**When**: `reactivateApplication` — `POST /api/v1/applications/billing/reactivate` como `admin`.

**Then**:
4. `200`; `status: "active"`. Repetirlo → `409 INVALID_STATE_TRANSITION`.
5. `requestNotification` con R1 → `202` y `status: "queued"`.

**Orden de evaluación** (`requestNotification` sobre una aplicación suspendida):
1. Aplicación resuelta; 2. clave no usada; 3. suspensión → `409 APPLICATION_SUSPENDED`.

**Casos borde**:
- Suspendida `billing`, R1 con `templateCode: "no-existe"` → `409 APPLICATION_SUSPENDED`: la
  suspensión (paso 4) precede a la plantilla (paso 5).
- `suspendApplication` y `reactivateApplication` de `nope-app` → `404 APPLICATION_NOT_FOUND`.

### FL-APP-030: listado de aplicaciones y su alcance

**Given**: P-APP, y `admin` registra además `alpha-app`, `zeta-app` (senderAddress válidos) y suspende
`zeta-app`.

**When**: `listApplications` — `GET /api/v1/applications` como `admin`.

**Then**:
1. `200`; `items` son, en este orden, `alpha-app`, `billing`, `shipping`, `zeta-app` (code
   ascendente), cada uno con la proyección `Application`; `page: 0`, `size: 20`,
   `totalElements: 4`, `totalPages: 1`.
2. `?status=suspended` → solo `zeta-app`, `totalElements: 1`.
3. `?size=2&page=1` → `shipping`, `zeta-app`, `totalPages: 2`. `?page=5` → `items: []`,
   `page: 5`, `size: 20`. `?size=500` → `size: 100`.
4. Como `operator` (claim `[billing]`) → solo `billing`, `totalElements: 1`, sin `403`.
5. Como `auditor` → las cuatro.
6. `?status=active` como `admin` en un estado sin aplicaciones activas no aplica aquí; con
   `?status=suspended&page=0` como `operator` → `items: []`, `totalElements: 0`, `totalPages: 0`.

---

## Plantillas

### FL-TPL-001: alta de plantilla, lectura y listado

**Given**: P-APP.

**When**: `createTemplate` — `POST /api/v1/applications/billing/templates` como `editor`
```json
{ "code": "invoice-issued", "name": "Factura emitida", "description": "Se envía al emitir una factura" }
```

**Then**:
1. `200`; cuerpo exactamente `{ id: t1, code: "invoice-issued", name: "Factura emitida", description:
   "Se envía al emitir una factura", hasActiveVersion: false, applicationId: a1, createdAt,
   updatedAt, createdBy: <id de editor>, updatedBy: <id de editor> }`, donde `a1` es el id de `billing`.
2. `getTemplate` — `GET /api/v1/applications/billing/templates/invoice-issued` → `200`, mismo cuerpo.
3. `listTemplates` — `GET /api/v1/applications/billing/templates` → un elemento, `t1`.
   `listTemplates` de `shipping` como `admin` → `items: []`, `totalElements: 0`, `totalPages: 0`.
4. `listTemplateVersions` de `invoice-issued` → página vacía.

**Orden de evaluación**:
1. Cotas → `400 VALIDATION_ERROR`; 2. alcance → `403 APPLICATION_FORBIDDEN`; 3. aplicación →
   `404 APPLICATION_NOT_FOUND`; 4. code libre → `409 TEMPLATE_ALREADY_EXISTS`.

**Casos borde**:
- Repetir el alta de `invoice-issued` en `billing` → `409 TEMPLATE_ALREADY_EXISTS`. El mismo code en
  `shipping` como `admin` → `200` (el code es único dentro de la aplicación).
- `editor` crea en `shipping` → `403 APPLICATION_FORBIDDEN`, aunque el code ya exista allí.
- `admin` crea en `nope-app` → `404 APPLICATION_NOT_FOUND`; `listTemplates` de `nope-app` →
  `404 APPLICATION_NOT_FOUND`.
- `getTemplate` de `no-existe` en `billing` → `404 TEMPLATE_NOT_FOUND`.
- Se puede crear una plantilla con `billing` suspendida.

### FL-TPL-002: dos altas del mismo code a la vez

**Given**: P-APP.

**When**: `createTemplate` dos veces **a la vez** en `billing` como `admin`, las dos con
`{ "code": "welcome", "name": "Bienvenida" }`.

**Then**:
1. Una responde `200`; la otra responde `409` con `CONCURRENT_MODIFICATION` o
   `TEMPLATE_ALREADY_EXISTS`.
2. `listTemplates` de `billing` devuelve **exactamente una** plantilla `welcome`.
3. Si el perdedor recibió `CONCURRENT_MODIFICATION`, repetir su petición → `409 TEMPLATE_ALREADY_EXISTS`.

### FL-TPL-010: publicar versiones, consultarlas y reintentar con la misma clave

**Given**: P-APP; `editor` crea `invoice-issued` en `billing`.

**When**: `publishTemplateVersion` — `POST /api/v1/applications/billing/templates/invoice-issued/versions`
como `editor`, cabecera `Idempotency-Key: pub-1`, con el cuerpo de la versión 1 de P-TPL.

**Then**:
1. `200`; cuerpo `{ id: v1, versionNumber: 1, subject: "Factura {{invoiceNumber}}", htmlBody: …,
   textBody: …, variables: [las cuatro, en ese orden], status: "active", publishedAt: <instante>,
   templateId: t1, applicationId: a1, createdAt, updatedAt, createdBy: <id de editor>, updatedBy:
   <id de editor> }` con `htmlBody` y `textBody` idénticos a los enviados.
2. `getTemplate` → `hasActiveVersion: true`.
3. Repetir la misma petición con `Idempotency-Key: pub-1` → `200` con el mismo cuerpo (`v1`);
   `listTemplateVersions` sigue con `totalElements: 1`.
4. La misma clave `pub-1` con otro `subject` → `409 IDEMPOTENCY_KEY_REUSED`, sin versión nueva.

**When**: `publishTemplateVersion` sin cabecera, con `subject: "Tu factura {{invoiceNumber}}"` y el
resto igual.

**Then**:
5. `200`; `versionNumber: 2`, `status: "active"`.
6. `getTemplateVersion` — `GET …/versions/1` → `200` con `id: v1`, `versionNumber: 1`, el mismo `subject`,
   `htmlBody`, `textBody`, `variables`, `publishedAt` y `createdAt` que en el paso 1, `status: "archived"`,
   `updatedAt` ≥ el del paso 1 y `updatedBy: <id de editor>`.
7. `listTemplateVersions` — `GET …/versions` → `items` en orden `versionNumber` 2, 1, cada uno **sin**
   `htmlBody` ni `textBody`; `totalElements: 2`.
8. `getTemplate` → `hasActiveVersion: true`.

**Orden de evaluación**:
1. Alcance → `403 APPLICATION_FORBIDDEN`.
2. `Idempotency-Key`, si viene: contenido distinto → `409 IDEMPOTENCY_KEY_REUSED`; en curso →
   `409 IDEMPOTENCY_KEY_IN_PROGRESS`; repetición → la versión ya creada.
3. Plantilla → `404 TEMPLATE_NOT_FOUND`.
4. Variables declaradas sin repetir → `422 DUPLICATE_TEMPLATE_VARIABLE`.
5. Sintaxis → `422 INVALID_TEMPLATE_SYNTAX`.
6. Variables usadas declaradas → `422 UNDECLARED_TEMPLATE_VARIABLES`.

**Casos borde** (todos salvo el último fallan y no crean versión):
- `htmlBody: "<p>{{customerName</p>"` → `422 INVALID_TEMPLATE_SYNTAX`. También `"{{ }}"`, `"{{1abc}}"`
  y `"{{#if x}}"`.
- `textBody: "Hola {{desconocida}}"` → `422 UNDECLARED_TEMPLATE_VARIABLES`.
- Dos variables con `name: "amount"` → `422 DUPLICATE_TEMPLATE_VARIABLE`.
- Variables repetidas **y** un marcador mal formado → `422 DUPLICATE_TEMPLATE_VARIABLE` (paso 4 antes que 5).
- Con `Idempotency-Key: pub-1` (ya usada con otro contenido) y un marcador mal formado →
  `409 IDEMPOTENCY_KEY_REUSED` (paso 2 antes que 5).
- Plantilla `no-existe` → `404 TEMPLATE_NOT_FOUND`; `getTemplateVersion` de la versión 9 →
  `404 TEMPLATE_VERSION_NOT_FOUND`.
- Sin `textBody`, o `subject` de 301 caracteres, o 51 variables → `400 VALIDATION_ERROR`.
- Una variable declarada que no se usa se admite: `200` y crea la versión siguiente.

### FL-TPL-011: dos publicaciones con la misma clave a la vez

**Given**: P-APP; `admin` crea `invoice-issued` en `billing`.

**When**: `publishTemplateVersion` dos veces **a la vez**, las dos con `Idempotency-Key: pub-race` y el
cuerpo de la versión 1 de P-TPL.

**Then**:
1. Las dos responden `200` con el mismo cuerpo (`versionNumber: 1`), o una responde `200` y la otra
   `409` con `IDEMPOTENCY_KEY_IN_PROGRESS` o `CONCURRENT_MODIFICATION`.
2. `listTemplateVersions` devuelve **exactamente una** versión, `versionNumber: 1`, `active`.

### FL-TPL-012: dos publicaciones distintas a la vez no dejan dos versiones activas

**Given**: P-TPL (versión 1 activa).

**When**: dos `publishTemplateVersion` **a la vez** sin cabecera, con `subject` `"A {{invoiceNumber}}"`
y `"B {{invoiceNumber}}"` (resto como la versión 1).

**Then**:
1. Las dos responden `200`, o una `200` y la otra `409 CONCURRENT_MODIFICATION`.
2. `listTemplateVersions` tiene **exactamente una** versión `active`, la de mayor `versionNumber`;
   el resto `archived`, y los `versionNumber` son correlativos sin repetidos.

### FL-TPL-020: revertir, retirar y volver a poner en servicio

**Given**: P-TPL, y `editor` publica la versión 2 (`subject: "Tu factura {{invoiceNumber}}"`).

**When**: `activateTemplateVersion` — `POST …/templates/invoice-issued/versions/1/activate` como `editor`.

**Then**:
1. `200`; cuerpo `EmailTemplateVersion` con `id: v1`, `versionNumber: 1`, el contenido de la versión 1 de
   P-TPL, `status: "active"` y `updatedBy: <id de editor>`. `getTemplateVersion` de la 2 → `archived`.
2. Repetirlo → `409 INVALID_STATE_TRANSITION`; la versión 1 sigue `active` y la 2 `archived`.
3. `requestNotification` con R1 → `202`, `templateVersionNumber: 1`, `renderedSubject: "Factura F-0001"`.

**When**: `retireTemplate` — `POST …/templates/invoice-issued/retire` como `editor`.

**Then**:
4. `200`; cuerpo `EmailTemplate` con `hasActiveVersion: false`. `listTemplateVersions` → las dos `archived`.
5. Repetirlo → `200` sin cambios.
6. `requestNotification` con otra clave (`inv-2026-0002`) → `422 TEMPLATE_NOT_PUBLISHED`.

**When**: `activateTemplateVersion` de la versión 2.

**Then**:
7. `200`, `status: "active"`; `getTemplate` → `hasActiveVersion: true`; la petición `inv-2026-0002`
   ahora → `202` con `templateVersionNumber: 2`.

**Casos borde**:
- `activateTemplateVersion` de la versión 7 → `404 TEMPLATE_VERSION_NOT_FOUND`.
- `retireTemplate` y `activateTemplateVersion` sobre `no-existe` → `404 TEMPLATE_NOT_FOUND`.

### FL-TPL-030: editar nombre y descripción de una plantilla

**Given**: P-TPL.

**When**: `updateTemplate` — `PUT /api/v1/applications/billing/templates/invoice-issued` como `editor`
con `{ "name": "Factura" }`.

**Then**:
1. `200`; cuerpo `EmailTemplate` con `name: "Factura"`, `description: null`, `hasActiveVersion: true`,
   `updatedBy: <id de editor>`.
2. `listTemplateVersions` sigue con una versión: editar no crea versiones.

**Casos borde**:
- `PUT …/templates/no-existe` → `404 TEMPLATE_NOT_FOUND`.
- `editor` sobre `shipping` → `403 APPLICATION_FORBIDDEN`.

---

## Envíos servidor a servidor

### FL-NTF-001: pedir un correo, recibirlo y consultarlo

**Given**: P-TPL. Canal `notificationEvents` vacío.

**When**: `requestNotification` — `POST /api/v1/notifications` con la credencial de máquina del
cliente `billing` y el cuerpo R1.

**Then**:
1. `202`; cuerpo exactamente la proyección `EmailMessage`: `{ id: m1, idempotencyKey:
   "inv-2026-0001", recipients: ["ana@example.com", "luis@example.com"], renderedSubject: "Factura
   F-0001", sender: { address: "facturas@notify.example.com", name: "Facturación Acme",
   replyToAddress: "soporte@example.com" }, templateCode: "invoice-issued", templateVersionNumber: 1,
   templateVersionId: v1, status: "queued", failureReason: null, failureDetail: null, requestedAt:
   <instante>, sendingSince: null, sentAt: null, failedAt: null, personalDataPurgedAt: null,
   applicationId: a1 }`.
2. El cuerpo **no** trae `variableValues` ni `contentFingerprint`.

**When**: pasa un ciclo del despacho.

**Then**:
3. El buzón recibe **exactamente un** correo con `X-Notification-Id: m1`: `From` `"Facturación Acme"
   <facturas@notify.example.com>`, `To` `ana@example.com, luis@example.com` (los dos en To),
   `Reply-To` `soporte@example.com`, `Subject` `Factura F-0001`; parte HTML
   `<p>Hola Ana, total 10,00 €.</p><p></p>` y parte texto `Hola Ana, total 10,00 €. ` (la variable
   opcional `note` ausente queda vacía).
4. `findMessageByIdempotencyKey` — `GET /api/v1/notifications/billing/inv-2026-0001` con la
   credencial de máquina del cliente `billing` → `200`, el cuerpo del paso 1 con `status: "sent"`,
   `sendingSince` y `sentAt` presentes (`sentAt` ≥ `sendingSince` ≥ `requestedAt`).
5. El canal `notificationEvents` recibe **exactamente un** `EmailSent` con `data: { messageId: m1,
   applicationCode: "billing", idempotencyKey: "inv-2026-0001", templateCode: "invoice-issued",
   templateVersionNumber: 1, sentAt: <el del paso 4> }`.

**Notas de determinación**: el orden de `recipients` es el de su primera aparición tras normalizar.

### FL-NTF-002: repetir la misma petición no manda un segundo correo

**Given**: P-TPL. `billing` pide R1 (`202`, `m1`) y pasa un ciclo del despacho (sale un correo).

**When**: `requestNotification` con la misma `idempotencyKey` y el mismo contenido escrito de otra
forma: `recipients: ["LUIS@example.com", "ana@example.com", "ana@EXAMPLE.com"]`, las variables en otro
orden y además `{ "name": "extra", "value": "x" }` (no declarada).

**Then**:
1. `202` con el cuerpo de `m1` en su estado actual (`status: "sent"`), mismo `id`.
2. El buzón **no** recibe un segundo correo con `X-Notification-Id: m1` ni ningún otro en el
   siguiente ciclo del despacho.
3. `listMessages` de `billing` como `admin` → `totalElements: 1`.

**When**: la misma clave con `recipients: ["ana@example.com"]`.

**Then**:
4. `409 IDEMPOTENCY_KEY_REUSED`; no se crea nada ni sale correo.

**Casos borde**:
- La misma clave con `amount: "11,00 €"` → `409 IDEMPOTENCY_KEY_REUSED`.
- `admin` crea y publica en `shipping` la plantilla `invoice-issued` (versión 1 de P-TPL); `shipping` pide
  R1 tal cual con su credencial → `202` con un mensaje nuevo (otro `id`, `applicationId` de `shipping`):
  la clave es por aplicación.
- Una clave nueva con el mismo contenido que R1 → `202` con un mensaje nuevo `m2` y un segundo correo:
  la clave, no el contenido, identifica el envío.

### FL-NTF-003: dos peticiones con la misma clave a la vez

**Given**: P-TPL.

**When**: `requestNotification` dos veces **a la vez** con la credencial de máquina del cliente
`billing`, las dos con R1 (`idempotencyKey: "race-0001"`).

**Then**:
1. Las dos responden `202` con el mismo `id`, o una `202` y la otra `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `listMessages` de `billing` como `admin` devuelve **exactamente un** mensaje con esa clave.
3. Tras un ciclo del despacho, el buzón recibe **exactamente un** correo para ese mensaje.

### FL-NTF-010: rechazos de la petición y su precedencia

**Given**: P-TPL, y `admin` suprime `vetado@example.com` en `billing` (`suppressAddress`, reason `manual`).

**When**: cada petición siguiente con la credencial de máquina del cliente `billing` (salvo la primera),
todas con claves nuevas.

**Then** (salvo la segunda mitad del punto 5, ninguna crea mensaje ni hace salir correo):
1. Con la credencial de máquina del cliente `dispatch-only` (no hay aplicación `dispatch-only`
   registrada) y R1 → `422 APPLICATION_NOT_FOUND`.
2. R1 con `variables` repitiendo `invoiceNumber` → `422 DUPLICATE_VARIABLE_VALUE`.
3. R1 con `templateCode: "no-existe"` → `422 TEMPLATE_NOT_FOUND`.
4. `admin` crea `draft-only` sin versiones; R1 con `templateCode: "draft-only"` →
   `422 TEMPLATE_NOT_PUBLISHED`.
5. R1 sin `amount` → `422 MISSING_TEMPLATE_VARIABLES`. En cambio, sin `note` (opcional) → `202`: esa sí
   crea un mensaje.
6. R1 con `recipients: ["ana@example.com", "Vetado@Example.com"]` → `422 RECIPIENT_SUPPRESSED`; no
   sale correo tampoco a `ana@example.com`.
7. R1 con `invoiceNumber` de 1990 caracteres → `422 RENDERED_SUBJECT_TOO_LONG`.

**Orden de evaluación**:
1. Aplicación del llamante → `422 APPLICATION_NOT_FOUND`.
2. Variables sin repetir → `422 DUPLICATE_VARIABLE_VALUE`.
3. Repetición de la clave → el mensaje, o `409 IDEMPOTENCY_KEY_REUSED` / `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
4. Suspensión → `409 APPLICATION_SUSPENDED`.
5. Plantilla → `422 TEMPLATE_NOT_FOUND`.
6. Versión activa → `422 TEMPLATE_NOT_PUBLISHED`.
7. Variables required → `422 MISSING_TEMPLATE_VARIABLES`.
8. Supresiones → `422 RECIPIENT_SUPPRESSED`.
9. Asunto renderizado ≤ 998 → `422 RENDERED_SUBJECT_TOO_LONG`.

**Casos borde**:
- `templateCode: "no-existe"` **y** un destinatario suprimido → `422 TEMPLATE_NOT_FOUND` (5 antes que 8).
- Variables repetidas **y** plantilla inexistente → `422 DUPLICATE_VARIABLE_VALUE` (2 antes que 5).
- `recipients: []`, 21 destinatarios, `idempotencyKey: "corta"` (5 caracteres) o `"con espacio 123"`
  → `400 VALIDATION_ERROR`.
- En el asunto, un valor con salto de línea (`"F-1\nBcc: x@y.z"`) se renderiza como `"Factura F-1 Bcc: x@y.z"`
  y el correo no lleva cabecera `Bcc`.

### FL-NTF-020: la repetición se honra aunque el mundo haya cambiado

**Given**: P-TPL. `billing` pide R1 (`m1`) y pasa un ciclo del despacho.

**When**: `admin` retira `invoice-issued` y suspende `billing`; `billing` repite R1 tal cual.

**Then**:
1. `202` con el cuerpo de `m1` (`status: "sent"`): la repetición no evalúa suspensión ni plantilla.
2. No sale ningún correo nuevo.
3. Con otra clave → `409 APPLICATION_SUSPENDED`.

### FL-NTF-030: la superficie M2M y sus credenciales

**Given**: P-TPL. `billing` pide R1 (`m1`). `admin` registra la aplicación `dispatch-only`
(`senderAddress: "ops@notify.example.com"`) y le crea y publica `invoice-issued` (versión 1 de P-TPL).

**Then**:
1. `POST /api/v1/notifications` sin credencial → `401 UNAUTHENTICATED`.
2. Con la credencial de máquina del cliente `mail-provider` (sin `notification:send`) →
   `403 ACCESS_DENIED`.
3. Con la credencial de máquina del cliente `billing` emitida para otra audiencia → `403 ACCESS_DENIED`.
4. Con el token de `admin` (usuario, sin el scope) → `403 ACCESS_DENIED`.
5. Con la credencial de máquina del cliente `dispatch-only` y R1 → `202`, mensaje en la aplicación
   `dispatch-only`, `sender.address: "ops@notify.example.com"`.
6. `findMessageByIdempotencyKey` de `dispatch-only/inv-2026-0001` con la credencial de máquina del
   cliente `dispatch-only` (sin `notification:read`) → `403 ACCESS_DENIED`.
7. `GET /api/v1/notifications/billing/inv-2026-0001` con la credencial de máquina del cliente
   `shipping` → `403 APPLICATION_FORBIDDEN`: no es su aplicación.
8. `GET /api/v1/notifications/billing/no-existe-0001` con la credencial de máquina del cliente
   `billing` → `404 MESSAGE_NOT_FOUND`.
9. `GET …/notifications/billing/inv-2026-0001` con la credencial de máquina del cliente `billing` →
   `200` con la proyección `EmailMessage` de `m1`, sin `variableValues` ni `contentFingerprint`.

### FL-NTF-040: el coste de comprobar supresiones no crece con los destinatarios

**Given**: P-TPL.

**When**: `requestNotification` con 2 destinatarios, y otra (clave distinta) con 20 destinatarios.

**Then**:
1. Las dos `202`.
2. El trabajo de la operación contra el almacén para comprobar supresiones es el mismo en las dos:
   veinte destinatarios no cuestan más que dos (una comprobación por lote, no una por destinatario).

---

## Eventos entrantes

### FL-EVT-001: pedir un correo por el canal de eventos

**Given**: P-TPL. Canal `notificationRequests` sin mensajes.

**When**: llega `NotificationRequested` por `notificationRequests`, con envoltura Keel,
`metadata.source: "billing"`, `metadata.eventId: e1` y `data` = R1 con `idempotencyKey: "evt-0001"`.

**Then**:
1. `findMessageByIdempotencyKey` de `billing/evt-0001` → `200`, mensaje `m1` en `billing`, con la misma
   proyección que produciría la petición HTTP.
2. Tras un ciclo del despacho, el buzón recibe **exactamente un** correo con `X-Notification-Id: m1`.

**When**: se **reentrega** el mismo mensaje (mismo `metadata.eventId: e1`).

**Then**:
3. No se crea ningún mensaje nuevo (`listMessages` de `billing` → `totalElements: 1`) y no sale un
   segundo correo.

**When**: llega otro `NotificationRequested` con `eventId: e2` y el mismo `data`.

**Then**:
4. Tampoco hay segundo mensaje ni segundo correo: la guarda es la `idempotencyKey`, no solo el `eventId`.

### FL-EVT-002: lo que no se puede aceptar acaba en la deadLetter

**Given**: P-TPL.

**When**: llega `NotificationRequested` con `metadata.source: "desconocida"` y R1 como `data`.

**Then**:
1. El mensaje acaba en la deadLetter: no se crea nada ni sale correo.

**When**: llega `NotificationRequested` de `billing` con `templateCode: "no-existe"` (rechazo de negocio
`TEMPLATE_NOT_FOUND`).

**Then**:
2. El mensaje acaba en la deadLetter.
3. `findMessageByIdempotencyKey` de esa clave → `404 MESSAGE_NOT_FOUND`; no sale correo.

**Casos borde**:
- `data` con `recipients: []` → deadLetter tras los intentos; nada creado.

### FL-EVT-003: la misma clave por las dos puertas es el mismo envío

**Given**: P-TPL. `billing` pide R1 por HTTP (`202`, `m1`).

**When**: llega `NotificationRequested` de `billing` con el mismo `data` que R1.

**Then**:
1. No se crea un segundo mensaje; tras un ciclo del despacho, sale **exactamente un** correo para `m1`.
2. Si el `data` difiere (otro destinatario), el evento acaba en la deadLetter tras los intentos
   (`IDEMPOTENCY_KEY_REUSED`) y no sale nada más.

---

## Despacho

### FL-DSP-001: el despacho vacía la cola

**Given**: P-TPL. `billing` pide R1 tres veces con `idempotencyKey` `queue-0001`, `queue-0002` y `queue-0003` (mensajes `m1`, `m2`, `m3`), en ese orden.

**When**: pasa un ciclo del despacho.

**Then**:
1. Los tres quedan `sent` (leídos con `getMessage` como `admin`) y el buzón recibe exactamente un
   correo por cada uno.
2. El canal `notificationEvents` recibe tres `EmailSent`, uno por mensaje.

### FL-DSP-010: una aplicación suspendida no deja salir su cola

**Given**: P-TPL. `billing` pide R1 (`m1`, `queued`) e inmediatamente `admin` suspende `billing`,
antes de que pase el despacho.

**When**: pasa un ciclo del despacho.

**Then**:
1. `getMessage` de `m1` → `status: "failed"`, `failureReason: "application-suspended"`, `failedAt`
   presente, `sentAt: null`.
2. El buzón no recibe ningún correo con `X-Notification-Id: m1`.
3. `notificationEvents` recibe un `EmailDeliveryFailed` con `failureReason: "application-suspended"` y
   `failedAt` igual al de `getMessage`.
4. Reactivar `billing` no reenvía `m1`.

### FL-DSP-011: una dirección suprimida después de aceptar no recibe el correo

**Given**: P-TPL. `billing` pide R1 (`m1`) e inmediatamente `admin` suprime `luis@example.com`.

**When**: pasa un ciclo del despacho.

**Then**:
1. `m1` → `failed` con `failureReason: "recipient-suppressed"`; no sale correo a nadie.
2. `EmailDeliveryFailed` con ese motivo.

### FL-DSP-020: el relay rechaza a todos los destinatarios

**Given**: P-TPL. El relay de pruebas rechaza de forma definitiva `ana@example.com` y `luis@example.com`.
`billing` pide R1 (`m1`).

**When**: pasa un ciclo del despacho.

**Then**:
1. `m1` → `failed`, `failureReason: "delivery-rejected"`, `failureDetail` no nulo, `failedAt` presente.
2. `EmailDeliveryFailed` con `failureReason: "delivery-rejected"`.
3. `listSuppressedAddresses` de `billing` como `admin` → las dos direcciones, `status: "active"`,
   `reason: "hard-bounce"`, `createdBy` el centinela del sistema.
4. Una petición nueva con esas direcciones → `422 RECIPIENT_SUPPRESSED`.

### FL-DSP-021: el relay rechaza a uno y acepta al otro

**Given**: P-TPL. El relay rechaza de forma definitiva solo `luis@example.com`. `billing` pide R1 (`m1`).

**When**: pasa un ciclo del despacho.

**Then**:
1. `m1` → `sent`, con `sentAt`; `EmailSent` publicado.
2. El buzón recibe el correo para `ana@example.com`.
3. `listSuppressedAddresses` de `billing` → solo `luis@example.com`, `reason: "hard-bounce"`.

### FL-DSP-022: con el relay caído el mensaje falla y no se reintenta

**Given**: P-TPL. El relay de pruebas indisponible. `billing` pide R1 (`m1`).

**When**: pasa un ciclo del despacho; después el relay vuelve y pasan dos ciclos más.

**Then**:
1. `m1` → `failed`, `failureReason: "delivery-error"`; `EmailDeliveryFailed` con ese motivo.
2. Con el relay de vuelta, **no** sale ningún correo para `m1`: el servicio no reintenta.
3. No se suprime ninguna dirección.

### FL-DSP-030: render de los cuerpos y escapado

**Given**: P-TPL.

**When**: `billing` pide R1 con `customerName: "Ana <b>&</b> \"O'Hara\""` y `note: "Gracias"`, y pasa un
ciclo del despacho.

**Then**:
1. La parte HTML es `<p>Hola Ana &lt;b&gt;&amp;&lt;/b&gt; &quot;O&#39;Hara&quot;, total 10,00 €.</p><p>Gracias</p>`
   (se escapan `& < > " '`).
2. La parte texto es `Hola Ana <b>&</b> "O'Hara", total 10,00 €. Gracias` (sin escapar).
3. El `Subject` es `Factura F-0001` (el asunto no se escapa).

**Notas de determinación**: las sustituciones son exactamente las cinco de la regla de `sendQueuedMessage`
(`&amp;`, `&lt;`, `&gt;`, `&quot;`, `&#39;`).

### FL-DSP-040: dos réplicas del despacho no mandan dos correos

**Given**: dos instancias del servicio vivas. P-TPL. `billing` pide R1 con `idempotencyKey`
`concurrent-0006` (mensaje de control `c-6`) y se espera a un ciclo, que lo deja `sent`; después pide R1 con
`concurrent-0001` … `concurrent-0005` (mensajes `c-1` … `c-5`, en `queued`).

**When**: pasan dos ciclos de `dispatchQueuedMessages` en las dos réplicas a la vez.

**Then**:
1. El buzón recibe **exactamente un** correo por cada uno de `c-1` … `c-5` y ninguno más para `c-6`.
2. Los cinco quedan `sent`; `notificationEvents` recibe exactamente un `EmailSent` por mensaje.

**Notas de determinación**: la réplica que pierde la transición `queued → sending` de un mensaje
termina en `MESSAGE_NOT_QUEUED` para él, sin contactar con el relay; no es observable salvo por la
ausencia del segundo correo.

### FL-RSC-001: se rescata el mensaje que quedó a medias en sending

**Given**: P-TPL. `billing` pide R1 (`m1`). Antes del despacho, el **arnés** pone `m1` en `sending` con
`sendingSince` más antiguo que `sendingTimeoutMinutes` (15) —ninguna operación pública deja ese estado: es
el exacto en que queda una réplica que murió con él en la mano—. No se baja el umbral ni se espera.

**When**: pasa un ciclo de `dispatchQueuedMessages`.

**Then**:
1. `m1` → `failed`, `failureReason: "sending-timeout"`, `failedAt` presente, `sentAt: null`.
2. **No** sale ningún correo para `m1`: el rescate nunca reenvía.
3. `notificationEvents` recibe exactamente un `EmailDeliveryFailed` con `failureReason: "sending-timeout"`.
4. Un segundo ciclo no cambia nada ni publica otro evento.
5. Ningún mensaje en `sending` tiene `sendingSince` nulo.

### FL-RSC-002: lo que acaba de entrar en sending no se toca

**Given**: P-TPL. `billing` pide R1 (`m1`). Antes del despacho, el **arnés** pone `m1` en `sending` con
`sendingSince` = ahora.

**When**: pasan dos ciclos de `dispatchQueuedMessages`.

**Then**:
1. `m1` sigue en `sending`, con el mismo `sendingSince`; no se ha publicado ningún evento para él y no
   ha salido correo.

---

## Rebotes

### FL-BNC-001: un rebote permanente suprime la dirección

**Given**: P-TPL. `billing` pide R1 (`m1`) y pasa un ciclo del despacho (`sent`). `admin` había
suprimido y después liberado `ana@example.com` en `billing`.

**When**: `reportEmailBounce` — `POST /api/v1/messages/{m1}/bounces` con la credencial de máquina del
cliente `mail-provider`:
```json
{ "recipient": "Ana@Example.com", "bounceType": "permanent", "diagnostic": "550 mailbox unavailable" }
```

**Then**:
1. `204` sin cuerpo.
2. `listSuppressedAddresses` de `billing` como `admin` → `ana@example.com` con `status: "active"`,
   `reason: "hard-bounce"`, `notes: "550 mailbox unavailable"`, `releasedAt: null`, `updatedBy` el `sub` de la
   credencial de máquina del cliente `mail-provider`: se reactiva aunque un humano la hubiera liberado.
3. Repetir el mismo reporte → `204` y la supresión con los mismos `reason`, `notes`, `status` y
   `suppressedAt`.
4. `requestNotification` nueva a `ana@example.com` → `422 RECIPIENT_SUPPRESSED`.

**When**: `reportEmailBounce` de `luis@example.com` con `bounceType: "transient"`.

**Then**:
5. `204`; `luis@example.com` no aparece en las supresiones.

**Orden de evaluación**:
1. Mensaje → `404 MESSAGE_NOT_FOUND`; 2. salió de queued → `409 MESSAGE_NOT_SENT`;
   3. destinatario del mensaje → `422 BOUNCE_RECIPIENT_UNKNOWN`.

**Casos borde**:
- Un uuid que no es de ningún mensaje → `404 MESSAGE_NOT_FOUND`.
- `recipient: "otro@example.com"` sobre `m1` → `422 BOUNCE_RECIPIENT_UNKNOWN`.
- Sobre un mensaje recién aceptado y aún `queued` (el reporte se hace inmediatamente tras el `202`,
  antes del siguiente ciclo) → `409 MESSAGE_NOT_SENT`.
- Sobre un mensaje `failed` por `delivery-error` → `204` y se suprime.
- `bounceType: "soft"` → `400 VALIDATION_ERROR`.

### FL-BNC-002: dos rebotes de la misma dirección a la vez

**Given**: P-TPL. `m1` enviado a `ana@example.com`; la dirección nunca se suprimió.

**When**: dos `reportEmailBounce` permanentes de `ana@example.com` **a la vez**.

**Then**:
1. Las dos `204`, o una `204` y la otra `409 APPLICATION_ADDRESS_ALREADY_EXISTS`.
2. `listSuppressedAddresses` de `billing` → **exactamente una** supresión de `ana@example.com`.

### FL-BNC-010: solo el proveedor reporta rebotes

**Given**: P-TPL; `m1` enviado.

**Then**:
1. `reportEmailBounce` sin credencial → `401 UNAUTHENTICATED`.
2. Con la credencial de máquina del cliente `billing` → `403 ACCESS_DENIED`.
3. Con el token de `admin` → `403 ACCESS_DENIED`.

---

## Supresiones

### FL-SUP-001: suprimir, listar, liberar y volver a suprimir

**Given**: P-APP.

**When**: `suppressAddress` — `PUT /api/v1/applications/billing/suppressed-addresses/Queja@Example.com`
como `operator`, con `{ "reason": "complaint", "notes": "Pidió no recibir más" }`.

**Then**:
1. `200`; cuerpo `{ id: s1, address: "queja@example.com", reason: "complaint", notes: "Pidió no
   recibir más", status: "active", suppressedAt: <instante>, releasedAt: null, applicationId: a1,
   createdAt, updatedAt, createdBy: <id de operator>, updatedBy: <id de operator> }`.
2. Repetirlo con `reason: "manual"` → `200` con los mismos `id`, `address`, `reason: "complaint"`,
   `notes`, `status`, `suppressedAt` y `releasedAt` del paso 1: no-op.
3. `admin` suprime también `b@example.com` y `c@example.com`, en ese orden.
   `listSuppressedAddresses` — `GET /api/v1/applications/billing/suppressed-addresses` → `c`, `b`,
   `queja` (`suppressedAt` descendente); `?size=1&page=1` → `b`; `?page=9` → `items: []`.

**When**: `releaseAddress` — `POST …/suppressed-addresses/queja@example.com/release` como `operator`.

**Then**:
4. `200`; `status: "released"`, `releasedAt` presente, `reason` y `notes` conservados.
5. `listSuppressedAddresses` ya no la incluye (`totalElements: 2`).
6. Repetir la liberación → `404 SUPPRESSED_ADDRESS_NOT_FOUND`.

**When**: `suppressAddress` de `queja@example.com` con `{ "reason": "manual" }`.

**Then**:
7. `200`; mismo `id: s1`, `status: "active"`, `reason: "manual"`, `notes: null`, `releasedAt: null`,
   `suppressedAt` posterior al del paso 1.

**Casos borde**:
- `reason: "hard-bounce"` → `400 VALIDATION_ERROR`.
- Dirección no suprimida nunca → liberar da `404 SUPPRESSED_ADDRESS_NOT_FOUND`.
- `suppressAddress` y `listSuppressedAddresses` en `nope-app` como `admin` → `404 APPLICATION_NOT_FOUND`.
- `operator` en `shipping` → `403 APPLICATION_FORBIDDEN` en las tres operaciones.
- `listSuppressedAddresses` de `shipping` como `admin` → `items: []`, `totalElements: 0`, `totalPages: 0`.

### FL-SUP-002: dos supresiones de la misma dirección a la vez

**Given**: P-APP.

**When**: dos `suppressAddress` de `x@example.com` en `billing` **a la vez**, como `admin`.

**Then**:
1. Las dos `200`, o una `200` y la otra `409 APPLICATION_ADDRESS_ALREADY_EXISTS`.
2. `listSuppressedAddresses` → **exactamente una** supresión de `x@example.com`.

---

## Consulta de correos

### FL-MSG-001: leer y listar correos con filtros

**Given**: P-TPL; `admin` crea y publica en `shipping` la plantilla `shipped` (subject `"Enviado"`, sin
variables). `billing` pide `b-1` (R1 con `idempotencyKey: "msg-b-0001"`), `b-2` (R1 con
`idempotencyKey: "msg-b-0002"` y `recipients: ["pepe@example.com"]`) y `b-3` (R1 con
`idempotencyKey: "msg-b-0003"` y `templateCode` inexistente → rechazado, no cuenta); `shipping` pide `s-1`
(`idempotencyKey: "msg-s-0001"`, plantilla `shipped`, `recipients: ["eva@example.com"]`, sin variables).
Pasa un ciclo del despacho.

**When**: `listMessages` — `GET /api/v1/messages` como `admin`.

**Then**:
1. `200`; `items` = `s-1`, `b-2`, `b-1` (`requestedAt` descendente), cada uno con la proyección
   `EmailMessage` sin `variableValues` ni `contentFingerprint`; `totalElements: 3`.
2. `?applicationCode=billing` → `b-2`, `b-1`. `?templateCode=shipped` → `s-1`.
   `?status=sent&applicationCode=shipping` → `s-1`. `?status=queued` → `items: []`.
3. `?recipient=ANA@example.com` → solo `b-1`.
4. `?requestedFrom=<requestedAt de b-2>&requestedTo=<requestedAt de s-1>` → solo `b-2` (desde
   inclusivo, hasta exclusivo).
5. `?size=1` → `s-1`, `totalPages: 3`; `?page=3&size=1` → `items: []`.
6. `?requestedFrom=X&requestedTo=X` (iguales) → `400 INVALID_TIME_WINDOW`.

**When**: consultas como `operator` (claim `[billing]`).

**Then**:
7. `listMessages` sin filtros → `b-2`, `b-1`: nunca `s-1`.
8. `?applicationCode=shipping` → `403 APPLICATION_FORBIDDEN`.
9. `getMessage` — `GET /api/v1/messages/{b-1}` → `200` con su proyección; `getMessage` de `s-1` →
   `404 MESSAGE_NOT_FOUND` (fuera de alcance se ve igual que inexistente).
10. `getMessage` de un uuid inexistente como `admin` → `404 MESSAGE_NOT_FOUND`.
11. Como `auditor` → los tres mensajes.

### FL-MSG-002: el listado de correos no degrada con el volumen

**Given**: P-TPL; `billing` pide 25 mensajes y pasa un ciclo del despacho.

**When**: `listMessages` como `admin` con `size=2` y con `size=20`.

**Then**:
1. Las dos respuestas `200`, en orden `requestedAt` descendente, `totalElements: 25`.
2. El trabajo contra el almacén de las dos es el mismo: veinte elementos no cuestan más que dos.

---

## Purga de datos personales

### FL-PRG-001: purga manual y repetición tras la purga

**Given**: P-TPL (`personalDataRetentionMonths: 0` en pruebas). `billing` pide R1 (`m1`) y pasa un
ciclo del despacho (`sent`).

**When**: `purgeMessagePersonalData` — `POST /api/v1/messages/personal-data-purges` como `admin`, sin cuerpo.

**Then**:
1. `200` con `{ purgedCount: 1 }`.
2. `getMessage` de `m1` → `recipients: []`, `renderedSubject: null`, `personalDataPurgedAt` presente;
   `status: "sent"`, `idempotencyKey`, `sender`, `templateCode` y `sentAt` sin cambios.
3. Repetir la purga → `{ purgedCount: 0 }`.
4. `requestNotification` con R1 otra vez → `202` con el cuerpo purgado de `m1`: la huella sobrevive
   a la purga. Con `amount` distinto → `409 IDEMPOTENCY_KEY_REUSED`.
5. `reportEmailBounce` de `ana@example.com` sobre `m1` → `422 BOUNCE_RECIPIENT_UNKNOWN`.
6. `listMessages?recipient=ana@example.com` → `items: []`.
7. Un mensaje aún `queued` al purgar no se purga (se pide `m2` y se purga de inmediato, antes del
   despacho: `purgedCount` no lo cuenta y `getMessage` de `m2` conserva `recipients`).

**Casos borde**:
- `operator` o `auditor` → `403 ACCESS_DENIED`.

---

## Transversal

### FL-SEC-001: CORS desde el navegador

**Given**: P-APP.

**When**: preflight `OPTIONS /api/v1/applications/billing/templates` **sin credencial**, desde un origen
web permitido, pidiendo método `POST` y cabeceras `Authorization, Content-Type, Idempotency-Key`.

**Then**:
1. Preflight aceptado: permite `POST` y las tres cabeceras, con `max-age` 3600; no responde `401`.

**When**: `GET /api/v1/applications` como `admin` desde ese origen (petición normal cross-origin).

**Then**:
2. `200`, y la respuesta expone al navegador `X-Correlation-Id`.

### FL-SEC-002: autorización por operación

**Given**: P-TPL; `admin` crea y publica en `shipping` la plantilla `shipped` (versión 1, subject
`"Enviado"`, sin variables); `m1` pedido por `billing` y enviado; `x@example.com` suprimida en `billing`.

**Then** (cada fila: sin credencial → `401 UNAUTHENTICATED`; con el rol indicado, que no tiene el
permiso → `403 ACCESS_DENIED`; con el rol permitido y fuera de su alcance, donde aplica →
`403 APPLICATION_FORBIDDEN`):
1. `registerApplication`, `updateApplication`, `suspendApplication`, `reactivateApplication`: sin
   permiso `editor`, `operator` y `auditor`.
2. `getApplication`: sin credencial `401`; `operator` sobre `shipping` → `403 APPLICATION_FORBIDDEN`;
   `auditor` sobre `shipping` → `200`.
3. `listApplications`: sin credencial `401`.
4. `createTemplate`, `updateTemplate`, `publishTemplateVersion`, `activateTemplateVersion`,
   `retireTemplate`: sin permiso `operator` y `auditor`; `editor` sobre `shipping` →
   `403 APPLICATION_FORBIDDEN`.
5. `getTemplate`, `listTemplates`, `getTemplateVersion`, `listTemplateVersions`: sin credencial `401`;
   `operator` sobre `shipping` → `403 APPLICATION_FORBIDDEN`; `auditor` sobre `shipping/shipped` (y su
   versión 1) → `200`.
6. `getMessage`, `listMessages`: sin permiso `editor` (`403 ACCESS_DENIED`).
7. `suppressAddress`, `releaseAddress`: sin permiso `editor` y `auditor`.
8. `listSuppressedAddresses`: sin permiso `editor`; `auditor` → `200`.
9. `purgeMessagePersonalData`: sin permiso `operator`, `editor` y `auditor`.
10. Toda operación de back-office con la credencial de máquina del cliente `billing` → `403 ACCESS_DENIED`.

---

## Lo que no tiene escenario

- **La guarda propia de `sendQueuedMessage`** (`OBL-GUARD-UNOBSERVABLE`, aceptada en `decisions.yaml`):
  es interna y ningún escenario de caja negra la alcanza sin pasar por el despacho, cuyo propio
  reclamo ya impide el doble envío (FL-DSP-040). Su verificación es la comprobación de idempotencia de
  infraestructura del generador (`infra/check-idempotency.sh`).
- **El disparo diario de `purgeMessagePersonalData` a las 03:00 UTC**: se verifica su efecto por el
  disparo manual (FL-PRG-001), no el cron. Con `personalDataRetentionMonths: 0` en pruebas, un
  entorno de pruebas vivo a esa hora purgaría los mensajes terminados de cualquier flujo en curso.
- **El tope de 200 mensajes por ciclo del despacho y de 5000 por lote de la purga**: fijar la cota
  exigiría producir más de 200 (o 5000) filas en un ciclo; se declara en las reglas y su verificación
  es estática.
