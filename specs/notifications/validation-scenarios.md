# notifications — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/notifications v1.6.0. Contrato de validación para la fase de generación.

Este archivo es el **contrato de equivalencia** del servicio: del mismo diseño se generan
servidores en stacks distintos, y esto es lo único que garantiza que se comporten igual. Es
además el único gate funcional de la generación, que no produce pruebas unitarias.

## Convenciones de determinación

Valen para **todo** el servicio y ningún escenario las repite.

- **Fecha y hora.** Todo instante es UTC en ISO-8601 con `Z` (`2026-08-20T09:15:00Z`). El
  servicio no tiene fechas locales ni concepto de «hoy» del negocio: los dos calendarios
  programados (`dispatchQueuedMessages` cada minuto, `purgeMessagePersonalData` a las 03:00) y el
  umbral de retención de 18 meses viven en UTC. `requestedAt`, `sentAt`, `suppressedAt`,
  `releasedAt` y `personalDataPurgedAt` los estampa el servidor: se verifican **por forma o por
  rango**, jamás por valor literal.
- **Identificadores generados.** Todo `id` es un UUID y se verifica **por forma** y por
  reutilización simbólica: el `id` devuelto en un escenario es el que usa el siguiente de su flujo.
- **Ausencia vs nulo.** Un campo declarado en la entidad y sin valor **aparece en la respuesta con
  valor `null`**; no se omite. Es convención única de servicio y aplica a `senderName`,
  `replyToAddress`, `description`, `notes`, `failureReason`, `sentAt`, `releasedAt` y
  `personalDataPurgedAt`.
- **Campos `sensitive`.** `EmailMessage.variableValues` **nunca** aparece en ninguna respuesta, ni
  en la de creación, ni en la de consulta, ni en la de listado. Su ausencia es aserción explícita.
- **Mayúsculas y acentos.** Los identificadores de negocio se guardan y comparan **en minúsculas**:
  `code` de aplicación, `code` de plantilla y **toda dirección de correo** (destinatarios,
  direcciones suprimidas, filtro `recipient`). `Juan@X.COM` y `juan@x.com` son la misma dirección
  en todas partes: colisionan en unicidad, se deduplican entre sí y una supresión sobre una alcanza
  a la otra. El resto del texto libre (`name`, `description`, `subject`, cuerpos, `notes`) se
  conserva tal cual llegó y distingue mayúsculas.
- **Orden de las colecciones.** Toda lista viaja con orden **total** declarado por su operación.
  Ningún escenario acepta «orden indiferente».
- **Números.** El servicio no maneja dinero ni decimales. `versionNumber` y `recipientCount` son
  enteros.
- **Forma del cuerpo de error.** El servicio tiene **una** forma de error, la que emite el
  generador: `{timestamp, status, error, code, message, details}` más `correlationId`. Los
  escenarios especifican únicamente el `code` y el status; la forma se da por descrita aquí.
- **Status de todo error.** Todo error lleva el status que declara su operación en el YAML. Los
  errores de **mecanismo** que el diseño no declara pero el framework publica igualmente son
  `VALIDATION_ERROR` (400, constraints de input), `CONCURRENT_MODIFICATION` (409,
  `optimisticLocking: all` sobre la raíz `Application`) e `INVALID_STATE_TRANSITION` (409).
- **Cabecera `Location`.** **Ninguna** operación de este servicio la emite: no hay ningún endpoint
  con `successStatus: 201`, por decisión de diseño (el servicio direcciona por clave natural, no
  por id). Ningún `Then` la asserta.
- **Paginación.** Sobre de paginación canónico del DSL, con `page`/`size`, `defaultSize: 20` y
  `maxSize: 100`.
- **Idempotencia (eje HTTP).** `requestNotification` declara `idempotency: { keySource: payload-field, keyField: idempotencyKey }`
  **sin TTL**: la guarda es permanente. La clave viaja en el cuerpo, en el campo `idempotencyKey`
  que el input declara, no en una cabecera. «Mismo contenido» son exactamente tres cosas —
  `templateCode`, el **conjunto** de destinatarios normalizados y deduplicados, y el **conjunto** de
  pares nombre/valor de variables declaradas — comparadas por igualdad de valores y **sin importar
  el orden**.
- **Idempotencia (eje evento).** La suscripción usa el `metadata.eventId` del sobre `keel` para
  cortar la reentrega del broker, y la `idempotencyKey` del payload como segunda barrera. No hay
  cabecera que enviar: esos escenarios se escriben contra el canal, no contra la API.
- **Identidad del solicitante M2M.** La aplicación en cuyo nombre se pide un envío **no viaja nunca
  en el cuerpo**. En la superficie HTTP la resuelve la credencial de máquina que autentica la
  petición; en el canal de eventos, el campo `metadata.source` del sobre. Cada entrada de
  `security.serviceClients` lleva por nombre el `code` de una `Application` registrada, y es esa
  correspondencia la que hace que un envío no pueda pedirse en nombre de otro.
- **Alcance por aplicación.** El token de un usuario humano lleva el claim `applications` con los
  códigos a los que alcanza. Las operaciones sobre una aplicación concreta lo comprueban y
  devuelven `APPLICATION_FORBIDDEN` (403) fuera de alcance; las de colección **filtran** en vez de
  fallar. Los roles `notifications-admin` y `notifications-auditor` están exentos por diseño.
- **Observable por la superficie pública.** Toda aserción se comprueba llamando a la API,
  escuchando un canal de eventos o inspeccionando el **buzón** del relay de correo. Nunca la base
  de datos.
- **El buzón.** La capa `mail` entrega por SMTP. Los `Then` que hablan de correo afirman sobre el
  buzón del relay (destinatario, remitente, asunto ya interpolado y las dos partes, `html` y
  `text`), nunca sobre el `2xx` de la petición: la respuesta acepta el encargo, no lo cumple.
- **Identidades disponibles.** Roles: `notifications-admin`, `template-editor`,
  `notifications-auditor`. Clientes de máquina: `billing` y `shipping`
  (`notification:send`, `message:read`), `dispatch-only` (solo `notification:send`) y
  `mail-provider` (solo `bounce:report`). No se nombra ninguna identidad fuera de estas.
- **Lo que fija el generador y el escenario describe.** El fallo de audiencia
  (`validateAudience: true`) responde **403**, no 401. Una operación `level: service` no rechaza
  por sí sola un token de usuario: la separación es por scopes.

## Proyección de las entidades en las respuestas

Derivada de `domain` menos los `exclude` del `output`. Todo `Then` que enumere un cuerpo usa
exactamente esta lista, y añade «el cuerpo no trae ningún campo adicional».

| Entidad | Campos que viajan |
|---|---|
| `Application` | `id`, `code`, `name`, `senderAddress`, `senderName`, `replyToAddress`, `status` |
| `EmailTemplate` (siempre con `exclude: [versions]`) | `id`, `applicationId`, `code`, `name`, `description`, `hasActiveVersion` |
| `EmailTemplateVersion` | `id`, `templateId`, `versionNumber`, `subject`, `htmlBody`, `textBody`, `variables[]`, `status` |
| `EmailMessage` | `id`, `applicationId`, `idempotencyKey`, `recipients[]`, `renderedSubject`, `senderAddress`, `senderName`, `templateCode`, `templateVersionNumber`, `templateVersionId`, `status`, `failureReason`, `requestedAt`, `sendingSince`, `sentAt`, `personalDataPurgedAt` — **nunca** `variableValues` |
| `SuppressedAddress` | `id`, `applicationId`, `address`, `reason`, `notes`, `status`, `suppressedAt`, `releasedAt` |

Los campos con `default` viajan **siempre**, aunque la petición no los mande: `Application.status`
(`active`), `EmailTemplateVersion.status` (`active`), `EmailMessage.status` (`queued`) y
`SuppressedAddress.status` (`active`).

## Matriz de cobertura

| Operación | Flujos | Superficie |
|-----------|--------|------------|
| `registerApplication` | FL-APP-001, FL-APP-010 | usuarios |
| `updateApplication` | FL-APP-020 | usuarios |
| `suspendApplication` | FL-APP-030 | usuarios |
| `reactivateApplication` | FL-APP-030 | usuarios |
| `getApplication` | FL-APP-001, FL-SEC-010 | usuarios |
| `listApplications` | FL-APP-040, FL-SEC-010 | usuarios |
| `createTemplate` | FL-TPL-001, FL-TPL-010 | usuarios |
| `publishTemplateVersion` | FL-TPL-020, FL-TPL-030 | usuarios |
| `activateTemplateVersion` | FL-TPL-040 | usuarios |
| `retireTemplate` | FL-TPL-050 | usuarios |
| `getTemplate` | FL-TPL-001, FL-SEC-010 | usuarios |
| `listTemplates` | FL-TPL-060 | usuarios |
| `getTemplateVersion` | FL-TPL-020 | usuarios |
| `listTemplateVersions` | FL-TPL-070 | usuarios |
| `requestNotification` | FL-REQ-001, FL-REQ-010, FL-REQ-020, FL-REQ-030, FL-REQ-040, FL-EVT-001 | **servidores (M2M)** + eventos |
| `dispatchQueuedMessages` | FL-SND-001, FL-SND-020, FL-SND-030 | reloj |
| `sendQueuedMessage` | FL-SND-001, FL-SND-020 | interna |
| `getMessage` | FL-MSG-001, FL-SEC-020 | usuarios |
| `findMessageByIdempotencyKey` | FL-MSG-010 | **servidores (M2M)** |
| `listMessages` | FL-MSG-020, FL-MSG-030 | usuarios |
| `suppressAddress` | FL-SUP-001, FL-SUP-010, FL-SUP-040 | usuarios |
| `releaseAddress` | FL-SUP-020 | usuarios |
| `listSuppressedAddresses` | FL-SUP-030 | usuarios |
| `reportEmailBounce` | FL-BNC-001, FL-BNC-010, FL-SUP-040 | **servidores (M2M)** |
| `purgeMessagePersonalData` | FL-RET-001 | usuarios (disparo manual) + reloj |

Cobertura transversal: `FL-SEC-001` (autenticación y permisos), `FL-SEC-010` / `FL-SEC-020`
(alcance por aplicación), `FL-SEC-030` (CORS), `FL-EVT-001` / `FL-EVT-010` / `FL-EVT-020`
(suscripción, política de fallo y reentrega), `FL-PAG-001` (paginación).

## Aplicaciones consumidoras

### FL-APP-001: alta de una aplicación y su consulta

**Given**: no existe ninguna aplicación con `code` `billing`. Se actúa con el rol
`notifications-admin`.

**When**: `registerApplication` — `POST /api/v1/applications`
```json
{ "code": "Billing", "name": "Facturación", "senderAddress": "Facturas@Acme.COM",
  "senderName": "Acme Facturación", "replyToAddress": "soporte@acme.com" }
```

**Then**:
1. Status `200`. **No** se emite cabecera `Location` (ninguna operación del servicio la emite).
2. El cuerpo trae `id` (forma UUID), `code: "billing"` (normalizado a minúsculas),
   `name: "Facturación"` (conservado tal cual), `senderAddress: "facturas@acme.com"`
   (normalizado), `senderName: "Acme Facturación"`, `replyToAddress: "soporte@acme.com"` y
   `status: "active"` — el valor por defecto viaja aunque la petición no lo mande.
3. El cuerpo no trae ningún campo adicional.
4. `getApplication` — `GET /api/v1/applications/billing` responde `200` con **el mismo cuerpo**,
   campo a campo. Consultarla por el código sin normalizar (`/applications/Billing`) devuelve el
   mismo recurso.

**When**: `registerApplication` con un alta mínima
```json
{ "code": "shipping", "name": "Logística", "senderAddress": "avisos@acme.com" }
```

**Then**:
5. Status `200`, `code: "shipping"`, `status: "active"`.
6. `senderName` y `replyToAddress` vienen con valor `null` — no se omiten.

**Orden de evaluación**:
1. La petición cumple las constraints del input → `VALIDATION_ERROR` (`400`).
2. No existe ya una aplicación con ese código → `APPLICATION_CODE_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `code: "AB"` (menos de 3 caracteres, incumple el patrón) → `400` `VALIDATION_ERROR`.
- `code: "9billing"` (no empieza por letra) → `400`.
- `senderAddress: "no-es-un-correo"` → `400`.
- `name` ausente → `400`.

**Notas de determinación**: `id` se verifica por forma y se reutiliza en el resto del flujo.

### FL-APP-010: el código de aplicación es único e insensible a mayúsculas

**Given**: existe la aplicación `billing` (alta como en FL-APP-001). Rol `notifications-admin`.

**When**: `registerApplication` con `{ "code": "BILLING", "name": "Otra", "senderAddress": "x@acme.com" }`

**Then**:
1. Status `409` con code `APPLICATION_CODE_ALREADY_EXISTS`.
2. `listApplications` devuelve **exactamente una** aplicación con `code: "billing"`: la colisión no
   creó un segundo registro.

**Casos borde**:
- `code: "billing"` idéntico → mismo `409`, mismo code.
- Un código libre (`code: "billingx"`) → `200`: la colisión era solo por el código.

### FL-APP-020: actualización parcial y borrado explícito de un campo opcional

**Given**: existe `billing` con `senderName: "Acme Facturación"` y
`replyToAddress: "soporte@acme.com"`. Rol `notifications-admin`.

**When**: `updateApplication` — `PATCH /api/v1/applications/billing`
```json
{ "name": "Facturación Acme" }
```

**Then**:
1. Status `200`.
2. `name: "Facturación Acme"`; `senderAddress`, `senderName`, `replyToAddress` y `status`
   conservan su valor anterior — omitir un campo significa **no tocarlo**.
3. `code` sigue siendo `billing`: el código identifica el registro y nunca se modifica.

**When**: `updateApplication` — `PATCH /api/v1/applications/billing` con `{ "replyToAddress": "" }`

**Then**:
4. Status `200` y `replyToAddress` con valor `null`: enviar vacío un campo opcional **lo borra**, y
   es la única forma de quitarlo.
5. `senderName` sigue informado — borrar uno no borra los demás.

**When**: `updateApplication` — `PATCH /api/v1/applications/billing` con `{ "senderAddress": "" }`

**Then**:
6. Status `400` con code `APPLICATION_FIELD_REQUIRED`: `senderAddress` es requerido y **no** admite
   el vaciado. La distinción con el paso 4 es la que importa: enviar vacío borra lo que la
   aplicación puede no tener, y se rechaza sobre lo que no puede faltar.
7. `getApplication` sobre `billing` sigue devolviendo el `senderAddress` anterior — un rechazo no
   deja el recurso a medias.

**Orden de evaluación**:
1. La petición cumple las constraints del input → `VALIDATION_ERROR` (`400`).
2. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
3. Los campos requeridos no llegan presentes y en blanco → `APPLICATION_FIELD_REQUIRED` (`400`).

**Casos borde**:
- `PATCH /api/v1/applications/inexistente` → `404` `APPLICATION_NOT_FOUND`.
- `senderAddress: "roto"` sobre una aplicación **inexistente** → `400` `VALIDATION_ERROR`: la
  validación del input precede a la búsqueda del recurso.
- `{ "name": "" }` → `400` `APPLICATION_FIELD_REQUIRED`: vale para los dos campos requeridos, no
  solo para el remitente.
- `{ "senderName": "" }` → `200` y `senderName` a `null`: es opcional, así que sí se vacía.

**Notas de determinación**: cambiar `senderAddress` aquí **no** reescribe el `senderAddress` de los
mensajes ya emitidos; lo verifica FL-REQ-040.

### FL-APP-030: suspensión y reactivación

**Given**: existe `billing` con `status: "active"`. Rol `notifications-admin`.

**When**: `suspendApplication` — `POST /api/v1/applications/billing/suspend`
```json
{ "reason": "impago" }
```

**Then**:
1. Status `200` y `status: "suspended"` (transición `active` → `suspended`).
2. `getApplication` sobre `billing` devuelve `status: "suspended"`.

**When**: `suspendApplication` sobre la misma aplicación otra vez

**Then**:
3. Status `409` con code `APPLICATION_ALREADY_SUSPENDED` — es la aplicación de `transitions` desde
   un estado que no está en su `from`.

**When**: `reactivateApplication` — `POST /api/v1/applications/billing/reactivate`

**Then**:
4. Status `200` y `status: "active"` (transición `suspended` → `active`).

**When**: `reactivateApplication` otra vez

**Then**:
5. Status `409` con code `APPLICATION_NOT_SUSPENDED`.

**Orden de evaluación** (ambas operaciones):
1. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
2. Está en el estado de partida → `APPLICATION_ALREADY_SUSPENDED` / `APPLICATION_NOT_SUSPENDED` (`409`).

**Casos borde**:
- `POST /api/v1/applications/inexistente/suspend` → `404`: la guarda 1 precede a la 2, así que una
  aplicación inexistente da `404` y no `409`, aunque las dos guardas «fallen».

**Notas de determinación**: suspender cierra **la entrada de envíos**, no la administración: las
plantillas de una aplicación suspendida se siguen gestionando (lo cubre FL-TPL-020, caso borde) y
sus mensajes ya encolados se siguen enviando (FL-SND-030).

### FL-APP-040: listado de aplicaciones, orden total y filtro por estado

**Given**: rol `notifications-admin`. Existen tres aplicaciones creadas en este orden:
`{ code: "zeta", name: "Común" }`, `{ code: "alfa", name: "Común" }` y
`{ code: "beta", name: "Aparte" }`. `zeta` se suspende.

**When**: `listApplications` — `GET /api/v1/applications`

**Then**:
1. Status `200` y sobre de paginación con `size: 20` (el `defaultSize`) y 3 elementos.
2. El orden es exactamente `beta` (`name: "Aparte"`), `alfa`, `zeta` — `name` ascendente y, para
   las dos que empatan en `Común`, `code` ascendente. El orden de creación es el inverso: si la
   implementación devolviera el de inserción, este `Then` falla.

**When**: `listApplications` — `GET /api/v1/applications?status=suspended`

**Then**:
3. Status `200` con **un** elemento, `code: "zeta"`.

**When**: `GET /api/v1/applications?status=active`

**Then**:
4. Status `200` con dos elementos, en orden `beta`, `alfa`.

**Casos borde**:
- `?status=inexistente` → `400` `VALIDATION_ERROR` (no es un valor del enum).
- Un filtro que no case con nada devuelve una página **vacía** con el sobre completo, no `404`.

## Plantillas y sus versiones

### FL-TPL-001: creación de una plantilla sin contenido publicable

**Given**: existe la aplicación `billing`. Se actúa con el rol `template-editor` cuyo claim
`applications` incluye `billing`.

**When**: `createTemplate` — `POST /api/v1/applications/billing/templates`
```json
{ "code": "InvoiceReady", "name": "Factura disponible",
  "description": "Avisa al cliente de que su factura ya puede descargarse" }
```

**Then**:
1. Status `200`.
2. El cuerpo trae `id` (UUID), `applicationId` (el de `billing`), `code: "invoiceready"`
   (normalizado), `name: "Factura disponible"`, `description` tal cual, y
   `hasActiveVersion: false` — recién creada no tiene ninguna versión, así que no puede usarse para
   enviar.
3. El cuerpo no trae `versions` (`exclude`) ni ningún campo adicional.
4. `getTemplate` — `GET /api/v1/applications/billing/templates/invoiceready` responde `200` con el
   mismo cuerpo.

**Orden de evaluación**:
1. Constraints del input → `VALIDATION_ERROR` (`400`).
2. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
3. La aplicación está en el alcance del solicitante → `APPLICATION_FORBIDDEN` (`403`).
4. No hay ya una plantilla con ese código en esa aplicación → `TEMPLATE_CODE_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `POST /api/v1/applications/inexistente/templates` → `404` `APPLICATION_NOT_FOUND`.
- `GET .../templates/noexiste` → `404` `TEMPLATE_NOT_FOUND`.

### FL-TPL-010: el código de plantilla es único dentro de su aplicación, no entre aplicaciones

**Given**: existen las aplicaciones `billing` y `shipping`, y en `billing` la plantilla
`invoiceready`. Rol `notifications-admin` (alcanza a las dos).

**When**: `createTemplate` sobre `billing` con `{ "code": "INVOICEREADY", "name": "Duplicada" }`

**Then**:
1. Status `409` con code `TEMPLATE_CODE_ALREADY_EXISTS` — la comparación es insensible a mayúsculas.
2. `listTemplates` de `billing` devuelve **una** plantilla con `code: "invoiceready"`.

**When**: `createTemplate` sobre `shipping` con `{ "code": "invoiceready", "name": "Otra app" }`

**Then**:
3. Status `200`: el código solo es único **dentro** de su aplicación.
4. Su `applicationId` es el de `shipping` y su `id` es distinto del de la plantilla de `billing`.

### FL-TPL-020: publicación de contenido y validación de variables

**Given**: existe `billing` con la plantilla `invoiceready` y **ninguna** versión. Rol
`template-editor` con `billing` en su alcance.

**When**: `publishTemplateVersion` — `POST /api/v1/applications/billing/templates/invoiceready/versions`
```json
{ "subject": "Tu factura {{invoiceNumber}} está lista",
  "htmlBody": "<p>Hola {{customerName}}, tu factura {{invoiceNumber}} ya está disponible.</p>",
  "textBody": "Hola {{customerName}}, tu factura {{invoiceNumber}} ya está disponible.",
  "variables": [ { "name": "customerName", "required": true },
                 { "name": "invoiceNumber", "required": true } ] }
```

**Then**:
1. Status `200`.
2. El cuerpo trae `id` (UUID), `templateId` (el de `invoiceready`), `versionNumber: 1` — la
   numeración empieza en 1 y la calcula el servidor, nunca el cliente —, `subject`, `htmlBody` y
   `textBody` tal cual se enviaron, `variables` con dos elementos (`customerName` y
   `invoiceNumber`, ambos con `required: true`) y `status: "active"`.
3. El cuerpo no trae ningún campo adicional.
4. `getTemplate` sobre `invoiceready` devuelve ahora `hasActiveVersion: true`: se recalculó en la
   misma transacción que la publicación.
5. `getTemplateVersion` — `GET .../templates/invoiceready/versions/1` responde `200` con el mismo
   cuerpo.

**When**: `publishTemplateVersion` otra vez, con `subject` `"Tu factura {{invoiceNumber}} (v2)"` y
las mismas variables

**Then**:
6. Status `200` con `versionNumber: 2` y `status: "active"`.
7. `getTemplateVersion` sobre la versión `1` responde `200` con `status: "archived"`: la anterior
   se archivó **en la misma transacción** (transición `active` → `archived`).
8. `getTemplate` sigue devolviendo `hasActiveVersion: true`.
9. El contenido de la versión `1` es **idéntico** al del paso 2: publicar una versión nueva no
   modifica el contenido de la anterior.

**Orden de evaluación**:
1. Constraints del input → `VALIDATION_ERROR` (`400`).
2. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
3. La aplicación está en el alcance → `APPLICATION_FORBIDDEN` (`403`).
4. La plantilla existe → `TEMPLATE_NOT_FOUND` (`404`).
5. Las variables declaradas no se repiten → `DUPLICATE_TEMPLATE_VARIABLE` (`422`).
6. El asunto y los cuerpos solo usan variables declaradas → `UNDECLARED_TEMPLATE_VARIABLE` (`422`).

**Casos borde**:
- `subject` con `{{nombreQueNadieDeclara}}` y `variables: []` → `422`
  `UNDECLARED_TEMPLATE_VARIABLE`.
- Una variable usada solo en `htmlBody` y no declarada → `422` `UNDECLARED_TEMPLATE_VARIABLE`: la
  comprobación cubre asunto **y** los dos cuerpos.
- `variables` con dos entradas de `name: "customerName"` → `422` `DUPLICATE_TEMPLATE_VARIABLE`.
- Duplicada **y** sin declarar a la vez → `422` `DUPLICATE_TEMPLATE_VARIABLE`: la guarda 5 precede
  a la 6.
- `htmlBody` de más de 262144 caracteres → `400` `VALIDATION_ERROR`.
- `textBody` de más de 65536 caracteres → `400` `VALIDATION_ERROR`.
- `textBody` ausente → `400`: la alternativa en texto plano es obligatoria.
- Publicar sobre una aplicación **suspendida** → `200`: suspender cierra el envío, no la
  administración de plantillas.
- `variables` con 51 elementos → `400` `VALIDATION_ERROR` (`maxItems: 50`).

### FL-TPL-030: dos publicaciones concurrentes sobre la misma plantilla

**Given**: existe `billing` con la plantilla `invoiceready` y su versión `1` activa. Rol
`notifications-admin`.

**When**: se lanzan **a la vez** dos `publishTemplateVersion` sobre `invoiceready`, con asuntos
distintos (`"A {{v}}"` y `"B {{v}}"`, ambos declarando `v`).

**Then**:
1. Desenlace admisible, disyunción cerrada: o bien una responde `200` y la otra `409` con code
   `CONCURRENT_MODIFICATION` — `persistence.consistency.optimisticLocking: all` sobre la raíz
   `Application` —, o bien ambas responden `200` habiéndose serializado.
2. **Sea cual sea el ganador**: `listTemplateVersions` de `invoiceready` devuelve versiones con
   `versionNumber` **todos distintos y consecutivos** empezando en 1. Nunca dos filas con el mismo
   número.
3. **Sea cual sea el ganador**: existe **exactamente una** versión con `status: "active"`. Es la
   invariante que el bloqueo optimista protege, y la aserción que no depende de quién gane.

**Notas de determinación**: no se deduce el ganador cruzando observaciones; solo se afirma el
conteo leído por la API.

### FL-TPL-040: revertir a una versión anterior

**Given**: existe `billing` con `invoiceready` y dos versiones: la `1` `archived` y la `2`
`active`. Rol `template-editor` con `billing` en su alcance.

**When**: `activateTemplateVersion` — `POST .../templates/invoiceready/versions/1/activate`

**Then**:
1. Status `200` con `versionNumber: 1` y `status: "active"` (transición `archived` → `active`).
2. `getTemplateVersion` de la versión `2` devuelve `status: "archived"` — la que estaba activa se
   archivó en la misma transacción (transición `active` → `archived`).
3. El `subject`, los cuerpos y las `variables` de la versión `1` son idénticos a los que tenía:
   reactivar no crea una versión nueva ni cambia su contenido.
4. `listTemplateVersions` devuelve **dos** versiones: no se creó ninguna tercera.

**When**: `activateTemplateVersion` sobre la versión `1` otra vez

**Then**:
5. Status `409` con code `TEMPLATE_VERSION_ALREADY_ACTIVE`.

**Orden de evaluación**:
1. Constraints del input → `VALIDATION_ERROR` (`400`).
2. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
3. La aplicación está en el alcance → `APPLICATION_FORBIDDEN` (`403`).
4. La plantilla existe → `TEMPLATE_NOT_FOUND` (`404`).
5. La versión existe → `TEMPLATE_VERSION_NOT_FOUND` (`404`).
6. La versión no era ya la activa → `TEMPLATE_VERSION_ALREADY_ACTIVE` (`409`).

**Casos borde**:
- `.../versions/99/activate` → `404` `TEMPLATE_VERSION_NOT_FOUND`.
- `.../versions/0/activate` → `400` `VALIDATION_ERROR` (`min: 1`).
- Plantilla inexistente **y** versión inexistente → `404` `TEMPLATE_NOT_FOUND`: la guarda 4 precede
  a la 5.

### FL-TPL-050: retirar una plantilla de circulación

**Given**: existe `billing` con `invoiceready` y su versión `1` activa. Rol `template-editor` con
`billing` en su alcance.

**When**: `retireTemplate` — `POST .../templates/invoiceready/retire`

**Then**:
1. Status `200` y el cuerpo de `EmailTemplate` con `hasActiveVersion: false` — recalculado en la
   misma transacción.
2. `getTemplateVersion` de la versión `1` devuelve `status: "archived"` (transición
   `active` → `archived`).
3. `getTemplate` sigue respondiendo `200`: retirar **no borra nada**, la plantilla y su historia se
   conservan.
4. `listTemplateVersions` sigue devolviendo la versión `1` con su contenido íntegro.

**When**: `retireTemplate` sobre la misma plantilla otra vez

**Then**:
5. Status `409` con code `TEMPLATE_NOT_PUBLISHED` — no había ninguna versión activa que retirar.

**When**: `activateTemplateVersion` sobre la versión `1`

**Then**:
6. Status `200`: la plantilla vuelve a circulación y `getTemplate` devuelve
   `hasActiveVersion: true`.

**Orden de evaluación**:
1. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
2. La aplicación está en el alcance → `APPLICATION_FORBIDDEN` (`403`).
3. La plantilla existe → `TEMPLATE_NOT_FOUND` (`404`).
4. Tiene una versión activa → `TEMPLATE_NOT_PUBLISHED` (`409`).

**Casos borde**:
- Retirar una plantilla **recién creada y nunca publicada** → `409` `TEMPLATE_NOT_PUBLISHED`.

### FL-TPL-060: listado de plantillas y filtro por circulación

**Given**: existe `billing`. Rol `notifications-admin`. Se crean las plantillas `zeta`, `alfa` y
`beta` en ese orden; se publica una versión en `zeta` y en `alfa`; después se retira `alfa`.

**When**: `listTemplates` — `GET /api/v1/applications/billing/templates`

**Then**:
1. Status `200` con sobre de paginación y 3 elementos.
2. Orden exacto `alfa`, `beta`, `zeta` — `code` ascendente. El orden de creación es otro.
3. `alfa` trae `hasActiveVersion: false`, `beta` `false` (nunca publicada) y `zeta` `true`.

**When**: `GET /api/v1/applications/billing/templates?hasActiveVersion=true`

**Then**:
4. Status `200` con **un** elemento, `code: "zeta"`.

**When**: `GET /api/v1/applications/billing/templates?hasActiveVersion=false`

**Then**:
5. Status `200` con dos elementos, en orden `alfa`, `beta` — una retirada y otra nunca publicada
   son ambas «fuera de circulación».

**Casos borde**:
- Sin el filtro se devuelven **todas**, en circulación y retiradas (paso 1).
- `GET /api/v1/applications/inexistente/templates` → `404` `APPLICATION_NOT_FOUND`.

### FL-TPL-070: historial de versiones, de la más reciente a la más antigua

**Given**: existe `billing` con `invoiceready` y **tres** versiones publicadas en orden (1, 2, 3).
Rol `notifications-auditor`.

**When**: `listTemplateVersions` — `GET .../templates/invoiceready/versions`

**Then**:
1. Status `200` con 3 elementos.
2. Orden exacto `versionNumber` `3`, `2`, `1` — descendente. El orden de inserción es el inverso.
3. La versión `3` trae `status: "active"`; las `2` y `1`, `"archived"`.
4. Cada elemento trae el cuerpo completo de `EmailTemplateVersion`, cuerpos incluidos.

**Casos borde**:
- `GET .../templates/noexiste/versions` → `404` `TEMPLATE_NOT_FOUND`.
- Una plantilla sin ninguna versión devuelve una página **vacía** con el sobre completo, no `404`.

## Petición de envío (superficie servidor-a-servidor)

### FL-REQ-001: contrato M2M completo de la petición de envío

**Given**: existe la aplicación `billing` (`status: active`,
`senderAddress: "facturas@acme.com"`, `senderName: "Acme Facturación"`,
`replyToAddress: "soporte@acme.com"`) con la plantilla `invoiceready` y su versión `1` activa
(asunto `"Tu factura {{invoiceNumber}} está lista"`, variables `customerName` e `invoiceNumber`,
ambas `required`). No existe ningún mensaje. Se actúa con la **credencial de máquina del cliente
`billing`** (scopes `notification:send`, `message:read`).

**When**: `requestNotification` — `POST /api/v1/notifications`
```json
{ "templateCode": "InvoiceReady", "idempotencyKey": "inv-2026-000123",
  "recipients": ["Cliente@Example.COM", "cliente@example.com", "otro@example.com"],
  "variables": [ { "name": "customerName", "value": "Ana" },
                 { "name": "invoiceNumber", "value": "F-000123" },
                 { "name": "sobra", "value": "se descarta" } ] }
```

**Then**:
1. Status `202` — la respuesta **acepta el encargo, no afirma que el correo se haya entregado**.
2. El cuerpo trae `id` (UUID), `applicationId` (el de `billing`), `idempotencyKey`
   `"inv-2026-000123"`, `recipients` con **exactamente dos** elementos —
   `["cliente@example.com", "otro@example.com"]`, normalizados a minúsculas y deduplicados: las dos
   primeras direcciones de la petición son la misma persona —, `renderedSubject`
   `"Tu factura F-000123 está lista"` (el asunto ya resuelto en el momento de aceptar),
   `senderAddress` `"facturas@acme.com"` y `senderName` `"Acme Facturación"` (las dos mitades
   del remitente, copiadas de la aplicación), `templateCode`
   `"invoiceready"`, `templateVersionNumber: 1`, `templateVersionId` (el `id` de la versión `1`),
   `status: "queued"`, `failureReason: null`, `requestedAt` (instante UTC, por forma),
   `sendingSince: null` —el mensaje no ha salido de `queued`—, `sentAt: null` y
   `personalDataPurgedAt: null`.
3. El cuerpo **no** trae `variableValues` (campo `sensitive`) ni ningún campo adicional. La variable
   `sobra`, que la versión no declara, se descartó sin error.
4. El cuerpo **no** trae `applicationCode` recibido del cliente: la aplicación se resolvió de la
   credencial, no del cuerpo.
5. `findMessageByIdempotencyKey` —
   `GET /api/v1/notifications/billing/inv-2026-000123`, misma credencial — responde `200` con el
   mismo cuerpo.
6. En ese instante **no ha salido ningún correo**: el buzón del relay sigue vacío. El envío lo hace
   `dispatchQueuedMessages` (FL-SND-001).

**When**: la misma petición (clave nueva) nombrando explícitamente otra aplicación —
`{ "applicationCode": "shipping", ... }` con la credencial de `billing`

**Then**:
7. Status `202`, y el mensaje creado es de **`billing`**: `applicationCode` no viaja en el cuerpo
   —lo estampa el servidor desde la credencial—, así que el campo enviado se ignora y el
   solicitante **no puede** actuar en nombre de otra aplicación. Se afirma sobre el recurso
   creado, no sobre el status: un 202 solo diría que la petición se aceptó, no en nombre de quién.
8. `listMessages?applicationCode=shipping` no devuelve ese mensaje.

**Orden de evaluación**:
1. Constraints del input → `VALIDATION_ERROR` (`400`).
2. La identidad del solicitante corresponde a una aplicación registrada →
   `APPLICATION_NOT_FOUND` (`404`).
3. *(Ya no existe: la aplicación la estampa el servidor desde la credencial, así que la petición no
   puede nombrar otra. La numeración se conserva para no reescribir las referencias de abajo.)*
4. **La repetición**: no existe ya un mensaje de esa aplicación con esa clave y otro contenido →
   `IDEMPOTENCY_KEY_REUSED` (`409`); ni otra petición con la misma clave confirmando ahora mismo →
   `IDEMPOTENCY_KEY_IN_PROGRESS` (`409`). Va **después** de resolver la aplicación porque la
   comprobación busca el duplicado dentro de ella, y **antes** que todo lo demás.
5. La aplicación no está suspendida → `APPLICATION_SUSPENDED` (`409`).
6. La plantilla existe → `TEMPLATE_NOT_FOUND` (`404`).
7. La plantilla tiene versión activa → `TEMPLATE_NOT_PUBLISHED` (`409`).
8. Ningún destinatario está suprimido → `RECIPIENT_SUPPRESSED` (`422`).
9. Están todas las variables `required` → `MISSING_TEMPLATE_VARIABLES` (`422`).

**Casos borde**:
- `recipients: []` → `400` `VALIDATION_ERROR` (`minItems: 1`).
- `recipients` con 21 elementos → `400` (`maxItems: 20`).
- `recipients: ["roto"]` → `400`.
- `idempotencyKey: "corta"` (menos de 8) → `400`.
- `idempotencyKey` ausente → `400`.
- `templateCode: "noexiste"` → `404` `TEMPLATE_NOT_FOUND`.
- Falta `invoiceNumber` en `variables` → `422` `MISSING_TEMPLATE_VARIABLES`.
- `invoiceNumber` presente con `value: ""` → `422` `MISSING_TEMPLATE_VARIABLES`: una variable
  `required` exige valor **no vacío**.
- Con la credencial de `dispatch-only` (solo `notification:send`) → `202`: pedir un envío no exige
  `message:read`.

### FL-REQ-010: precedencia entre las guardas de la petición de envío

**Given**: existe `billing` **suspendida**, con la plantilla `invoiceready` **retirada** (sin
versión activa) y con `cliente@example.com` en su lista de supresión (`status: active`). Credencial
de máquina del cliente `billing`.

**When**: `requestNotification` con `templateCode: "invoiceready"`,
`idempotencyKey: "prec-000001"`, `recipients: ["cliente@example.com"]` y **sin** variables — de
modo que las guardas 5, 7, 8 y 9 fallarían todas a la vez.

**Then**:
1. Status `409` con code `APPLICATION_SUSPENDED`, y **no** `TEMPLATE_NOT_PUBLISHED`,
   `RECIPIENT_SUPPRESSED` ni `MISSING_TEMPLATE_VARIABLES`: la guarda 5 precede a las tres.
2. `findMessageByIdempotencyKey` sobre `prec-000001` responde `404` `MESSAGE_NOT_FOUND`: una
   petición rechazada **no encola nada**.
3. El buzón del relay sigue vacío.

**When**: se reactiva `billing` y se repite la petición (misma clave)

**Then**:
4. Status `409` con code `TEMPLATE_NOT_PUBLISHED`, y no `RECIPIENT_SUPPRESSED`: la guarda 7 precede
   a la 8.

**When**: se publica una versión con las dos variables `required` y se repite la petición

**Then**:
5. Status `422` con code `RECIPIENT_SUPPRESSED`, y no `MISSING_TEMPLATE_VARIABLES`: la guarda 8
   precede a la 9. Sigue sin encolarse nada.

**When**: se libera `cliente@example.com` y se repite la petición, aún sin variables

**Then**:
6. Status `422` con code `MISSING_TEMPLATE_VARIABLES`.

**When**: se repite con las dos variables informadas

**Then**:
7. Status `202`: la clave `prec-000001` **no había quedado quemada** por los rechazos anteriores,
   porque ninguno llegó a encolar un mensaje.

### FL-REQ-020: el reintento secuencial con la misma clave no manda un segundo correo

**Given**: el mismo estado inicial de FL-REQ-001. Credencial de máquina del cliente `billing`.

**When**: `requestNotification` con `idempotencyKey: "inv-2026-000200"` y contenido válido

**Then**:
1. Status `202` con el cuerpo completo del mensaje; se guarda su `id`.

**When**: **la misma petición otra vez**, byte a byte

**Then**:
2. Mismo status `202` y **el mismo cuerpo**, `id` incluido: se reprodujo la respuesta original.
3. `listMessages?applicationCode=billing` devuelve **exactamente un** mensaje: no se encoló un
   segundo.

**When**: la misma clave con los destinatarios y las variables **en otro orden**

**Then**:
4. Mismo `202` y mismo `id`: reordenar no es contenido distinto.

**When**: la misma clave con `templateCode` distinto (otra plantilla publicada de `billing`)

**Then**:
5. Status `409` con code `IDEMPOTENCY_KEY_REUSED`.
6. `listMessages?applicationCode=billing` sigue devolviendo **un** mensaje, el original y sin
   modificar.

**When**: una clave **distinta** (`"inv-2026-000201"`) con el mismo contenido

**Then**:
7. Status `202` con un `id` **nuevo** y distinto: la guarda es la clave, no el contenido.
8. `listMessages?applicationCode=billing` devuelve **dos** mensajes.

**When**: la aplicación se suspende y se repite la petición original de `"inv-2026-000200"`

**Then**:
9. Status `202` con el mismo cuerpo, **sin** `APPLICATION_SUSPENDED`: la comprobación de repetición
   precede a la de suspensión, y una repetición reproduce la respuesta aunque el estado haya
   cambiado.

**Notas de determinación**: la guarda es **permanente** (`idempotency` sin `ttlSeconds`); ningún
escenario espera a que caduque porque no caduca.

### FL-REQ-030: dos peticiones con la misma clave a la vez

**Given**: el mismo estado inicial de FL-REQ-001, sin ningún mensaje. Credencial de máquina del
cliente `billing`.

**When**: se lanzan **a la vez** dos `requestNotification` idénticas con
`idempotencyKey: "race-000001"`.

**Then**:
1. Desenlace admisible, disyunción cerrada: o bien las dos responden `202` con **el mismo** `id`, o
   bien una responde `202` y la otra `409` con code `IDEMPOTENCY_KEY_IN_PROGRESS` — la clave se
   registra en la misma transacción que el mensaje, así que hasta que la ganadora no confirma no hay
   respuesta que reproducir.
2. **Sea cual sea el ganador**, y esta es la aserción que no depende de él:
   `listMessages?applicationCode=billing` devuelve **exactamente un** mensaje, y
   `findMessageByIdempotencyKey` sobre `race-000001` responde `200` con ese mismo `id`.
3. **Sea cual sea el ganador**: tras ejecutar `dispatchQueuedMessages`, el buzón del relay contiene
   **exactamente un** correo para esos destinatarios.

**Notas de determinación**: este escenario no sustituye a FL-REQ-020. El reintento secuencial
encuentra el registro de la clave ya confirmado y lo resuelve una lectura; esta carrera cae en la
ventana en la que todavía no lo está, que es donde vive el fallo real.

### FL-REQ-040: el mensaje congela la versión de plantilla y el remitente

**Given**: existe `billing` con `senderAddress: "facturas@acme.com"` y
`senderName: "Acme Facturación"`, y la plantilla `invoiceready` con su versión `1` activa (asunto
`"V1 {{invoiceNumber}}"`). Credencial de máquina del cliente `billing`; rol `notifications-admin`
para la administración.

**When**: `requestNotification` con `idempotencyKey: "freeze-000001"` y las variables informadas

**Then**:
1. Status `202` con `templateVersionNumber: 1`, `renderedSubject: "V1 F-000123"`,
   `senderAddress: "facturas@acme.com"` y `senderName: "Acme Facturación"`; se guarda su `id`.

**When**: se publica la versión `2` (asunto `"V2 {{invoiceNumber}}"`) y se cambian **las dos
mitades** del remitente de la aplicación: `senderAddress` a `"nuevas@acme.com"` y `senderName` a
`"Acme Cobros"`

**Then**:
2. `getMessage` sobre el `id` guardado sigue devolviendo `templateVersionNumber: 1`,
   `templateVersionId` el de la versión `1`, `renderedSubject: "V1 F-000123"`, `senderAddress`
   `"facturas@acme.com"` y `senderName` `"Acme Facturación"`: publicar una versión nueva o cambiar
   el remitente **no reescribe el histórico**.
3. `senderAddress` y `senderName` se congelaron **juntos**. Un cuerpo que devolviera la dirección
   de entonces con el nombre de ahora (`"facturas@acme.com"` + `"Acme Cobros"`) no pasa este
   escenario: describe una cabecera `From` que nunca existió.
4. Tras ejecutar `dispatchQueuedMessages`, el correo del buzón sale con asunto `"V1 F-000123"` y
   remitente `"facturas@acme.com"` con nombre visible `"Acme Facturación"` — el correo se compone
   de lo **registrado en el mensaje**, no de lo vigente en la aplicación.

**When**: `requestNotification` con una clave nueva (`"freeze-000002"`)

**Then**:
5. Status `202` con `templateVersionNumber: 2`, `renderedSubject: "V2 F-000123"`,
   `senderAddress: "nuevas@acme.com"` y `senderName: "Acme Cobros"`: las peticiones **nuevas** sí
   usan lo vigente.

**Casos borde**:

- Aplicación **sin nombre visible**: `shipping` tiene `senderAddress: "avisos@acme.com"` y
  `senderName: null`. Su mensaje se acepta con `senderName: null` —el campo viaja con valor nulo,
  no se omite— y el correo sale con remitente `"avisos@acme.com"` **sin** nombre visible; no sale
  con el nombre vacío entre comillas.
- Vaciar el `senderName` de la aplicación (`updateApplication` con `{ "senderName": "" }`) **no**
  toca el `senderName` de los mensajes ya aceptados, por la misma razón que no toca su
  `senderAddress`.

## Despacho, salida del correo y desenlace

Estos flujos son los que ejercitan la capa `mail`. Sus `Then` afirman sobre el **buzón del relay**,
nunca sobre el `2xx` de la petición que encoló el mensaje.

### FL-SND-001: el correo sale con sus dos partes y se publica EmailSent

**Given**: existe `billing` (`senderAddress: "facturas@acme.com"`,
`senderName: "Acme Facturación"`, `replyToAddress: "soporte@acme.com"`) con la plantilla
`invoiceready` y su versión `1` activa: asunto `"Tu factura {{invoiceNumber}} está lista"`,
`htmlBody` `"<p>Hola {{customerName}}</p>"`, `textBody` `"Hola {{customerName}}"`. Existe un
mensaje `status: "queued"` para `cliente@example.com` con `customerName: "Ana"` e
`invoiceNumber: "F-000123"` (encolado como en FL-REQ-001); se conoce su `id`. El buzón del relay
está vacío y el canal `notificationEvents` no tiene mensajes.

**When**: se ejecuta el ciclo `dispatchQueuedMessages`, que invoca `sendQueuedMessage` sobre el
mensaje (transición `queued` → `sending` → `sent`). El relay acepta el mensaje.

**Then**:
1. El buzón del relay contiene **exactamente un** correo.
2. Ese correo va dirigido a `cliente@example.com`, sale de `facturas@acme.com` con nombre visible
   `Acme Facturación` y lleva `Reply-To: soporte@acme.com`.
3. Su asunto es `"Tu factura F-000123 está lista"` — **ya interpolado**, sin ningún `{{...}}` sin
   resolver.
4. Lleva **las dos partes** que declara `delivery.parts`: una `html` con `"<p>Hola Ana</p>"` y una
   `text` con `"Hola Ana"`. Un correo con solo HTML no pasa este escenario.
5. No lleva ningún adjunto (`attachments: false`).
6. `getMessage` sobre el `id` devuelve `status: "sent"`, `sentAt` informado (instante UTC, por
   forma) y `failureReason: null`.
7. Se publica **exactamente un** `EmailSent` en el canal `notificationEvents`, con `messageId` (el
   `id` del mensaje), `applicationCode: "billing"`, `templateCode: "invoiceready"`,
   `idempotencyKey` (el del mensaje), `recipientCount: 1` y `sentAt`.
8. **No** se publica ningún `EmailDeliveryFailed`.

**When**: se ejecuta `dispatchQueuedMessages` otra vez, sin encolar nada nuevo

**Then**:
9. El buzón sigue con **un** correo: un mensaje que ya no está `queued` no se vuelve a componer.
   Es la guarda contra el doble envío, y es una transición irrepetible, no una clave.
10. No se publica ningún evento nuevo.
11. `getMessage` sobre el `id` sigue devolviendo `status: "sent"` con el **mismo** `sentAt`: el
    segundo intento no reescribió nada.

**Notas de determinación**: una aplicación sin `replyToAddress` produce un correo **sin** cabecera
`Reply-To`; no se envía vacía.

Los dos errores de `sendQueuedMessage` **no** son pasos de este `Then`, y no por olvido: la
operación es `internal: true` y ningún endpoint la expone, así que no hay llamada que devuelva su
`code`. `MESSAGE_NOT_QUEUED` (`409`) sobre un mensaje que ya no está `queued` es exactamente lo que
el paso 9 observa desde fuera —el buzón no recibe un segundo correo— y `MESSAGE_NOT_FOUND` (`404`)
sobre un identificador inexistente no tiene ninguna proyección observable. Los dos son contrato de
la operación interna y viven en `use-cases.keel.yaml`; escribirlos aquí como aserciones numeradas
prometería una comprobación que ninguna suite de caja negra puede escribir.

### FL-SND-020: el rescate de un mensaje atascado en sending

**Given**: existe un mensaje en estado `sending` cuyo `sendingSince` tiene **más de 15 minutos** —
el estado en el que lo dejaría un proceso caído después de tomarlo y antes de resolver el envío.
Ninguna otra vía saca un mensaje de ese estado. `sendingSince` es la marca que hace este `Given`
alcanzable: es una columna determinista que la prueba puede retrasar, y sin ella no había ningún
dato observable que distinguiera un mensaje recién tomado de uno atascado.

**When**: se ejecuta el ciclo `dispatchQueuedMessages`, que **antes** de tomar su tanda rescata los
atascados (transición `sending` → `failed`).

**Then**:
1. `getMessage` devuelve `status: "failed"` y `failureReason` informado con el motivo del **rescate**
   (distinguible del motivo que reporta un relay: el rescate no habló con el relay).
2. El buzón del relay **no** recibe ningún correo por ese mensaje: el rescate **no reenvía**.
3. No se publica `EmailSent`.
4. Se publica **exactamente un** `EmailDeliveryFailed` en el canal `notificationEvents`, con
   `messageId` (el `id` del mensaje), `applicationCode`, `templateCode`, `idempotencyKey`,
   `failureReason` —el del **rescate**, el mismo que devuelve `getMessage`— y `failedAt` (instante
   UTC, por forma). El rescate es una de las **dos** vías que llevan a `failed`, y las dos avisan:
   un mensaje que muere porque su despachador se cayó no puede ser invisible para quien escucha.
5. `failureReason` es lo que distingue las dos vías en el cable. El del rescate no afirma que el
   correo no saliera —afirma que no se sabe—, y es el dato con el que un consumidor decide si
   re-pide el envío. Ningún escenario afirma que un mensaje rescatado no llegó nunca.

**When**: se ejecuta un **segundo** ciclo de `dispatchQueuedMessages` sobre el mismo mensaje ya
rescatado

**Then**:
6. No se publica un segundo `EmailDeliveryFailed` para ese `messageId`: el evento sale únicamente
   si la transición a `failed` se aplicó en ese ciclo, y `failed` es terminal.
7. Si el despachador que lo tenía tomado estaba **lento** y no muerto, y vuelve del relay después
   del rescate, no escribe nada y **no publica nada** —ni `EmailSent` ni un segundo
   `EmailDeliveryFailed`—: el mensaje ya no está en `sending` y ninguna de sus dos transiciones es
   legal desde `failed`.

**Notas de determinación**: el falso negativo es deliberado y está en el diseño. Si la caída ocurrió
*después* de que el relay aceptara el mensaje, aquí se marca como fallido un correo que sí salió. Se
prefiere ese falso negativo **visible** a un segundo correo a una persona real, porque un correo no
lo deshace ninguna transacción. Ningún escenario afirma que un mensaje rescatado no llegó nunca.

### FL-SND-030: el ciclo de despacho — cota, orden y arrastre

**Given**: existe `billing` con `invoiceready` publicada. Se encolan **250** mensajes con claves de
idempotencia distintas, en orden conocido. El relay acepta todo.

**When**: se ejecuta **un** ciclo de `dispatchQueuedMessages`.

**Then**:
1. El buzón recibe **como mucho 200** correos: la cota por ciclo es 200.
2. Los mensajes despachados son los **200 más antiguos por `requestedAt`**, no un subconjunto
   arbitrario: `listMessages?status=queued` devuelve los 50 restantes y todos son posteriores por
   `requestedAt` a cualquiera de los ya despachados.

**When**: se ejecuta un **segundo** ciclo

**Then**:
3. El buzón recibe los 50 restantes y `listMessages?status=queued` queda vacío: lo que no entra en
   un ciclo lo recoge el siguiente.
4. En total el buzón tiene **exactamente 250** correos: ningún mensaje se envió dos veces.

**When**: se encolan mensajes nuevos de `billing` y **después** se suspende la aplicación, con esos
mensajes todavía en `queued`

**Then**:
5. El ciclo los sigue enviando y acaban en `sent`: suspender cierra **la entrada**, no la cola. Lo
   que la suspensión impide es aceptar peticiones nuevas (FL-REQ-010).

**Notas de determinación**: los ciclos no se disparan, se **esperan** — `dispatchQueuedMessages`
solo tiene `schedule`, así que cada paso de este flujo espera al minuto siguiente. El solapamiento
de dos ciclos **no** se afirma aquí: no hay puerta por la que un ejecutor de caja negra dispare dos
a la vez (ver § Lo que no tiene escenario, y por qué).

## Lista de supresión

### FL-SUP-001: suprimir una dirección y su efecto sobre las peticiones

**Given**: existe `billing` con `invoiceready` publicada y **ninguna** dirección suprimida. Rol
`notifications-admin`.

**When**: `suppressAddress` — `POST /api/v1/applications/billing/suppressions`
```json
{ "address": "Queja@Example.COM", "reason": "complaint", "notes": "el cliente pidió la baja" }
```

**Then**:
1. Status `200`.
2. El cuerpo trae `id` (UUID), `applicationId` (el de `billing`), `address: "queja@example.com"`
   (normalizada), `reason: "complaint"`, `notes` tal cual, `status: "active"`, `suppressedAt`
   (instante UTC, por forma) y `releasedAt: null`.
3. El cuerpo no trae ningún campo adicional.

**When**: `requestNotification` (credencial de máquina del cliente `billing`) con
`recipients: ["QUEJA@example.com"]` y clave nueva

**Then**:
4. Status `422` con code `RECIPIENT_SUPPRESSED` — la comparación es insensible a mayúsculas.
5. No se encoló nada: `listMessages?applicationCode=billing` sigue vacío y el buzón también.

**When**: `requestNotification` con `recipients: ["queja@example.com", "ok@example.com"]`

**Then**:
6. Status `422` `RECIPIENT_SUPPRESSED`: basta **un** destinatario suprimido para rechazar la
   petición entera. No se encola un mensaje parcial para el resto.

**Orden de evaluación**:
1. Constraints del input → `VALIDATION_ERROR` (`400`).
2. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
3. La aplicación está en el alcance → `APPLICATION_FORBIDDEN` (`403`).

**Casos borde**:
- `reason: "inexistente"` → `400` `VALIDATION_ERROR`.
- `POST /api/v1/applications/inexistente/suppressions` → `404` `APPLICATION_NOT_FOUND`.

**Notas de determinación**: la supresión **no** afecta a los mensajes ya encolados para esa
dirección; solo cierra la entrada de peticiones nuevas.

### FL-SUP-010: la supresión es por aplicación, y suprimir dos veces no duplica

**Given**: existen `billing` y `shipping`, ambas con una plantilla publicada.
`queja@example.com` está suprimida en `billing`. Rol `notifications-admin`.

**When**: `suppressAddress` sobre `billing` con la misma dirección y `reason: "manual"`

**Then**:
1. Status `200` — suprimir una dirección ya `active` **no es un error**.
2. `listSuppressedAddresses` de `billing` devuelve **un** registro: no se creó un segundo.
3. Ese registro conserva `reason: "complaint"` y sus `notes` originales: no se pisó el motivo.

**When**: `requestNotification` con la credencial de máquina del cliente `shipping`, plantilla de
`shipping` y `recipients: ["queja@example.com"]`

**Then**:
4. Status `202`: la supresión vale **solo dentro de su aplicación**. Una dirección que rebotó para
   una aplicación puede seguir siendo válida para otra.
5. `listSuppressedAddresses` de `shipping` sigue **vacío**.

### FL-SUP-020: liberar una dirección

**Given**: existe `billing` con `queja@example.com` suprimida (`status: active`). Rol
`notifications-admin`.

**When**: `releaseAddress` — `DELETE /api/v1/applications/billing/suppressions/queja%40example.com`

**Then**:
1. Status `204` **sin cuerpo** (`output: void`).
2. `listSuppressedAddresses` de `billing` devuelve una página **vacía**: solo lista los `active`.
3. El registro **no se borró**: liberar es una transición (`active` → `released`), y lo demuestra el
   paso 6 de este mismo flujo.
4. `requestNotification` con `recipients: ["queja@example.com"]` y clave nueva responde `202`: la
   dirección vuelve a ser escribible.

**When**: `releaseAddress` sobre la misma dirección otra vez

**Then**:
5. Status `404` con code `SUPPRESSED_ADDRESS_NOT_FOUND` — liberar algo ya liberado **no** es un
   no-op válido, a diferencia de suprimir dos veces (FL-SUP-010).

**When**: `suppressAddress` sobre `queja@example.com` con `reason: "manual"` y
`notes: "volvió a quejarse"`

**Then**:
6. Status `200` con el **mismo `id`** que en el `Given`: se reactivó el registro existente
   (transición `released` → `active`), no se creó uno nuevo. La pareja aplicación+dirección es única
   para siempre.
7. `reason: "manual"` y `notes: "volvió a quejarse"` — al reactivar **sí** se toman los de esta
   petición.
8. `releasedAt` vuelve a `null` y `status` es `"active"`.

**Orden de evaluación**:
1. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
2. La aplicación está en el alcance → `APPLICATION_FORBIDDEN` (`403`).
3. Existe un registro `active` para esa dirección → `SUPPRESSED_ADDRESS_NOT_FOUND` (`404`).

**Casos borde**:
- Liberar una dirección **sin ningún registro** → `404` `SUPPRESSED_ADDRESS_NOT_FOUND`.
- La dirección viaja URL-encoded en el path; `QUEJA%40EXAMPLE.COM` alcanza el mismo registro.

### FL-SUP-030: listado de supresiones, orden total y filtro por motivo

**Given**: existe `billing`. Rol `notifications-auditor`. Se suprimen, **en el mismo instante** en
lo que al segundo se refiere, `zeta@example.com` (`hard-bounce`) y `alfa@example.com`
(`hard-bounce`); después `beta@example.com` (`manual`). Luego se libera `beta@example.com`.

**When**: `listSuppressedAddresses` — `GET /api/v1/applications/billing/suppressions`

**Then**:
1. Status `200` con **dos** elementos: `beta@example.com` está `released` y no aparece.
2. Orden exacto `alfa@example.com`, `zeta@example.com` — las dos empatan en `suppressedAt`, así que
   desempata `address` ascendente. Sin ese desempate el orden sería arbitrario.

**When**: `GET /api/v1/applications/billing/suppressions?reason=manual`

**Then**:
3. Status `200` con página **vacía**: el único `manual` está `released`.

**When**: `GET /api/v1/applications/billing/suppressions?reason=hard-bounce`

**Then**:
4. Status `200` con los dos elementos, en el mismo orden del paso 2.

### FL-SUP-040: dos supresiones de la misma dirección a la vez

**Given**: existe `billing` con la plantilla `invoiceready` publicada. `nueva@example.com` **no
tiene ningún registro** de supresión en `billing`. Rol `notifications-admin`.

**When**: se envían **a la vez** dos `suppressAddress` sobre `billing` con
`address: "nueva@example.com"` y `reason: "manual"`

**Then**:
1. Una de las dos responde `200` con el registro creado (`address: "nueva@example.com"`,
   `reason: "manual"`, `status: "active"`, `suppressedAt` informado, `releasedAt: null`).
2. La otra responde **`200` con ese mismo registro** (llegó cuando la ganadora ya había
   confirmado) **o** `409` con code `APPLICATION_ADDRESS_ALREADY_EXISTS` (las dos miraron a la
   vez, ninguna vio registro, y la que perdió chocó contra la clave natural
   `(application, address)`). La disyunción es **cerrada**: no hay tercer desenlace, y en
   particular el conflicto **no** sale con un código de unicidad genérico ni con
   `CONCURRENT_MODIFICATION` — la clave que choca es la de `SuppressedAddress`, no la raíz
   `Application`.
3. `listSuppressedAddresses` de `billing` devuelve **exactamente un** registro para
   `nueva@example.com`. El desenlace admite disyunción; el **efecto no**.

**When**: se entregan **a la vez** dos `reportEmailBounce` con `permanent: true` sobre el mismo
mensaje `sent` de `billing`, con `address: "rebote@example.com"` — dirección sin registro previo —,
con la credencial de máquina del cliente `mail-provider`

**Then**:
4. Una responde `204`; la otra responde `204` o `409` `APPLICATION_ADDRESS_ALREADY_EXISTS`, misma
   disyunción cerrada y por la misma puerta: un rebote permanente escribe en la lista de supresión.
5. `listSuppressedAddresses` de `billing` devuelve **exactamente un** registro para
   `rebote@example.com`, con `reason: "hard-bounce"` y `status: "active"`.

**Notas de determinación**: este código **solo** es observable en la carrera. La repetición
**secuencial** no lo produce nunca: suprimir una dirección ya `active` responde `200` sin crear un
segundo registro (FL-SUP-010) y un rebote permanente sobre una dirección ya suprimida responde
`204` igual. Un servidor que devolviera `409` en el caso secuencial rompe FL-SUP-010, y uno que
crease dos registros en la carrera rompe las aserciones 3 y 5 — que es lo que hace fallable a este
flujo y no a una redacción que solo dijera «no se duplica».

## Rebote asíncrono notificado por el proveedor

### FL-BNC-001: un rebote permanente suprime la dirección

**Given**: existe `billing` y un mensaje ya `sent` para `rebota@example.com` (recorrido de
FL-SND-001); se conoce su `id`. `billing` no tiene ninguna dirección suprimida. Se actúa con la
**credencial de máquina del cliente `mail-provider`** (único scope: `bounce:report`).

**When**: `reportEmailBounce` — `POST /api/v1/messages/{id}/bounces`
```json
{ "address": "Rebota@Example.COM", "permanent": true, "reason": "550 mailbox does not exist" }
```

**Then**:
1. Status `204` sin cuerpo.
2. `listSuppressedAddresses` de `billing` devuelve **un** registro con
   `address: "rebota@example.com"` (normalizada), `reason: "hard-bounce"`, `status: "active"` y
   `notes` con el motivo reportado.
3. `getMessage` sobre el `id` sigue devolviendo `status: "sent"` y su `sentAt`: **el rebote no
   cambia el estado del mensaje**. Es un hecho posterior a que el relay lo aceptara.
4. `requestNotification` con `recipients: ["rebota@example.com"]` y clave nueva → `422`
   `RECIPIENT_SUPPRESSED`.

**When**: `reportEmailBounce` sobre el mismo mensaje con `permanent: false`, dirección
`blando@example.com`

**Then**:
5. Status `204`.
6. `listSuppressedAddresses` sigue devolviendo **un** registro: un rebote **blando** se registra
   pero **no suprime** la dirección.
7. `requestNotification` con `recipients: ["blando@example.com"]` → `202`.

**Orden de evaluación**:
1. Constraints del input → `VALIDATION_ERROR` (`400`).
2. El mensaje existe → `MESSAGE_NOT_FOUND` (`404`).

**Casos borde**:
- `POST /api/v1/messages/{uuid-inexistente}/bounces` → `404` `MESSAGE_NOT_FOUND`.
- `permanent` ausente → `400` `VALIDATION_ERROR`.
- Sobre un mensaje en estado `queued` o `failed` → `204`: el rebote no exige que el mensaje esté en
  ningún estado concreto.
- Con la credencial de `billing` (sin `bounce:report`) → `403`.

### FL-BNC-010: un rebote permanente reactiva una supresión liberada

**Given**: existe `billing`, un mensaje `sent` para `rebota@example.com` (se conoce su `id`), y un
registro de supresión de esa dirección en estado `released` (suprimida y luego liberada a mano por
un operador). Credencial de máquina del cliente `mail-provider`.

**When**: `reportEmailBounce` con `address: "rebota@example.com"` y `permanent: true`

**Then**:
1. Status `204`.
2. `listSuppressedAddresses` de `billing` devuelve **un** registro con el **mismo `id`** que tenía:
   se reactivó el existente (transición `released` → `active`), no se creó otro.
3. `reason` es ahora `"hard-bounce"`, `notes` el motivo del rebote, `status: "active"` y
   `releasedAt` vuelve a `null`.
4. `requestNotification` con esa dirección → `422` `RECIPIENT_SUPPRESSED`.

**Notas de determinación**: reactivar va deliberadamente **contra** la decisión humana que liberó la
dirección. El proveedor acaba de confirmar que sigue muerta, y la reputación del remitente es
compartida por todas las aplicaciones del servicio. Quien la liberó puede volver a liberarla.

## Consulta de mensajes

### FL-MSG-001: consulta de un mensaje por su identificador

**Given**: existe `billing` con un mensaje `sent` (recorrido de FL-SND-001); se conoce su `id`. Rol
`notifications-auditor`.

**When**: `getMessage` — `GET /api/v1/messages/{id}`

**Then**:
1. Status `200` con el cuerpo completo de `EmailMessage`: `id`, `applicationId`, `idempotencyKey`,
   `recipients[]`, `renderedSubject`, `senderAddress`, `senderName`, `templateCode`,
   `templateVersionNumber`, `templateVersionId`, `status: "sent"`, `failureReason: null`, `requestedAt`, `sendingSince`
   (informado: el mensaje salió de `queued` alguna vez), `sentAt` y `personalDataPurgedAt: null`.
2. El cuerpo **no** trae `variableValues` — es `sensitive` y no sale en **ninguna** respuesta,
   aunque el solicitante sea auditor.
3. El cuerpo no trae ningún campo adicional.

**Casos borde**:
- `GET /api/v1/messages/{uuid-inexistente}` → `404` `MESSAGE_NOT_FOUND`.
- `GET /api/v1/messages/no-es-un-uuid` → `400` `VALIDATION_ERROR`.

### FL-MSG-010: consulta M2M por clave de idempotencia

**Given**: existen `billing` y `shipping`, cada una con un mensaje encolado con la clave
`"key-000001"` (la misma cadena en las dos: la clave es única **por aplicación**). Credencial de
máquina del cliente `billing` (scopes `notification:send`, `message:read`).

**When**: `findMessageByIdempotencyKey` — `GET /api/v1/notifications/billing/key-000001`

**Then**:
1. Status `200` con el cuerpo completo de `EmailMessage`, sin `variableValues`.
2. Su `applicationId` es el de `billing` — no el de `shipping`, aunque la clave coincida.

**When**: `GET /api/v1/notifications/shipping/key-000001` con la credencial de `billing`

**Then**:
3. Status `403` con code `APPLICATION_FORBIDDEN`: un cliente máquina solo consulta los mensajes de
   la aplicación que su credencial representa. **Se rechaza, no se responde vacío** — responder
   `404` filtraría si esa aplicación tiene o no un mensaje con esa clave.

**When**: `GET /api/v1/notifications/billing/clave-que-no-existe` con la credencial de `billing`

**Then**:
4. Status `404` con code `MESSAGE_NOT_FOUND`.

**When**: `GET /api/v1/notifications/billing/key-000001` con la credencial de `dispatch-only`
(solo `notification:send`)

**Then**:
5. Status `403`: leer los mensajes exige `message:read`, y pedir un envío no lo concede. Es lo que
   hace que `dispatch-only` exista como cliente separado.

**Orden de evaluación**:
1. La aplicación existe → `APPLICATION_NOT_FOUND` (`404`).
2. La credencial representa a esa aplicación → `APPLICATION_FORBIDDEN` (`403`).
3. Existe un mensaje suyo con esa clave → `MESSAGE_NOT_FOUND` (`404`).

**Casos borde**:
- `GET /api/v1/notifications/inexistente/key-000001` → `404` `APPLICATION_NOT_FOUND`.

### FL-MSG-020: listado de mensajes, orden total y filtros

**Given**: existe `billing` con `invoiceready` publicada. Rol `notifications-admin`. Se encolan
**dos** mensajes con claves y contenidos distintos y se espera a que el ciclo de despacho los deje
`sent`; **después** se encola un **tercero**, que sigue `queued` porque su ciclo no ha pasado
todavía. El estado mixto se consigue así y no de otra forma: el ciclo toma el lote entero de
`queued` en una pasada, así que no hay manera de dejar `queued` uno del medio.

**When**: `listMessages` — `GET /api/v1/messages`

**Then**:
1. Status `200` con sobre de paginación y 3 elementos.
2. Orden exacto: `requestedAt` **descendente** — el tercero (el `queued`) primero, el primero el
   último.
3. Cada elemento trae el cuerpo completo de `EmailMessage` y **ninguno** trae `variableValues`.

**When**: `GET /api/v1/messages?status=queued`

**Then**:
4. Status `200` con **un** elemento: el tercero. Es la aserción que discrimina — con los tres en
   el mismo estado, el filtro pasaría igual sin filtrar nada.

**When**: `GET /api/v1/messages?applicationCode=billing&templateCode=invoiceready`

**Then**:
5. Status `200` con los 3 elementos, mismo orden.

**When**: `GET /api/v1/messages?recipient=CLIENTE@Example.COM` (destinatario del primer mensaje)

**Then**:
6. Status `200` con **un** elemento, el primero: el filtro se normaliza a minúsculas antes de
   comparar, igual que se normalizaron los destinatarios al aceptar la petición.

**When**: `GET /api/v1/messages?requestedFrom={instante posterior a los tres}`

**Then**:
7. Status `200` con página **vacía**.

**When**: `GET /api/v1/messages?requestedFrom=2030-01-01T00:00:00Z&requestedTo=2020-01-01T00:00:00Z`

**Then**:
8. Status `200` con página **vacía**, **no** un error: una ventana temporal invertida devuelve
   vacío por diseño.

**Casos borde**:
- Sin filtros no se acota nada (paso 1).
- `?status=inexistente` → `400` `VALIDATION_ERROR`.

**Notas de determinación**: los dos primeros llegan a `sent` **esperando al ciclo**, nunca marcando
el estado por otra vía: `sendQueuedMessage` es `internal` y no hay endpoint que la dispare. El
tercero se encola después de esa espera, que es lo único que hace alcanzable el lote mixto.

### FL-MSG-030: el desempate del orden con mensajes del mismo instante

**Given**: existe `billing` con `invoiceready` publicada. Rol `notifications-admin`. Se encolan
**cinco** mensajes en la misma petición de tanda, de modo que sus `requestedAt` **empatan** al
segundo.

**When**: `GET /api/v1/messages?size=2` y después la página siguiente y la siguiente

**Then**:
1. Las tres páginas devuelven `2`, `2` y `1` elementos.
2. Los cinco `id` obtenidos a lo largo de las tres páginas son **todos distintos**: ninguno se
   repite entre páginas y ninguno se pierde. Es lo que garantiza el desempate por `id`, y lo que
   fallaría con un orden solo por `requestedAt`.
3. Concatenadas, las tres páginas contienen exactamente los cinco mensajes encolados.

## Retención de datos personales

### FL-RET-001: la purga borra los datos personales y conserva la guarda

**Given**: existe `billing` con un mensaje `sent` cuyo `requestedAt` tiene **más de 18 meses** y
`personalDataPurgedAt: null`, y otro mensaje `sent` **reciente**. Se conocen sus `id` y la clave de
idempotencia del antiguo (`"old-000001"`). Rol `notifications-admin`.

**When**: `purgeMessagePersonalData` — `POST /api/v1/messages/purge` (el mismo efecto que produce
el ciclo de las 03:00 UTC)

**Then**:
1. Status `204` sin cuerpo.
2. `getMessage` sobre el mensaje **antiguo** devuelve `200` — la fila se conserva — con
   `recipients: []` (vacío), `renderedSubject` vacío y `personalDataPurgedAt` informado.
3. Ese mismo cuerpo conserva `id`, `applicationId`, `idempotencyKey: "old-000001"`,
   `senderAddress`, `senderName`, `templateCode`, `templateVersionNumber`, `templateVersionId`,
   `status: "sent"`, `requestedAt`, `sendingSince` y `sentAt`: el estado y el desenlace **no se
   borran**, y tampoco la huella de cuándo se intentó. El remitente **no** es dato personal del
   destinatario y sus dos mitades sobreviven a la purga: sin ellas el registro dejaría de decir
   de quién dijo venir el correo.
4. `getMessage` sobre el mensaje **reciente** lo devuelve intacto, con sus `recipients` y su
   `renderedSubject`: el umbral es de 18 meses.
5. `GET /api/v1/messages?recipient={el destinatario del antiguo}` **no** lo devuelve: ya no conserva
   a quién se envió.

**When**: `requestNotification` (credencial de máquina del cliente `billing`) con
`idempotencyKey: "old-000001"` y contenido cualquiera

**Then**:
6. Status `202` reproduciendo el mensaje **purgado**, o `409` `IDEMPOTENCY_KEY_REUSED` si el
   contenido difiere del original — pero en **ningún** caso se encola un mensaje nuevo, y el buzón
   del relay no recibe nada. Es la razón exacta de no borrar la fila: la guarda contra el doble
   envío es permanente y sobrevive a la retención.

**When**: `purgeMessagePersonalData` otra vez

**Then**:
7. Status `204` y el mensaje antiguo **no cambia**: `personalDataPurgedAt` conserva su valor
   original. El ciclo es repetible sin daño porque solo toma mensajes sin esa marca.

**Casos borde**:
- Sin `retention:purge` (rol `notifications-auditor`) → `403`.
- Un servicio sin ningún mensaje antiguo → `204` sin efecto observable.

**Notas de determinación**: el umbral de 18 meses se calcula en **UTC**. Este flujo necesita un
mensaje con `requestedAt` de hace más de 18 meses, que ninguna suite puede alcanzar esperando: el
disparador manual `POST /api/v1/messages/purge` existe precisamente para poder ejercitar el efecto,
y el estado previo se prepara con datos de arranque cuyo `requestedAt` esté fuera de ventana. La
cota de 5000 mensajes por ciclo no se afirma con un número absoluto: lo que se afirma es que lo que
no entra en un ciclo lo recoge el siguiente (paso 7 aplicado sobre un conjunto mayor).

## Petición de envío por el canal de eventos

Estos flujos se escriben **contra el canal**, no contra la API: no hay cabecera que enviar ni
respuesta HTTP que leer. La identidad del solicitante sale de `metadata.source` del sobre.

### FL-EVT-001: un servidor registrado encarga un envío publicando en el canal

**Given**: existe `billing` (`status: active`) con la plantilla `invoiceready` y su versión `1`
activa. No existe ningún mensaje. El canal `notificationRequests` está vacío.

**When**: se publica en `notificationRequests` un mensaje `NotificationRequested` con sobre `keel`,
`metadata.source: "billing"` y payload
```json
{ "templateCode": "invoiceready", "idempotencyKey": "evt-000001",
  "recipients": ["cliente@example.com"],
  "variables": [ { "name": "customerName", "value": "Ana" },
                 { "name": "invoiceNumber", "value": "F-000123" } ] }
```

**Then**:
1. Se ejecuta `requestNotification` y `findMessageByIdempotencyKey` —
   `GET /api/v1/notifications/billing/evt-000001`, credencial de máquina del cliente `billing` —
   responde `200`.
2. Ese mensaje trae `applicationId` el de `billing`, resuelto de `metadata.source` y **no** de
   ningún campo del payload: el payload no lleva `applicationCode` y no hay forma de que el emisor
   lo elija.
3. Trae `status: "queued"`, `templateCode: "invoiceready"`, `templateVersionNumber: 1` y
   `renderedSubject` ya interpolado.
4. Tras ejecutar `dispatchQueuedMessages`, el buzón del relay contiene **un** correo con remitente
   `facturas@acme.com` — el de `billing`, no el de ninguna otra aplicación.

**Casos borde**:
- Un payload con un campo que el contrato no declara → se ignora (`unknownFields: ignore`) y el
  mensaje se procesa igual.
- `recipients: []` en el payload → el mensaje falla y acaba en la cola de descartes (FL-EVT-010).

### FL-EVT-010: la política de fallo distingue lo permanente de lo transitorio

**Given**: existe `billing` (`status: active`) con la plantilla `invoiceready` publicada. El canal
`notificationRequests` está vacío y la cola de descartes también.

**When**: se publica un `NotificationRequested` con `metadata.source: "billing"` y
`templateCode: "no-existe"` — un fallo **permanente**: reintentarlo no lo cura.

**Then**:
1. El mensaje llega a la **cola de descartes** sin consumir reintentos: el desenlace es
   `TEMPLATE_NOT_FOUND`, uno de los errores de negocio declarados, y esos no se reintentan.
2. No se encoló ningún mensaje: `listMessages?applicationCode=billing` sigue vacío.
3. El buzón del relay sigue vacío.

**When**: se publica un `NotificationRequested` cuyo procesamiento falla por una causa
**transitoria** (almacén indisponible), y la causa se mantiene

**Then**:
4. Se reintenta con backoff exponencial hasta **3** intentos (`initialDelayMs: 1000`,
   `maxDelayMs: 10000`).
5. Agotados los reintentos, el mensaje va a la **cola de descartes** (`deadLetter: true`).

**When**: se publica un `NotificationRequested` con `metadata.source: "aplicacion-no-registrada"`

**Then**:
6. El mensaje va a la **cola de descartes**, no se descarta en silencio (`onUnresolved: deadLetter`).
   El caso frecuente es un equipo que integra su servicio y olvida registrar su aplicación, y en
   silencio eso son correos que no salen sin que nada dé error en ningún sitio.
7. No se encoló ningún mensaje.

### FL-EVT-020: la reentrega del mismo mensaje no manda un segundo correo

**Given**: el estado final de FL-EVT-001: existe el mensaje de la clave `evt-000001`, ya despachado,
y el buzón del relay contiene **un** correo.

**When**: se **reentrega** el mismo mensaje del canal — mismo `metadata.eventId` del sobre `keel`,
mismo payload — y después se ejecuta `dispatchQueuedMessages`.

**Then**:
1. `listMessages?applicationCode=billing` sigue devolviendo **exactamente un** mensaje: el
   `metadata.eventId` del sobre corta la reentrega antes de tocar el dominio.
2. El buzón del relay sigue con **exactamente un** correo.

**When**: se publica un mensaje **nuevo** (`eventId` distinto) con el **mismo payload**, misma
`idempotencyKey` `evt-000001`, y se ejecuta `dispatchQueuedMessages`

**Then**:
3. `listMessages?applicationCode=billing` sigue devolviendo **un** mensaje: la `idempotencyKey` es
   la segunda barrera, y es la que cubre el caso que el `eventId` no cubre — un emisor que
   republique el mismo encargo con un sobre nuevo.
4. El buzón sigue con **un** correo.

**When**: se entregan **a la vez** dos copias del mismo mensaje

**Then**:
5. `listMessages?applicationCode=billing` devuelve **exactamente un** mensaje y el buzón **un**
   correo, sea cual sea la copia que gane. La entrega simultánea no es la reentrega con otras
   palabras: la secuencial encuentra la marca ya confirmada, y la simultánea cae en la ventana en
   la que todavía no lo está.

**Notas de determinación**: `publishing.reliability` es `best-effort`, así que ningún escenario
afirma que `EmailSent` o `EmailDeliveryFailed` **siempre** lleguen. Lo que se afirma en FL-SND-001 es su
contenido cuando el canal está disponible. Quien necesite certeza del desenlace
consulta `findMessageByIdempotencyKey` o `listMessages`, que es lo que el diseño promete.

## Autenticación, permisos y CORS

### FL-SEC-001: sin credencial y sin permiso

**Given**: existe `billing` con `invoiceready` publicada y un mensaje `sent`.

**Then** (una llamada por operación protegida, sin ninguna credencial):
1. `POST /api/v1/applications`, `PATCH /api/v1/applications/billing`,
   `POST /api/v1/applications/billing/suspend`, `POST /api/v1/applications/billing/reactivate`,
   `GET /api/v1/applications/billing`, `GET /api/v1/applications`,
   `POST /api/v1/applications/billing/templates`, `GET /api/v1/applications/billing/templates`,
   `GET .../templates/invoiceready`, `POST .../templates/invoiceready/retire`,
   `POST .../templates/invoiceready/versions`, `GET .../templates/invoiceready/versions`,
   `GET .../templates/invoiceready/versions/1`, `POST .../templates/invoiceready/versions/1/activate`,
   `GET /api/v1/messages`, `GET /api/v1/messages/{id}`,
   `POST /api/v1/applications/billing/suppressions`,
   `GET /api/v1/applications/billing/suppressions`,
   `DELETE /api/v1/applications/billing/suppressions/x%40y.com`,
   `POST /api/v1/messages/purge`, `POST /api/v1/notifications`,
   `GET /api/v1/notifications/billing/k-000001` y `POST /api/v1/messages/{id}/bounces`
   responden **todas** `401`. El acceso por defecto es `required`: sin regla explícita nada se
   sirve.

**Then** (con credencial válida pero sin el permiso exigido):
2. Con rol `notifications-auditor` (sin `application:write`): `POST /api/v1/applications` → `403`.
3. Con rol `notifications-auditor` (sin `template:write`):
   `POST /api/v1/applications/billing/templates` → `403`, y
   `POST .../templates/invoiceready/versions` → `403`.
4. Con rol `template-editor` (sin `message:read`): `GET /api/v1/messages` → `403` y
   `GET /api/v1/messages/{id}` → `403`.
5. Con rol `template-editor` (sin `suppression:write` ni `suppression:read`):
   `POST /api/v1/applications/billing/suppressions` → `403` y
   `GET /api/v1/applications/billing/suppressions` → `403`.
6. Con rol `notifications-auditor` (sin `retention:purge`): `POST /api/v1/messages/purge` → `403`.
7. Con rol `template-editor` (no es `notifications-admin`):
   `POST /api/v1/applications/billing/suspend` → `403`.
8. Con la credencial de máquina de `dispatch-only` (solo `notification:send`):
   `GET /api/v1/notifications/billing/k-000001` → `403` y
   `POST /api/v1/messages/{id}/bounces` → `403`.
9. Con la credencial de máquina de `mail-provider` (solo `bounce:report`):
   `POST /api/v1/notifications` → `403`.
10. Con la credencial de máquina de `billing` pero un token cuyo `aud` **no** nombra a este
    servicio: `POST /api/v1/notifications` → `403` (`validateAudience: true` responde 403, no 401).

**Notas de determinación**: una operación `level: service` no rechaza por sí sola un token de
usuario; la separación es por scopes. Ningún escenario afirma lo contrario.

### FL-SEC-010: el alcance por aplicación en las operaciones de administración

**Given**: existen `billing` y `shipping`, cada una con una plantilla publicada y una dirección
suprimida. Se actúa con el rol `template-editor` cuyo claim `applications` contiene **solo**
`billing`.

**Then**:
1. `GET /api/v1/applications/billing` → `200`.
2. `GET /api/v1/applications/shipping` → `403` `APPLICATION_FORBIDDEN`.
3. `GET /api/v1/applications/inexistente` → `404` `APPLICATION_NOT_FOUND`, no `403`: la
   comprobación de existencia precede a la de alcance, porque negar la existencia de algo que no
   existe no filtra nada.
4. `GET /api/v1/applications` → `200` con **un** elemento, `billing`: las de fuera del alcance no
   aparecen, y no se devuelve error.
5. `POST /api/v1/applications/shipping/templates` → `403`.
6. `GET /api/v1/applications/shipping/templates` → `403`.
7. `GET /api/v1/applications/shipping/templates/{code}` → `403`.
8. `POST /api/v1/applications/shipping/templates/{code}/versions` → `403`.
9. `POST /api/v1/applications/shipping/templates/{code}/versions/1/activate` → `403`.
10. `POST /api/v1/applications/shipping/templates/{code}/retire` → `403`.
11. `GET /api/v1/applications/shipping/templates/{code}/versions` → `403`.
12. `GET /api/v1/applications/shipping/templates/{code}/versions/1` → `403`.

**Given**: se actúa con el rol `notifications-admin`, cuyo token **no** lleva el claim acotado.

**Then**:
13. `GET /api/v1/applications` → `200` con **las dos** aplicaciones: admin y auditor están exentos
    del alcance por diseño, porque su función es transversal.
14. `GET /api/v1/applications/shipping` → `200`.
15. `POST /api/v1/applications/shipping/suppressions` → `200` y
    `GET /api/v1/applications/shipping/suppressions` → `200`.

### FL-SEC-020: el alcance por aplicación sobre los datos personales de los mensajes

**Given**: existen `billing` y `shipping`, cada una con **un** mensaje `sent` cuyo `recipients`
contiene una dirección distinta. Se conocen los dos `id`. Se actúa con el rol
`application-operator` cuyo claim `applications` contiene solo `billing`. Es el único rol que
combina alcance por aplicación con `message:read`: `template-editor` no lee correos y los dos
roles que sí (`notifications-admin`, `notifications-auditor`) están exentos del alcance por
diseño, así que sin él este escenario no tiene identidad que lo satisfaga.

**Then**:
1. `GET /api/v1/messages/{id de billing}` → `200` con `recipients` y `renderedSubject` informados.
2. `GET /api/v1/messages/{id de shipping}` → `403` `APPLICATION_FORBIDDEN`. Sin esta acotación, la
   respuesta expondría los destinatarios y el asunto — datos personales — de los clientes de otra
   aplicación.
3. `GET /api/v1/messages/{uuid inexistente}` → `404`, no `403`.
4. `GET /api/v1/messages` → `200` con **un** elemento, el de `billing`: la colección **filtra**, se
   pase o no el filtro `applicationCode`.
5. `GET /api/v1/messages?applicationCode=shipping` → `200` con página **vacía**: pedir
   explícitamente las de fuera del alcance no las devuelve.
6. `GET /api/v1/messages?recipient={la dirección del mensaje de shipping}` → `200` con página
   **vacía**: el filtro no es una vía lateral para leer lo de otra aplicación.

**Given**: se actúa con el rol `notifications-auditor`.

**Then**:
7. `GET /api/v1/messages` → `200` con **los dos** mensajes: el auditor es transversal por diseño.

### FL-SEC-030: CORS para la SPA de back-office

**Given**: el servicio en marcha. La política declara `allowCredentials: false`, `allowedHeaders`
`Authorization` y `Content-Type`, `exposedHeaders` `X-Correlation-Id` y `maxAgeSeconds: 3600`.

**When**: **preflight** — `OPTIONS /api/v1/applications` **sin credencial**, desde un origen web,
con `Access-Control-Request-Method: POST` y
`Access-Control-Request-Headers: Authorization, Content-Type`.

**Then**:
1. Status `2xx`: el preflight **no muere en la cadena de seguridad**, aunque la ruta esté protegida.
2. Se aceptan el método `POST` y las cabeceras `Authorization` y `Content-Type`.
3. Se anuncia un tiempo de cacheo de `3600` segundos.
4. **No** se anuncia soporte de credenciales (`allowCredentials: false`).

**When**: petición **normal cross-origin** — `GET /api/v1/applications` desde un origen web, con
token de rol `notifications-admin`.

**Then**:
5. Status `200` con el cuerpo esperado.
6. La respuesta anuncia `X-Correlation-Id` como cabecera **legible por el navegador**. Un servicio
   que contesta al preflight y no expone sus cabeceras pasa el primer escenario y rompe al cliente
   igual: por eso son dos.

**Notas de determinación**: los escenarios hablan de «un origen web» y de la **política**, nunca de
orígenes concretos — esos son despliegue y no están en el diseño.

## Lo que no tiene escenario, y por qué

Un hueco declarado es honesto; uno tapado con un escenario que nunca se ejecuta es peor, porque
además apaga la sospecha — y en el gate de generación aparece como `NO_EJERCITADO`, que dice «sin
cobertura» sin decir por qué. Estos tres caminos del servicio **no** producen escenario `FL-*`, y
conviene que esté escrito para que nadie se los invente.

### El rechazo síncrono del relay y la supresión automática

`sendQueuedMessage` declara que, **si y solo si** el rechazo del relay identifica un rebote duro
sobre una dirección **concreta**, esa dirección se añade a la lista de supresión de la aplicación.
El camino arranca con un rechazo del relay, y el buzón de prueba acepta SMTP incondicionalmente: no
hay forma determinista de provocarlo desde fuera del proceso. Como escenario `FL-*` quedaría
permanentemente sin ejercitar y haría inalcanzable el gate del 100%, que es lo que decide si la
generación avanza.

Se verifica con **prueba unitaria** del puerto de envío, con un doble que rechaza desglosando por
destinatario, y lo que esa prueba fija es: el mensaje queda `failed` con `failureReason` informado y
`sentAt: null`; se publica `EmailDeliveryFailed` y no `EmailSent`; la dirección aparece en la lista
con `reason: "hard-bounce"` y `status: "active"`; repetir sobre la misma dirección **no** duplica el
registro ni cambia su `suppressedAt`; una dirección `released` vuelve a `active`; y la colisión con
la lista **nunca** hace fallar el envío — no hay `409` en ningún sitio, porque la operación es
interna y no tiene cliente a quien devolvérselo.

Y la precondición es la que explica por qué es unitaria: un relay que rechaza **sin** desglosar por
destinatario —el caso normal, y el único que ofrece un relay de prueba— deja el mensaje en `failed`
y **no suprime nada**, que es el comportamiento correcto y no un fallo. Suprimir exige saber cuál
rebotó; adivinarla castigaría a destinatarios sanos del mismo mensaje. El camino determinista del
rebote duro es `reportEmailBounce`, que recibe dirección y permanencia explícitas del proveedor y sí
tiene escenarios: FL-BNC-001 y FL-BNC-010.

Si algún día el relay de prueba admite rechazar destinatarios concretos, esto vuelve a ser un
`FL-SND-010` sin cambiar nada de lo de arriba.

### Dos ciclos de despacho a la vez

La guarda contra el doble envío es la transición `queued` → `sending`, y su caso interesante es el
solapamiento de dos ciclos. Pero `dispatchQueuedMessages` solo tiene `schedule`: no hay puerta por
la que un ejecutor de caja negra dispare **un** ciclo, mucho menos dos simultáneos. Lo que sí es
observable —que ningún mensaje salga dos veces— lo afirma FL-SND-030 con ciclos consecutivos, y la
atomicidad del reclamo se verifica en **estático**: el reclamo condicional tiene que ser una
escritura que diga cuántas filas se llevó, no una lectura del estado de partida.

### La purga por antigüedad de 18 meses

`purgeMessagePersonalData` tiene dos disparadores, y solo uno es alcanzable: el manual, que cubre
FL-RET-001. Su condición de entrada real —«el mensaje lleva 18 meses»— no la fabrica ninguna suite
sin escribir el pasado directamente en el almacén, que es justo lo que un ejecutor de caja negra no
hace. Lo que se declara es la política (qué se purga, cada cuánto, qué se conserva, y que la guarda
de idempotencia sobrevive a la purga); el umbral no se simula.

## Paginación

### FL-PAG-001: primera página, siguiente, vacía y tope de tamaño

**Given**: existe `billing`. Rol `notifications-admin`. Se registran **25** aplicaciones más, de
modo que hay 26 en total con nombres que hacen el orden distinguible.

**When**: `GET /api/v1/applications` (sin parámetros)

**Then**:
1. Status `200` con **20** elementos: `defaultSize: 20`.
2. El sobre de paginación indica que hay más.

**When**: `GET /api/v1/applications?page=1&size=20`

**Then**:
3. Status `200` con los **6** restantes, y ninguno repite `code` con los de la primera página.

**When**: `GET /api/v1/applications?page=99&size=20`

**Then**:
4. Status `200` con página **vacía** y el sobre completo, no `404`.

**When**: `GET /api/v1/applications?size=1000`

**Then**:
5. Se sirven como mucho **100** elementos: `maxSize: 100` es el tope, y no se responde con error por
   pedir de más.

**Notas de determinación**: el sobre de paginación y los nombres `page`/`size` son contrato canónico
del DSL y no pueden divergir entre stacks; lo que este flujo fija son `defaultSize`, `maxSize` y el
comportamiento de la página vacía. El mismo comportamiento aplica a `listTemplates`,
`listTemplateVersions`, `listMessages` y `listSuppressedAddresses`, cuyos órdenes cubren
FL-TPL-060, FL-TPL-070, FL-MSG-020/030 y FL-SUP-030.
