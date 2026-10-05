# notifications — Documento de diseño

> specs/notifications v0.1.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`notifications` envía **correo transaccional** en nombre de los sistemas de una organización
(facturación, envíos…). Esos sistemas piden un correo a partir de una **plantilla** que gestiona
negocio, sin desplegar, y lo piden por dos puertas con el mismo contrato: **HTTP servidor a
servidor** y un **canal de eventos**. El servicio acepta la petición, la congela y la entrega de
forma asíncrona. Si falla, no reintenta: el fallo es terminal y se publica.

Todo el diseño se ordena alrededor de un riesgo: **enviar el mismo correo dos veces a una persona
real**. Por eso la idempotencia es permanente, hay un único emisor guardado por una transición de
estado y el rescate de envíos atascados nunca reenvía.

Es **monoinquilino**: hay una sola organización con un remitente, un catálogo de plantillas y una
lista de supresión. Los sistemas llamantes no son inquilinos; solo tienen su propio espacio de
claves de idempotencia.

Sirve a tres públicos:

- **Sistemas llamantes** (M2M): piden correos y reconcilian el estado de sus peticiones.
- **Back-office** (SPA, por rol): configura el remitente, gestiona plantillas y versiones, consulta
  correos y gestiona supresiones.
- **El proveedor de correo** (M2M): notifica rebotes.

Fuera de alcance, a propósito: adjuntos, localización por idioma, envíos programados a futuro,
seguimiento de aperturas y clics, canal automático de quejas y multitenancy.

## 2. Modelo de dominio

**Value types**

| Tipo | Forma | Significado |
|---|---|---|
| `EmailAddress` | string ≤254 con forma de correo | Se normaliza a minúsculas antes de guardar o comparar |
| `TemplateCode` | `^[a-z][a-z0-9-]{2,47}$` | Clave pública y estable de una plantilla |
| `IdempotencyKey` | `^[A-Za-z0-9._:-]{8,128}$` | Clave que asigna el llamante a cada petición |
| `VariableName` | `^[a-zA-Z][a-zA-Z0-9_]{0,63}$` | Se usa como `{{name}}`; sin expresiones ni literal escapable |
| `TemplateVariable` | `{ name, description, required }` | Contrato de una variable de plantilla |
| `VariableValue` | `{ name, value ≤10000 }` | Valor que da el llamante |
| `SenderSnapshot` | `{ address, name, replyToAddress }` | Remitente congelado en el mensaje |
| `FailureReason` | `delivery-rejected \| delivery-error \| stuck-in-sending \| recipient-suppressed` | Por qué falló un envío |
| `SuppressionReason` | `hard-bounce \| complaint \| manual` | Por qué se suprime una dirección |

**Agregados**

| Agregado | Entidades | Por qué juntas |
|---|---|---|
| `SenderSettings` | `SenderSettings` | Único (clave constante `key: default`): el remitente verificado del servicio |
| `EmailTemplate` | `EmailTemplate` → `EmailTemplateVersion` | Publicar, activar y retirar cambian a la vez la versión y `hasActiveVersion` de la plantilla |
| `EmailMessage` | `EmailMessage` | Lo congela todo al aceptarse: no depende de cambios posteriores en plantillas ni remitente |
| `SuppressedAddress` | `SuppressedAddress` | Un registro por dirección para siempre |

**Entidades y campos relevantes**

- `EmailTemplate`: `code` (único), `name`, `description`, `hasActiveVersion` (**computed** y
  persistido; se recalcula en la misma transacción que publish, activate y retire).
- `EmailTemplateVersion`: `versionNumber` (**generated**, correlativo desde 1, nunca reutilizado),
  `subject` ≤300, `htmlBody` ≤256 KiB, `textBody` ≤64 KiB (los dos obligatorios), `variables` ≤50,
  `publishedAt`. Su contenido es inmutable.
- `EmailMessage`: `requestedBy` (identidad del llamante, la fija el servidor), `idempotencyKey`,
  `recipients` ≤20, `renderedSubject`, `variableValues` (**sensitive**: no sale en ninguna
  respuesta), el snapshot `sender` + `templateCode` + `templateVersionNumber` +
  `templateVersionId`, y los instantes `requestedAt`, `sendingSince`, `sentAt`, `failedAt` y
  `personalDataPurgedAt`.
- `SuppressedAddress`: `address` (única sin distinguir mayúsculas), `reason`, `notes`,
  `suppressedAt`, `releasedAt`.

**Ciclos de vida**

| Entidad | Transiciones |
|---|---|
| `EmailTemplateVersion` | `active ⇄ archived` (como mucho una `active` por plantilla) |
| `EmailMessage` | `queued → sending → sent \| failed`; `queued → failed` (destinatario suprimido al enviar); `sent` y `failed` son terminales |
| `SuppressedAddress` | `active ⇄ released` |

## 3. Invariantes y reglas clave

- Todo marcador `{{name}}` del asunto y los cuerpos nombra una variable declarada por la versión.
- `sent ⇒ sentAt`; `failed ⇒ failureReason + failedAt`; `sending|sent ⇒ sendingSince`; `failed`
  sin `sendingSince` ⇒ `recipient-suppressed`.
- Con `personalDataPurgedAt`, `recipients`, `renderedSubject` y `variableValues` están vacíos, pero
  la fila y su clave se conservan para siempre.
- `active ⇒ releasedAt` vacío; `released ⇒ releasedAt` con valor.
- **Mismo contenido** = mismo `templateCode` + mismo conjunto de destinatarios normalizados + mismo
  conjunto de pares de variables declaradas, sin importar el orden. Sobre un mensaje ya purgado,
  basta el `templateCode`.
- Las variables no declaradas se descartan en silencio y las `required` se exigen.
- Los destinatarios conservan el orden de llegada y la primera aparición de cada duplicado; van
  todos en `To`.
- Render: en HTML se escapan solo `& < > " '`; en el asunto, los saltos de línea pasan a espacio;
  una variable opcional ausente se sustituye por cadena vacía. El motor nunca evalúa expresiones.
- El `Message-ID` del correo lleva como parte local el id del `EmailMessage`, que es lo que el
  proveedor devuelve en los rebotes.

## 4. Qué hace

**Remitente** — `configureSenderSettings` (`PUT /sender-settings`, crear o reemplazar) y
`getSenderSettings`.

**Plantillas** — `createTemplate` (sin contenido todavía), `publishTemplateVersion` (crea la
versión, la activa y archiva la anterior), `activateTemplateVersion` (revertir a una archivada; un
no-op si ya está activa), `retireTemplate` (archiva la activa y conserva el historial),
`getTemplate`, `listTemplates` (por `code`), `getTemplateVersion` y `listTemplateVersions` (de la
más nueva a la más antigua, sin cuerpos). Las creaciones responden `200`: los recursos se
direccionan por clave natural.

**Envío** — `requestNotification` acepta la petición (`202`, no promete la entrega) y la deja
`queued`. `dispatchQueuedMessages` corre cada minuto: primero rescata los mensajes atascados en
`sending` más de `sendingTimeoutMinutes` (15) y los da por fallidos **sin reenviarlos**; después
envía hasta 200 `queued` por antigüedad. `sendQueuedMessage` (interna) es **el único emisor**: su
guarda es la transición `queued → sending`, que se confirma antes de hablar con el relay.

**Consulta** — `getMessage` y `listMessages` (filtros por llamante, estado, plantilla, destinatario
y ventana temporal; orden `requestedAt` desc + `id`).

**Supresión** — `suppressAddress` (no-op si ya está activa, reactiva si está liberada),
`releaseAddress` (no-op si ya está liberada), `getSuppressedAddress` y `listSuppressedAddresses`
(solo las activas).

**Retención** — `purgeMessagePersonalData`: diaria a las 03:00 UTC y a demanda del admin. Vacía los
datos personales de los mensajes terminales más antiguos que `personalDataRetentionMonths` (18),
como mucho 5000 por ciclo, y se puede repetir sin efectos.

**Superficie servidor a servidor** (contrato propio con otros equipos)

| Operación | Ruta | Quién | Contrato |
|---|---|---|---|
| `requestNotification` | `POST /notifications` → `202` | `billing`, `shipping`, `dispatch-only` (`notification:send`) | Idempotencia permanente por `(requestedBy, idempotencyKey)`; errores `IDEMPOTENCY_KEY_REUSED`, `IDEMPOTENCY_KEY_IN_PROGRESS`, `SENDER_SETTINGS_NOT_CONFIGURED`, `TEMPLATE_NOT_FOUND`, `TEMPLATE_NOT_PUBLISHED`, `MISSING_TEMPLATE_VARIABLES`, `RECIPIENT_SUPPRESSED`, `DUPLICATE_TEMPLATE_VARIABLES` |
| `findMessageByIdempotencyKey` | `GET /notifications/{idempotencyKey}` | `billing`, `shipping` (`notification:read`) | Solo los mensajes del propio llamante; `404` = no aceptada |
| `reportEmailBounce` | `POST /messages/{messageId}/bounces` → `204` | `mail-provider` (`bounce:report`) | Un rebote `permanent` suprime con `hard-bounce`, aunque una persona la hubiera liberado; uno `transient` solo se registra en el log |

La misma petición entra también por el evento `NotificationRequested` (canal `notificationRequests`).

## 5. Fronteras e integraciones

- **Mensajería.** Consume `NotificationRequested` (`nature: request`, `source:
  any-registered-system`, envoltura Keel). La identidad sale de `metadata.source` y solo resuelve
  si coincide con un serviceClient con `notification:send`. Los rechazos de negocio y los emisores
  no resueltos van a la deadLetter; los fallos transitorios se reintentan 3 veces con espera
  exponencial. Publica `EmailSent` y `EmailDeliveryFailed` en `notificationEvents` por **outbox**,
  con un payload estable y sin destinatarios.
- **Correo.** SMTP, `multipart/alternative` html+text, sin adjuntos. El remitente y el reply-to
  salen del snapshot del mensaje (`source: data`), **sin fallback**. Las plantillas son dato
  (`source: data`) con variables declaradas.
- **Persistencia.** Relacional, transacción por agregado, bloqueo optimista en todas las raíces y
  auditoría completa (cuándo y quién) solo en el almacén. Las claves naturales son guardas:
  `(requestedBy, idempotencyKey)` contra el doble envío, `address` contra la doble supresión,
  `code`, `(emailTemplate, versionNumber)` y una versión activa por plantilla (índice único
  condicionado).
- **Seguridad.** OIDC para el back-office y client-credentials con `aud=notifications` para las
  máquinas. Roles: `notifications-admin` (todo), `template-editor` (plantillas; no lee correos),
  `notifications-operator` (lee correos y plantillas, gestiona supresiones) y
  `notifications-auditor` (solo lectura). CORS para la SPA.

## 6. Decisiones de diseño (qué / por qué)

**Monoinquilino, con varios llamantes.** El encargo original era multi-aplicación; el diseñador lo
redujo a una organización. Desaparecen `Application`, el claim `applications`,
`APPLICATION_FORBIDDEN` y la suspensión por aplicación. Lo único que se conserva de la identidad
del llamante es su **espacio de claves**. *Descartado*: mantener `Application` con una sola fila,
que traía un contrato entero que nadie ejercita.

**Remitente como dato editable, sin fallback.** El remitente se edita sin desplegar y se congela en
cada mensaje. Sin remitente configurado no se acepta ninguna petición: antes que enviar desde una
dirección que nadie verificó, no se envía. *Descartado*: dirección fija en el diseño, que ataría un
dominio de correo a un diseño reutilizable.

**Idempotencia permanente por `(requestedBy, idempotencyKey)`** (§3.2). La petición entra por HTTP y
por eventos (at-least-once), así que la clave viaja en el cuerpo y no caduca. Va por llamante para
que `billing` y `shipping` no choquen al usar ambos el id de un pedido. La clave sobrevive a la
purga: una petición repetida 18 meses después sigue sin enviar. *Descartado*: clave global; sin
idempotencia. El resto de commands no lleva clave: ninguno duplica efecto al repetirse.
`publishTemplateVersion` repetida crea una versión idéntica, inocua y visible en el historial.

**Como mucho una vez.** `queued → sending` se confirma antes de entregar al relay; si el proceso cae,
el rescate cierra el mensaje como `stuck-in-sending` y **no reenvía**. Si el relay acepta después
del rescate, el mensaje se queda `failed`: se acepta ese desajuste. Si el relay acepta a algún
destinatario, el mensaje queda `sent` y los rechazados llegan como rebotes; marcarlo `failed`
invitaría a reintentar y duplicaría el correo a quien ya lo recibió. Un envío `failed` se recupera
pidiéndolo con una clave nueva.

**Supresión recomprobada al enviar.** Un mensaje aceptado justo antes de un rebote no sale
(`recipient-suppressed`), para proteger la reputación del remitente.

**Outbox en vez de best-effort** (§3.1). El encargo pedía best-effort, pero también prometía que el
fallo se publica. Con best-effort, un `EmailDeliveryFailed` se pierde sin rastro si el broker no
está en el instante de confirmar. El coste es pequeño, porque la persistencia ya es relacional y
por agregado.

**Política de fallo de la suscripción** (§3.5): 3 reintentos exponenciales y deadLetter; la
identidad no resuelta va directa a deadLetter. Una petición perdida es un correo que nadie sabe que
faltó; la idempotencia hace segura la reinyección. *Descartado*: solo retry, o descartar en
silencio.

**Suplantación aceptada por escrito.** Por eventos, la identidad es política de desarrollo:
cualquier sistema que el broker deje publicar puede escribir el nombre de otro, pedir correos en
su nombre y ocupar sus claves. Se acepta mientras todos los emisores sean de la organización; la
señal para revisarlo es un emisor externo, que pasaría a un destino por emisor con ACL.

**Superficie M2M con operaciones propias** (§3.4): ninguna `audience: both`. El back-office no
envía correos; la reconciliación es por la clave del llamante, no por el id interno.

**Concurrencia: bloqueo optimista, 409** (§3.9). Dos publicaciones simultáneas no comparten número
ni dejan dos versiones activas, y dos réplicas del despachador no envían el mismo mensaje. Dos
altas simultáneas de la misma supresión responden `SUPPRESSED_ADDRESS_ALREADY_EXISTS`. *Descartado*:
último gana.

**Frontera transaccional por agregado** (§3.7). Ninguna operación escribe dos agregados; una
petición usa lo que leyó, y la recomprobación de supresión cubre la carrera que importa.

**Auditoría `all`, fuera del contrato** (§3.9b). Sirve para operar e investigar; el dominio ya
expone las fechas de negocio. La autoría registra la última escritura, así que un rebote que
reactiva una dirección tapa quién la había liberado.

**Sin caché** (§3.3). Lecturas de back-office y de reconciliación de poco volumen; un estado o una
supresión rancios llevarían a decisiones erróneas.

**Paginación offset 20/100 con orden total** (§3.8).

**Retención como parámetro** (`personalDataRetentionMonths`, meses de calendario, 18). Hace la purga
verificable en pruebas (valor 0).

**`complaint` solo manual.** No hay canal de quejas del proveedor; abrirlo es una evolución.

**Dirección en la ruta de supresiones.** Se acepta con codificación porcentual: el registro de
supresión se conserva indefinidamente por diseño, así que el log no añade una exposición nueva.

## 7. Ficha de reutilización

### Contrato estable vs adaptable

**Estable** (cambiarlo rompe a alguien): los codes de error; los eventos `EmailSent`,
`EmailDeliveryFailed` y `NotificationRequested` con sus payloads; las tres rutas M2M y los scopes
`notification:send`, `notification:read` y `bounce:report`; la sintaxis `{{name}}` (las plantillas
la llevan escrita); el `Message-ID` con el id del mensaje (el adaptador de rebotes depende de él).

**Adaptable**: el lote del despacho (200) y el de la purga (5000), los dos parámetros, los roles del
back-office, los filtros de `listMessages` y la política de reintentos de la suscripción.

Versionado según `docs/methodology.md`: patch para aclaraciones, minor para añadidos compatibles y
major para cualquier cambio de lo estable.

### Puntos de extensión típicos

- Canal de quejas del proveedor (otra operación M2M que suprima con `complaint`).
- Localización: un `locale` en `EmailTemplateVersion` y en la petición.
- Adjuntos: capa `storage` y `delivery.attachments: true`.
- Volver a multitenancy: reintroducir un agregado `Application` dueño de plantillas y supresiones,
  con `security.scoping`.
- Patrones reutilizables: la idempotencia por clave natural que sobrevive a la purga, el emisor
  único con rescate sin reenvío y el snapshot congelado al aceptar.

### Supuestos y limitaciones

- **Monoinquilino y volumen moderado.** Una sola organización. El despacho saca como mucho 200
  mensajes por minuto (~12.000/h); un volumen sostenido mayor exige revisar el lote o el cron.
- **Emisores de confianza.** Todos los sistemas que publican en `notificationRequests` son de la
  organización; la identidad por evento es política de desarrollo, no control técnico.
- **Fuera de alcance a propósito.** Sin adjuntos, sin localización por idioma (una plantilla, un
  idioma), sin envíos programados a futuro, sin seguimiento de aperturas o clics, sin canal
  automático de quejas, y un correo por petición con todos los destinatarios en `To`.

### Cómo reutilizarlo

`keel describe notifications` da el resumen mecánico. Si sirve **tal cual** —un servicio de correo
transaccional para una organización—, **adóptalo** con `keel registry get notifications`: llega con
sus derivados al día y se va directo a generar. Si hay que cambiar algo de lo estable (multitenancy,
localización, adjuntos), **derívalo** con `keel new <nuevo> --from registry:notifications` y
`/keel-design` entrevistará solo lo que cambia. Lo esperado es **adoptarlo**: lo adaptable cubre
la mayoría de los despliegues.

## Cobertura de comportamiento

`specs/notifications/validation-scenarios.md` cubre las 22 operaciones en 40 flujos: remitente,
plantillas, envío M2M (idempotencia secuencial y en carrera, precedencia de las 7 guardas,
credenciales), eventos (reentrega, DLQ y reinyección), despacho (supresión al enviar, rechazo total
y parcial, relay caído, render y escapado, dos réplicas, rescate y su cota, outbox), rebotes,
supresiones, consulta, purga y autorización. Sin escenario, y declarado: la guarda interna de
`sendQueuedMessage`, el cron de purga y la aceptación tardía del relay.
