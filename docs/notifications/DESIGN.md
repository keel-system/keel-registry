# notifications — Documento de diseño

> specs/notifications v0.3.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

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
| `SenderSnapshot` | `{ address, name, replyToAddress }` | Remitente congelado en el mensaje; buzón de la organización, no dato personal |
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
  `recipients` ≤20 (**personal**), `renderedSubject` (**personal**), `variableValues` (**sensitive**
  y **personal**: no sale en ninguna respuesta), el snapshot `sender` + `templateCode` +
  `templateVersionNumber` + `templateVersionId`, y los instantes `requestedAt`, `sendingSince`,
  `sentAt`, `failedAt` y `personalDataPurgedAt`.
- `SuppressedAddress`: `address` (única sin distinguir mayúsculas, **personal**), `reason`, `notes`
  (**personal**), `suppressedAt`, `releasedAt`.
- `SenderSettings`: `senderAddress`, `senderName`, `replyToAddress`. Es un buzón de la organización,
  no de una persona: no es dato personal, y `senderName` no debe nombrar a un empleado.

Los campos **personal** (DSL 2.28) no salen en ningún log ni traza del servidor; la marca se repite
en los inputs y en el payload de `NotificationRequested` que los llevan.

**Ciclos de vida**

| Entidad | Transiciones |
|---|---|
| `EmailTemplateVersion` | `active ⇄ archived` (como mucho una `active` por plantilla) |
| `EmailMessage` | `queued → sending → sent \| failed`; `queued → failed` (destinatario suprimido al enviar); `sent` y `failed` son terminales |
| `SuppressedAddress` | `active ⇄ released` |

## 3. Invariantes y reglas clave

- Toda aparición de `{{` en el asunto y los cuerpos abre un marcador `{{name}}` bien formado:
  `{{`, espacios U+0020 opcionales (ni tabuladores ni saltos de línea), un nombre de variable,
  espacios opcionales y `}}`. Solo `{{` abre un marcador; un `}` o `}}` suelto es texto. La triple
  llave `{{{name}}}` no existe: no hay forma de escribir un valor sin escapar. Rige desde la v0.2.0.
- Todo marcador `{{name}}` del asunto y los cuerpos nombra una variable declarada por la versión.
- **Checks** (DSL 2.31; la entidad los hace cumplir en cada mutación, `422 INVARIANT_VIOLATED`):
  - `EmailMessage`: `sent ⇒ sentAt`; `failed ⇒ failureReason + failedAt`; `sending|sent ⇒
    sendingSince`; `queued ⇒` sin `sendingSince`, `sentAt`, `failedAt` ni `failureReason`; un
    `failureReason` distinto de `recipient-suppressed` exige `sendingSince`; exactamente uno de
    `recipients` y `personalDataPurgedAt`, y exactamente uno de `renderedSubject` y
    `personalDataPurgedAt`.
  - `SuppressedAddress`: `active ⇒ releasedAt` vacío; `released ⇒ releasedAt` con valor.
- Con `personalDataPurgedAt`, además, `variableValues` está vacío; la fila y su clave se conservan
  para siempre.
- **Mismo contenido** = mismo `templateCode` + mismo conjunto de destinatarios normalizados + mismo
  conjunto de pares de variables declaradas, sin importar el orden. Sobre un mensaje ya purgado,
  basta el `templateCode`.
- Las variables no declaradas se descartan en silencio y las `required` se exigen.
- Los destinatarios conservan el orden de llegada y la primera aparición de cada duplicado; van
  todos en `To`.
- Render: de una sola pasada (un valor insertado nunca se reinterpreta, aunque contenga
  `{{otra}}`); en HTML se escapan solo `& < > " '`; en el asunto, cada salto de línea (`\r\n`, `\n`
  o `\r` suelto) pasa a un espacio; un asunto renderizado vacío o solo con espacios se rechaza
  (`EMPTY_RENDERED_SUBJECT`) y uno de más de 300 caracteres también, sin recortarlo; una variable
  opcional ausente se sustituye por cadena vacía. El motor nunca evalúa
  expresiones. Un `{{` mal formado de una versión anterior a la v0.2.0 se renderiza como texto
  literal, escapado.
- El `Message-ID` del correo lleva como parte local el id del `EmailMessage`, que es lo que el
  proveedor devuelve en los rebotes.

## 4. Qué hace

**Remitente** — `configureSenderSettings` (`PUT /sender-settings`, crear o reemplazar) y
`getSenderSettings`.

**Plantillas** — `createTemplate` (sin contenido todavía), `publishTemplateVersion` (crea la
versión, la activa y archiva la anterior; rechaza marcadores mal formados con
`MALFORMED_TEMPLATE_MARKER`), `activateTemplateVersion` (revertir a una archivada; un no-op si ya
está activa; no reactiva una versión anterior a la v0.2.0 con marcadores mal formados),
`retireTemplate` (archiva la activa y conserva el historial),
`getTemplate`, `listTemplates` (por `code`; el cliente puede ordenar por `code` o `name`),
`getTemplateVersion` y `listTemplateVersions` (de la más nueva a la más antigua, sin cuerpos; el
cliente puede ordenar por `versionNumber` o `publishedAt`). Las creaciones responden `200`: los recursos se
direccionan por clave natural.

**Envío** — `requestNotification` acepta la petición (`202`, no promete la entrega) y la deja
`queued`. `dispatchQueuedMessages` corre cada minuto: primero rescata los mensajes atascados en
`sending` más de `sendingTimeoutMinutes` (15, mínimo 2) y los da por fallidos **sin reenviarlos**; después
envía hasta 200 `queued` por antigüedad. `sendQueuedMessage` (interna) es **el único emisor**: su
guarda es la transición `queued → sending`, que se confirma antes de hablar con el relay.

**Consulta** — `getMessage` y `listMessages` (filtros por llamante, estado, plantilla, destinatario
y ventana temporal; orden `requestedAt` desc + `id`; el cliente puede ordenar por `requestedAt` o
`templateCode`).

**Supresión** — `suppressAddress` (no-op si ya está activa, reactiva si está liberada),
`releaseAddress` (no-op si ya está liberada), `getSuppressedAddress` y `listSuppressedAddresses`
(solo las activas; el cliente puede ordenar por `suppressedAt`).

En los cuatro listados, `?sort=` solo admite las columnas declaradas (`output.sortable`, DSL 2.32);
cualquier otra, o otra dirección, es `400`. El `id` desempata siempre.

**Retención** — `purgeMessagePersonalData`: diaria a las 03:00 UTC y a demanda del admin. Vacía los
datos personales de los mensajes terminales más antiguos que `personalDataRetentionMonths` (18),
como mucho 5000 por ciclo, y se puede repetir sin efectos.

**Superficie servidor a servidor** (contrato propio con otros equipos)

| Operación | Ruta | Quién | Contrato |
|---|---|---|---|
| `requestNotification` | `POST /notifications` → `202` | `billing`, `shipping`, `dispatch-only` (`notification:send`) | Idempotencia permanente por `(requestedBy, idempotencyKey)`; errores `IDEMPOTENCY_KEY_REUSED`, `IDEMPOTENCY_KEY_IN_PROGRESS`, `SENDER_SETTINGS_NOT_CONFIGURED`, `TEMPLATE_NOT_FOUND`, `TEMPLATE_NOT_PUBLISHED`, `MISSING_TEMPLATE_VARIABLES`, `EMPTY_RENDERED_SUBJECT`, `RENDERED_SUBJECT_TOO_LONG`, `RECIPIENT_SUPPRESSED`, `DUPLICATE_TEMPLATE_VARIABLES` |
| `findMessageByIdempotencyKey` | `GET /notifications/{idempotencyKey}` | `billing`, `shipping` (`notification:read`) | Solo los mensajes del propio llamante; `404` = no aceptada |
| `reportEmailBounce` | `POST /messages/{messageId}/bounces` → `204` | `mail-provider` (`bounce:report`) | Un rebote `permanent` suprime con `hard-bounce`, aunque una persona la hubiera liberado; uno `transient` solo se registra en el log. `notes` = id del mensaje + diagnostic, recortando el diagnostic para no pasar de 1000 |

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
  auditoría completa (cuándo y quién) solo en el almacén. Retención: la purga propia de
  `EmailMessage` (no el primitivo `retention`) y conservación indefinida de `SuppressedAddress`. Las claves naturales son guardas:
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

**Paginación offset 20/100 con orden total** (§3.8). Desde la v0.3.0 el cliente ordena por las
columnas que ofrece cada pantalla: plantillas por `code` y `name`, versiones por `versionNumber` y
`publishedAt`, mensajes por `requestedAt` y `templateCode`, supresiones por `suppressedAt`. Solo
columnas con índice que las sirva; `address` no se ofrece porque es dato personal, y ningún enum
(`status`, `reason`), porque no tiene orden de negocio. `?sort=name` ordena por punto de código, sin
colación de idioma: el back-office busca plantillas por `code`. *Descartado*: listados sin `?sort`;
ofrecer todas las columnas con un índice por cada una.

**Retención como parámetro** (`personalDataRetentionMonths`, meses de calendario, 18). Hace la purga
verificable en pruebas (valor 0).

**Marcadores estrictos, sin migrar lo publicado** (v0.2.0). El servidor generado rechazaba
`{{{name}}}` sin que el diseño lo dijera, así que dos generadores podían divergir. Se declara
`MALFORMED_TEMPLATE_MARKER` (422) para todo `{{` que no abra un marcador bien formado, con un error
propio porque `UNDECLARED_TEMPLATE_VARIABLES` engañaría al editor si la variable sí está declarada.
Solo el espacio U+0020 cuenta dentro de las llaves, para que «espacio» signifique lo mismo en
cualquier stack. Las versiones ya publicadas no se migran: no se pueden reactivar y, si siguen
activas, su `{{` mal formado va literal y escapado. *Descartado*: rechazarlo como
`UNDECLARED_TEMPLATE_VARIABLES`, tratar las llaves como literales, aceptar cualquier blanco, o
archivar las versiones antiguas con un paso de migración. Se versiona como **minor** porque el
error nuevo solo rechaza contenido cuyo resultado en la v0.1.0 no estaba definido: ningún cliente
dependía de un comportamiento contractual.

**Asunto largo: rechazar, no recortar** (v0.2.0). Con valores de hasta 10000 caracteres el asunto
renderizado podía pasar de 300. `requestNotification` responde `RENDERED_SUBJECT_TOO_LONG` (422):
el llamante se entera en el momento y ningún correo sale con el dato importante cortado.
*Descartado*: recortar a 300 con `…`.

**Sustitución de una sola pasada** (v0.2.0). Un valor que contiene `{{otra}}` sale literal: así un
valor no puede inyectar el de otra variable, y el correo es el mismo en cualquier motor.

**Datos personales marcados, con la retención que ya tenía** (v0.3.0). La clasificación que la
prosa ya hacía (la purga, la exclusión de `address` del orden) se declara con `personal` en los
campos, inputs y payloads que llevan direcciones, asuntos, valores de variables o notas: ningún log
ni traza del servidor los lleva. El remitente no se marca: es un buzón de la organización. La
retención no pasa al primitivo `retention`, porque no se acota a mensajes terminales, no admite
meses y su anonimización no fija `personalDataPurgedAt`, del que dependen los checks y la
comparación de repeticiones. Las supresiones se conservan siempre: borrarlas reabriría el envío a
quien rebotó o pidió la baja. *Descartado*: migrar a `retention`; no clasificar.

**Asunto vacío: rechazar** (v0.3.0). Un subject que es solo el marcador de una variable opcional
podía renderizar `""` y romper la invariante sin error declarado. `requestNotification` responde
`EMPTY_RENDERED_SUBJECT` (422), antes de comprobar la longitud. *Descartado*: rechazarlo al publicar
(no se sabe qué variables llegarán) o admitir correos sin asunto.

**Invariantes de estado como checks** (v0.3.0). Las de `EmailMessage` y `SuppressedAddress` pasan a
`checks`, que la entidad hace cumplir en cada mutación en vez de solo donde un escenario las pide.
En prosa queda lo que el vocabulario no expresa: inmutabilidad, marcadores y `variableValues` vacío
tras la purga.

**Diagnostic largo: recortar, no rechazar** (v0.3.0). `notes` es el id del mensaje, un espacio y el
diagnostic; si no cabe en 1000 caracteres se recorta el diagnostic por el final. Un rebote permanente
siempre suprime. *Descartado*: subir la cota de `notes` o bajar la del `diagnostic`, que rompería al
proveedor de correo.

**El formulario del remitente: gana la última escritura** (§3.12, v0.3.0). `/sender-settings` es un
recurso único sin id en la ruta, y el DSL solo admite `If-Match` sobre una ruta que nombra su
recurso. Se cambia rara vez; el bloqueo optimista y la autoría cubren lo que importa. Tampoco tiene
historial: cada mensaje congela el remitente con el que se aceptó. *Descartado*: mover la ruta a
`/sender-settings/{id}` (major).

**`?recipient=` en los logs de acceso** (v0.3.0). El filtro por destinatario de `listMessages` lleva
una dirección en la URL. Se acepta por ser back-office interno con logs de acceso restringido; pasar
a `POST /messages/search` rompería el contrato. Ese filtro sigue con consulta propia (casa con un
elemento de la lista `recipients`); la ventana temporal se declara con `filter`.

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
back-office, los filtros de `listMessages`, la política de reintentos de la suscripción y las
columnas ordenables (añadir una es minor; quitarla, major).

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
plantillas (marcadores bien formados y su precedencia), envío M2M (idempotencia secuencial y en
carrera, precedencia de las 9 guardas, asunto vacío y demasiado largo, supresión comprobada en todos
los destinatarios, credenciales), eventos (reentrega, DLQ y reinyección), despacho (supresión al
enviar, rechazo total y parcial, relay caído, render de una pasada y escapado, dos réplicas, rescate
y su cota, outbox), rebotes (con diagnostic recortado), supresiones, consulta, orden pedido por el
cliente en los cuatro listados, purga y autorización. Sin escenario, y declarado: la guarda interna
de `sendQueuedMessage` (incluida la carrera entre supresión y reclamo), el rechazo y el render de
versiones anteriores a la v0.2.0, el cron de purga, que los datos personales no salgan en logs, los
checks de dominio (ninguna operación los rompe), la comprobación de supresiones en lote y la
aceptación tardía del relay.
