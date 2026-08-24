# notifications — Documento de diseño

> specs/notifications v1.5.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`notifications` emite el **correo transaccional** de las aplicaciones de la organización. Un
servidor que necesita avisar a una persona —una factura emitida, un envío despachado— no compone el
correo ni habla con el proveedor de correo: le pide a este servicio que mande una plantilla con unos
valores, y el servicio se encarga del resto.

Resuelve tres problemas que, resueltos por separado en cada servidor, se resuelven mal:

- **El contenido lo escribe negocio, no desarrollo.** Las plantillas viven en la base de datos del
  servicio y se editan por API, versionadas: cambiar el texto de un aviso no es un despliegue.
- **La reputación del remitente es un recurso compartido.** Insistir sobre una dirección que rebotó
  duro perjudica la entregabilidad de *todos* los consumidores, así que la lista de supresión no
  puede quedar en manos de cada llamante: la mantiene el servicio y se aplica en cada aceptación.
- **Un correo enviado no lo deshace nada.** La guarda contra el doble envío es del servicio, es
  permanente y sobrevive incluso a la purga de datos personales.

Sirve a dos públicos: **personas** que gestionan aplicaciones y plantillas desde un back-office, y
**servidores** que piden envíos con credencial de máquina o publicando en un canal de eventos.

**Queda fuera a propósito**: el reintento de envíos fallidos (el fallo es terminal), el seguimiento
de aperturas y clics, la programación de envíos y las campañas a listas, los adjuntos, y cualquier
canal que no sea el correo. Ver § 7.3.

## 2. Modelo de dominio

### Value types

Pocos y con significado, en lugar de repetir constraints:

| Tipo | Significado |
|---|---|
| `EmailAddress` | Dirección de correo. Se guarda y compara **en minúsculas** en todo el servicio. |
| `ApplicationCode` | Identificador estable y legible de una aplicación consumidora. Es lo que viaja en eventos y peticiones M2M, no el uuid. |
| `TemplateCode` | Identificador estable de una plantilla dentro de su aplicación; es lo que el consumidor nombra al pedir un envío. |
| `IdempotencyKey` | Con lo que quien pide el envío distingue una petición nueva de un reintento de la misma. |
| `TemplateVariable` / `TemplateVariableValue` | Value objects: la variable que una versión **declara** y el valor que el solicitante **aporta**. Separarlos es lo que permite validar una petición antes de renderizar. |

Enums nominales: `ApplicationStatus`, `TemplateVersionStatus`, `MessageStatus`, `SuppressionReason`,
`SuppressionStatus`.

### Entidades

**`Application`** — la aplicación o servidor autorizado a pedir envíos. Posee sus plantillas y
aporta el **remitente verificado** desde el que salen sus correos (`senderAddress`, `senderName`,
`replyToAddress` opcional). `code` es único en todo el servicio y no cambia nunca.

**`EmailTemplate`** — el contenedor estable de una plantilla: `code` (único dentro de su
aplicación), `name`, `description` y `hasActiveVersion`. El contenido **no** vive aquí.

- `hasActiveVersion` es **`computed`** y se guarda: se recalcula dentro de la misma transacción que
  publicar, reactivar o retirar. Es lo que distingue una plantilla en circulación de una retirada
  sin duplicar el estado que ya viven las versiones.

**`EmailTemplateVersion`** — el contenido concreto e **inmutable**: `versionNumber`, `subject`,
`htmlBody` (≤ 256 KB), `textBody` (≤ 64 KB) y las `variables` que declara. Publicar contenido nuevo
crea una versión nueva; la anterior se archiva y sigue siendo consultable.

- `versionNumber` es **`computed`**: el mayor existente más uno, empezando en 1, nunca reutilizado.
  Lo calcula el servidor, jamás el cliente.

**`EmailMessage`** — el registro de un correo que el servicio aceptó emitir. Es la unidad de
trazabilidad **y** la guarda contra repeticiones. Congela en el momento de aceptar todo lo que
podría cambiar después: `senderAddress`, `senderName`, `templateCode`, `templateVersionNumber`,
`templateVersionId` y el `renderedSubject` ya resuelto.

- `senderAddress` y `senderName` son **`computed`** y se congelan **juntos**, copiados de la
  aplicación al aceptar la petición. Son las dos mitades de una misma cabecera `From`: componerla
  con la dirección de entonces y el nombre de ahora produce un remitente que nunca existió, y el
  registro dejaría de decir qué leyó el destinatario. `senderName` es opcional —una aplicación
  puede no declarar nombre visible— y entonces el correo sale solo con la dirección.

- `variableValues` es **`sensitive`**: puede contener datos personales, así que no sale en ninguna
  respuesta y se borra en la purga.
- `failureReason`, `requestedAt`, `sendingSince`, `sentAt` y `personalDataPurgedAt` son
  **`generated`**.
- `sendingSince` es la marca del intento, no de la petición: se estampa al pasar a `sending` y se
  conserva después. Es el único dato observable que distingue «lo acaban de tomar» de «el proceso
  que lo tomó murió hace media hora», y sin él el rescate por antigüedad no tendría criterio.

**`SuppressedAddress`** — la dirección a la que una aplicación no debe volver a escribir, con su
`reason` (`hard-bounce`, `complaint`, `manual`). Liberarla **no borra el registro**: lo conserva
para auditoría y para poder reactivarlo si la dirección vuelve a rebotar.

### Agregados

| Agregado | Raíz | Entidades internas | Por qué cambian juntas |
|---|---|---|---|
| `Application` | `Application` | `EmailTemplate`, `EmailTemplateVersion` | Publicar una versión **archiva la anterior en la misma transacción**: la invariante «como mucho una versión activa» no se puede sostener a través de dos fronteras. |
| `EmailMessage` | `EmailMessage` | — | Cada mensaje es su propia frontera; referencia a la aplicación y a la versión por id. |
| `SuppressedAddress` | `SuppressedAddress` | — | Se escribe desde el desenlace de un envío —otra transacción que la del agregado `Application`— y se consulta al aceptar cada petición. |

### Ciclos de vida

| Entidad | Estados | Transiciones |
|---|---|---|
| `Application` | `active` (inicial) ⇄ `suspended` | `suspendApplication` / `reactivateApplication` |
| `EmailTemplateVersion` | `active` (inicial) ⇄ `archived` | `publishTemplateVersion`, `activateTemplateVersion`, `retireTemplate` |
| `EmailMessage` | `queued` (inicial) → `sending` → `sent` \| `failed` | `sendQueuedMessage`; el rescate de `dispatchQueuedMessages` lleva `sending` → `failed` |
| `SuppressedAddress` | `active` (inicial) ⇄ `released` | `releaseAddress`, `suppressAddress`, `reportEmailBounce`, `sendQueuedMessage` (supresión automática) |

`sent` y `failed` son **terminales**: no hay vuelta atrás por diseño (ver § 6).

## 3. Invariantes y reglas clave

- Una aplicación `suspended` **no puede originar nuevos envíos**, pero sí se sigue administrando:
  suspender cierra la entrada, no el back-office. Sus mensajes ya encolados se siguen enviando.
- Una plantilla tiene **como mucho una versión `active`**, y una sin ninguna no puede usarse para
  enviar.
- El contenido de una versión **no se modifica una vez creada**; su número es único en su plantilla
  y nunca se reutiliza.
- Toda variable referenciada en el asunto o en los cuerpos está **declarada** en `variables`, y toda
  variable `required` debe venir con valor no vacío en la petición.
- **La pareja aplicación + clave de idempotencia es única en todo el servicio, para siempre**, y se
  conserva aunque se purguen los datos personales del mensaje.
- La versión de plantilla y el remitente de un mensaje **no cambian** aunque después se publique una
  versión nueva o se cambie el remitente de la aplicación.
- Un mensaje que ha salido alguna vez de `queued` tiene siempre `sendingSince` informado, y uno en
  `queued` nunca lo tiene. Un `sent` tiene `sentAt`; un `failed`, `failureReason`.
- La pareja aplicación + dirección de `SuppressedAddress` es única **para siempre**: liberar no abre
  hueco para un segundo registro, y una dirección liberada que vuelve a rebotar **reactiva el mismo
  registro**.
- La supresión vale **solo dentro de su aplicación**: una dirección que rebotó para una puede seguir
  siendo válida para otra.
- En la edición de una aplicación, **omitir un campo lo conserva y enviarlo presente y en blanco lo
  borra**; pero solo son vaciables los que la aplicación puede no tener (`replyToAddress`,
  `senderName`). `name` y `senderAddress` son requeridos en el dominio: enviarlos en blanco no los
  borra, se rechaza.
- La aplicación solicitante se resuelve **de la identidad del llamante**, nunca de un dato que él
  elija.
- Todo identificador de negocio y **toda dirección de correo** se normaliza a minúsculas antes de
  comparar, deduplicar o guardar.

## 4. Qué hace

### Superficie de usuarios (back-office)

**Aplicaciones consumidoras** — `registerApplication`, `updateApplication`, `suspendApplication`,
`reactivateApplication`, `getApplication`, `listApplications`.

**Plantillas** — `createTemplate` (el contenedor, todavía sin contenido publicable),
`publishTemplateVersion` (contenido nuevo, archivando lo anterior), `activateTemplateVersion`
(revertir a una versión anterior sin recomponerla a mano), `retireTemplate` (sacar de circulación
conservando la historia), `getTemplate`, `listTemplates`, `getTemplateVersion`,
`listTemplateVersions`.

**Correos emitidos** — `getMessage` y `listMessages`. Este último es el **único camino para
descubrir los envíos fallidos**, que el servicio no reintenta, y su filtro `recipient` es lo que
responde a «¿le llegó el correo a esta persona?». Los dos llevan datos personales, así que los dos
se acotan por el claim `applications`: `getMessage` devuelve `APPLICATION_FORBIDDEN` fuera de
alcance y `listMessages` **filtra en silencio**, se pase o no el filtro `applicationCode`.

**Lista de supresión** — `suppressAddress`, `releaseAddress`, `listSuppressedAddresses`.

**Retención** — `purgeMessagePersonalData` (`schedule` diario a las 03:00 UTC, más disparo manual).

**Despacho** — `dispatchQueuedMessages` (`schedule` cada minuto) invoca `sendQueuedMessage`
(`internal: true`), la **única operación del servicio que produce un correo real**.

### Superficie servidor-a-servidor

Tres operaciones **propias**, con contrato propio: ninguna se comparte con la superficie de usuarios
(`audience: both` no se usa en ningún endpoint).

| Operación | Endpoint | Scope | Quién la consume |
|---|---|---|---|
| `requestNotification` | `POST /api/v1/notifications` (`202`) | `notification:send` | `billing`, `shipping`, `dispatch-only` |
| `findMessageByIdempotencyKey` | `GET /api/v1/notifications/{applicationCode}/{idempotencyKey}` | `message:read` | `billing`, `shipping` |
| `reportEmailBounce` | `POST /api/v1/messages/{messageId}/bounces` (`204`) | `bounce:report` | `mail-provider` |

`requestNotification` es además la operación que dispara la suscripción `NotificationRequested`: el
mismo contrato sirve las dos vías, HTTP y canal de eventos.

**Idempotencia**: solo `requestNotification` la declara (`keySource: payload-field` sobre
`idempotencyKey`, **sin TTL**: la guarda es permanente). Las operaciones de back-office no la llevan a propósito (§ 6).
**Caché**: ninguna query la declara (§ 6).

## 5. Fronteras e integraciones

El servicio **no depende de ningún otro servidor**: no declara capa `dependencies` ni
`http-clients`. El relay SMTP no es una dependencia —no publica contrato ni es un servicio Keel—,
es su salida propia por la capa `mail`.

### Correo saliente (`mail`)

Transporte SMTP con las **dos partes**, `html` y `text` — un HTML sin alternativa textual levanta
sospechas en los filtros antispam y eso no falla en ninguna prueba. Sin adjuntos.

- **Remitente**: `source: data`, de `Application.senderAddress` y `Application.senderName`,
  **sin `fallback`**. Falla cerrado. Las dos mitades se copian al mensaje al aceptar la petición,
  y es lo copiado —no lo vigente en la aplicación— lo que compone el `From` al enviar.
- **Plantillas**: `source: data`, con `declaredVariables: true`. El original vive en la base de
  datos y lo edita negocio sin desplegar. Como el cuerpo es entrada de origen externo, no puede
  renderizarse con un motor que evalúe expresiones arbitrarias.

### Eventos (`messaging`)

Dos canales lógicos: `notificationRequests` (entrada de encargos) y `notificationEvents` (salida
informativa).

- **Publica** `EmailSent` y `EmailDeliveryFailed` con `reliability: best-effort`. **No son fuente de
  verdad**: pueden perderse aunque el correo sí haya salido. Quien necesite certeza consulta
  `findMessageByIdempotencyKey` o `listMessages`.
- **Consume** `NotificationRequested` (`nature: request`, sobre `keel`): cualquier servidor
  registrado encarga un envío. El payload es **nuestra firma de entrada**, no el hecho de nadie. La
  identidad del emisor sale de `metadata.source`; un emisor sin `Application` registrada va a la
  cola de descartes, no se descarta en silencio.
- **Política de fallo**: los errores de negocio (plantilla inexistente, destinatario suprimido,
  variables que faltan…) van **directos a la cola de descartes sin reintentar** — reintentarlos no
  los cura; el resto se reintenta 3 veces con backoff exponencial y acaba en la DLQ.

### Estado (`persistence`)

Modelo relacional, 5 entidades persistidas. Claves naturales por
`[application, code]`, `[template, versionNumber]`, `[application, idempotencyKey]` y
`[application, address]`. Índices derivados de las queries reales, incluido uno **sobre la lista**
`recipients` que sostiene el filtro por destinatario. Ese índice nombra **solo** la lista, a
propósito: el acotado por aplicación y el orden por fecha los aporta `[application, status,
requestedAt]` sobre el propio mensaje, y un índice compuesto que mezclara la lista con esos campos
no sería materializable —los elementos de una lista no viven en la misma tabla que ellos—.

- `transactionalBoundary: per-aggregate`, `optimisticLocking: all`.
- `audit.timestamps: all` y `audit.authorship: all`, con el actor **real** de cada vía: el usuario,
  el cliente de máquina, o el identificador de correlación cuando la escritura la dispara el reloj
  o el canal de eventos. Nunca un actor inventado.

### Acceso (`security`)

OIDC para personas, `client-credentials` con `validateAudience` para máquinas. **Cuatro roles** y
diez permisos, con mínimo privilegio por cliente: `dispatch-only` solo puede pedir envíos y no leer
mensajes, `mail-provider` solo puede reportar rebotes.

| Rol | Alcance por aplicación | Permisos |
|---|---|---|
| `notifications-admin` | **exento** (transversal) | `application:write/read`, `template:write/read`, `message:read`, `suppression:write/read`, `retention:purge` |
| `template-editor` | acotado | `application:read`, `template:write`, `template:read` |
| `application-operator` | **acotado** (es su propósito) | `application:read`, `template:read`, `message:read`, `suppression:read` |
| `notifications-auditor` | **exento** (transversal) | `application:read`, `template:read`, `message:read`, `suppression:read` |

**Alcance por aplicación**: el token humano lleva el claim `applications`. Las 13 operaciones que
actúan sobre una aplicación concreta devuelven `APPLICATION_FORBIDDEN` fuera de alcance; las de
colección (`listApplications`, `listMessages`) **filtran** en vez de fallar. `notifications-admin` y
`notifications-auditor` están exentos por diseño; `application-operator` es el rol que **sí** lo
ejerce, y por eso existe: es la única identidad que combina `message:read` con acotación por
aplicación, y sin ella la acotación sobre los datos personales de los mensajes no sería observable
desde fuera. `cors` declarado para la SPA de back-office.

## 6. Decisiones de diseño (qué / por qué)

### El fallo de envío es terminal

**Qué**: un mensaje `failed` no se recupera nunca; `failed` no tiene transición de salida y el
servicio no reintenta. **Por qué**: un correo que sale no lo deshace nada, y un reintento automático
sobre un fallo cuya causa no se conoce es la vía más rápida a mandarle dos correos a una persona
real. Quien quiera recuperar el aviso vuelve a pedirlo con otra clave, que es una decisión
consciente de quien conoce el negocio. **Descartado**: reintento con backoff dentro del servicio.

### El rescate de un mensaje atascado marca `failed` sin reenviar

**Qué**: un mensaje que lleva más de 15 minutos en `sending` pasa a `failed`. **Por qué**: un proceso
caído lo dejaría ahí para siempre y ninguna otra vía lo saca de ese estado. El rescate **no reenvía**
a sabiendas: si la caída ocurrió después de que el relay aceptara el mensaje, se marca como fallido
un correo que sí salió. Se prefiere ese **falso negativo visible** a un segundo correo real.

### El rescate se mide sobre `sendingSince`, no sobre `requestedAt`

**Qué**: `EmailMessage` lleva una marca propia del intento, estampada en la misma transición que
`queued → sending`. **Por qué**: la antigüedad que importa es **cuánto lleva atascado el intento**,
no cuánto lleva pedido el correo. Medir sobre `requestedAt` confunde las dos cosas: un mensaje que
esperó dos horas en cola y acaba de pasar a `sending` sería rescatado de inmediato, y el rescate
marcaría `failed` correos que están saliendo bien en ese momento. **Alternativa descartada**:
reutilizar `requestedAt` para no añadir campo — más barato en el modelo, pero deja el rescate sin
criterio observable y convierte una salvaguarda en una fuente de falsos negativos.

### La guarda de idempotencia es permanente

**Qué**: `keySource: payload-field` sobre `idempotencyKey`, **sin `ttlSeconds`**. **Por qué**: una
ventana que caduca dejaría salir un segundo correo pasado el plazo, y eso no se deshace. Es la razón
exacta por la que la purga de datos personales conserva la fila y su clave. **Descartado**: un TTL
de días o meses. **Descartado también**: `payload-hash`, que haría indistinguibles dos avisos
legítimamente idénticos —el mismo recordatorio enviado dos veces a propósito.

**Por dónde llega la clave no es aquí un detalle de transporte, sino lo que decide si el mecanismo
sirve.** La operación entra por **dos puertas** —HTTP y el canal de eventos— y el broker no manda
cabeceras, así que `client-key` —que describe una cabecera `Idempotency-Key`— dejaría sin deduplicar
la mitad de las peticiones. La clave ya es un campo del contrato, y eso es exactamente lo que
`payload-field` declara: cubre las dos puertas por igual. **Descartado**: `client-key`, que fue la
declaración inicial del diseño y se corrigió al ver que solo cubría el eje HTTP.

Consecuencia que el bloque no declara sino que provoca: `idempotencyKey` participa en la clave
natural `(application, idempotencyKey)` de `EmailMessage`, así que **esa constraint es la guarda** y
no hay registro de claves aparte que mantener.

«Mismo contenido» se definió de forma cerrada —`templateCode` + el **conjunto** de destinatarios
normalizados + el **conjunto** de variables declaradas, **sin importar el orden**— porque un cliente
que reintenta tras un timeout puede recomponer su petición desde una estructura sin orden estable, y
devolverle un `409` por eso convertiría un reintento legítimo en un correo que nunca sale.

### La aplicación solicitante nunca viaja en el cuerpo

**Qué**: se resuelve del `serviceClient` de la credencial en HTTP y de `metadata.source` en el canal.
**Por qué**: si viajara en el cuerpo, cualquier cliente autenticado podría enviar en nombre de otro
**desde su remitente verificado**. Cada entrada de `serviceClients` lleva por nombre el `code` de una
`Application`, y esa correspondencia es lo que cierra la puerta.

En el canal de eventos la garantía es **más débil y se acepta a sabiendas**: el broker autentica a
los emisores al publicar, pero cada uno estampa su propio nombre en `metadata.source`, lo que es
política de desarrollo y no un control técnico. Se sostiene mientras todos los emisores sean
sistemas propios. **El día que se integre un tercero, o que haga falta no-repudio del origen, se
pasa a `location: header` y lo garantiza el broker.**

### Fiabilidad de publicación: `best-effort`

**Qué**: `EmailSent` y `EmailDeliveryFailed` pueden perderse. **Por qué**: son informativos y la
fuente de verdad —`findMessageByIdempotencyKey` y `listMessages`— está siempre disponible. Un outbox
metería tabla y despachador para un aviso que ya tiene camino fiable de consulta.
**Descartado**: `outbox`. Se justificaría el día que algún consumidor actúe sobre estos eventos sin
poder sondear. Hoy **no tienen consumidor conocido** y se publican igualmente, con el payload
declarado como contrato estable, para que uno futuro pueda reaccionar sin sondeo.

### Frontera transaccional `per-aggregate` y su consistencia aceptada

**Qué**: casi toda operación escribe dentro de un único agregado. La única escritura que cruza es la
supresión automática de `sendQueuedMessage`, que marca el mensaje `failed` (agregado `EmailMessage`)
y añade la dirección a la lista (agregado `SuppressedAddress`) en **dos transacciones distintas**.
**Consecuencia aceptada**: si el proceso cae entre las dos, el mensaje queda `failed` y la dirección
sin suprimir — el rebote siguiente la suprime. **Por qué**: se prefiere eso a ensanchar la frontera
para un efecto que se corrige solo.

### Concurrencia: `optimisticLocking: all`

**Qué**: dos escrituras concurrentes sobre la misma raíz producen un `409`
(`CONCURRENT_MODIFICATION`, el código canónico del framework, publicado en el contrato aunque
ninguna operación lo declare). **Por qué**: publicar una versión **calcula el número a partir del
mayor existente** y archiva la activa. Sin conflicto explícito, dos publicaciones simultáneas
romperían la invariante de una sola versión activa. **Descartado**: último-gana silencioso.

### `APPLICATION_ADDRESS_ALREADY_EXISTS`: la carrera de la lista de supresión tiene código propio

**Qué**: `suppressAddress` y `reportEmailBounce` declaran un `409`
`APPLICATION_ADDRESS_ALREADY_EXISTS` para el choque contra la clave natural
`(application, address)` de `SuppressedAddress`. **Por qué**: la repetición **secuencial** no es un
error —suprimir una dirección ya `active` responde `200` sin duplicar—, pero dos peticiones que
miran a la vez pasan las dos la comprobación previa y la segunda choca en la base. Sin declararlo,
el generador sintetiza un código de convención suyo y el conflicto sale por el cable con un nombre
que el contrato no promete; el nombre lo fija la clave, así que acaba en los campos que la componen.
**Descartado**: dejarlo caer en el código genérico de unicidad, y también reutilizar
`CONCURRENT_MODIFICATION`, que es el conflicto de la raíz `Application` y no el de esta clave.
Lo cubre `FL-SUP-040`, por sus dos puertas.

`sendQueuedMessage` escribe en esa misma lista y **no** declara este código, a propósito: es interna
y no tiene cliente a quien dar un `409`, así que su colisión se absorbe sin cambiar el desenlace del
mensaje. El código es de las dos puertas HTTP, no de la clave.

### `APPLICATION_FIELD_REQUIRED`: vaciar no es lo mismo que omitir

**Qué**: la edición de una aplicación distingue tres cosas y no dos —omitir un campo (conserva),
enviarlo con valor (sustituye) y enviarlo presente y en blanco (borra)—, y el borrado solo vale
sobre los campos que la aplicación puede no tener. Sobre `name` y `senderAddress` se rechaza con
código propio, `400`. **Por qué**: `VALIDATION_ERROR` no puede expresarlo. Para el validador de
input una cadena vacía en un `PATCH` es un valor perfectamente válido; el rechazo no nace del
formato sino de la semántica del `PATCH` y de que el campo sea requerido en el dominio.
**Descartado**: colgarlo de `VALIDATION_ERROR`, que dejaría al cliente sin saber si el problema es
el formato del dato o su intento de dejar la aplicación sin remitente.

Lo que el código protege es concreto: una aplicación sin `senderAddress` es una aplicación que **no
puede enviar**, y el fallo no aparecería al editarla sino más tarde, en el despacho, sobre correos
que ya nadie relaciona con aquel `PATCH`.

### Sin idempotencia en el back-office

**Qué**: `registerApplication`, `createTemplate`, `publishTemplateVersion` y `suppressAddress` no
declaran `idempotency`. **Por qué**: son operaciones de un panel humano, no de un cliente que
reintenta solo; las de alta ya fallan con `409` por clave natural, y una versión duplicada la
arregla el propio editor publicando o reactivando la que quiera. **Descartado**: obligar a la SPA a
generar y gestionar una clave en cada formulario. Queda dicho en voz alta, no asumido.

### Sin caché en ninguna query

**Qué**: ninguna operación declara `cache`, ni siquiera la resolución de la versión activa que corre
en **cada** aceptación de envío. **Por qué**: son búsquedas por índice de igualdad sobre tablas
pequeñas, y ahí una caché añade riesgo de dato rancio sin ganar nada — servir una versión retirada,
o dejar pasar un correo a una dirección recién suprimida, que es justo lo que la lista existe para
impedir. **Descartado**: cachear la versión activa de plantilla.

### Superficie M2M con operaciones propias

**Qué**: tres operaciones M2M propias, **cero `audience: both`**. **Por qué**: compartir endpoint es
compartir output, errores, paginación y scopes entre dos contratos que crecen en direcciones
opuestas. `findMessageByIdempotencyKey` existe separada de `getMessage` precisamente porque un
cliente máquina direcciona por **su** clave, no por un uuid que no conoce.

### Sin endpoint de lote

**Qué**: `POST /notifications` acepta un correo por llamada (hasta 20 destinatarios del mismo
mensaje). **Por qué**: quien tenga volumen publica N mensajes en `notificationRequests`, que es
asíncrono, absorbe picos y ya tiene reintentos y DLQ. El endpoint HTTP queda para el envío
transaccional puntual que necesita respuesta inmediata. **Descartado**:
`requestNotificationsBatch`.

### El remitente falla cerrado

**Qué**: `sender.source: data` **sin `fallback`**. Un mensaje cuyo remitente no resuelva pasa a
`failed` y se descubre por `listMessages`. **Por qué**: antes que enviar desde una dirección que
nadie verificó, no se envía. Quemar la reputación del dominio no se deshace, y es un recurso
compartido por todos los consumidores.

### Las dos mitades del remitente se congelan juntas

**Qué**: `EmailMessage` copia de la aplicación **tanto** `senderAddress` **como** `senderName` al
aceptar la petición, y el correo se compone con lo copiado. **Por qué**: son una sola cabecera
`From`. La alternativa descartada era guardar solo la dirección y leer el nombre visible de
`Application` en el momento de enviar —más barato, y una mitad menos que mantener—, pero un cambio
de nombre entre la aceptación y el envío, o una consulta posterior del histórico, produciría un
`From` que nunca salió: dirección de entonces con nombre de ahora. El registro de un correo existe
para decir qué leyó el destinatario, y una mitad fresca lo convierte en una reconstrucción. Quien
derive este diseño y no necesite trazabilidad del remitente puede volver a la alternativa: es
retirar un campo `computed`, no tocar ningún flujo.

### La supresión es automática y por aplicación

**Qué**: un rechazo duro del relay, o un rebote permanente notificado después, añade la dirección a
la lista **sin que nadie lo pida**. **Por qué**: la reputación del remitente es compartida, así que
insistir sobre una dirección muerta no puede quedar en manos de cada llamante. Es por aplicación y
no global porque una dirección puede ser válida para un consumidor y no para otro.

**Un rebote permanente reactiva un registro `released`**, yendo deliberadamente contra la decisión
humana que lo liberó: el proveedor acaba de confirmar que la dirección sigue muerta. Quien la liberó
puede volver a liberarla. En cambio **liberar algo ya liberado sí es un error** (`404`), a
diferencia de suprimir dos veces, que es un no-op válido: no es la operación inversa.

**La supresión automática de `sendQueuedMessage` sigue exactamente las mismas reglas** que
`reportEmailBounce` —no duplica registro, no hace nada si la dirección ya está `active`, y reactiva
una `released`—, y por el mismo motivo: la reputación es compartida, así que la vía por la que llega
la noticia del rebote no debería cambiar lo que se hace con ella. **Descartado**: que el rechazo
síncrono NO reactivara un registro liberado, tratándolo como señal más débil que la notificación del
proveedor. Se rechazó porque haría que el mismo hecho —esta dirección rebota duro— produjera dos
resultados distintos según la puerta por la que entrase.

**Y nunca hace fallar el envío**: el desenlace del mensaje es `failed` con supresión o sin ella, y
una colisión al escribir la lista no se propaga como error. Es la diferencia con `reportEmailBounce`,
que sí declara `APPLICATION_ADDRESS_ALREADY_EXISTS` (abajo): aquella entra por HTTP y tiene un
cliente a quien devolverle un `409`; `sendQueuedMessage` es interna y no lo tiene.

**Su precondición es que el rechazo identifique la dirección concreta.** Si el relay rechaza sin
desglosar por destinatario, el mensaje queda `failed` y **no se suprime nada** — que es el
comportamiento correcto, no una carencia: suprimir exige saber cuál rebotó, y adivinarla castigaría
a destinatarios sanos del mismo mensaje. Por eso el camino determinista del rebote duro es
`reportEmailBounce`, y esta regla solo adelanta el trabajo cuando el dato ya viene en el rechazo.

### `reportEmailBounce`: la mitad del rebote que casi siempre falta

**Qué**: una operación M2M explícita para el rebote que llega **después** de que el relay aceptara el
mensaje. **Por qué**: el rechazo síncrono que `sendQueuedMessage` detecta es solo la mitad; el relay
acepta y el rebote vuelve horas después. Sin un canal declarado para esa noticia, la regla de
suprimir por rebote duro **no sería implementable ni verificable** — ninguna infraestructura de
prueba rebota sola. El `serviceClient` `mail-provider` es quien la usa.

### La purga conserva la fila

**Qué**: a los 18 meses se vacían `recipients`, `renderedSubject` y `variableValues`, y se sella
`personalDataPurgedAt` — pero la fila, su clave de idempotencia, su estado y su desenlace se
conservan **para siempre**. **Por qué**: borrar la fila rompería la guarda permanente contra el
doble envío, que es lo único que impide que una clave antigua vuelva a mandar un correo. La
consecuencia aceptada es que la tabla crece sin cota.

### Ninguna creación devuelve `201`

**Qué**: todas devuelven `200`. **Por qué**: el generador emite `Location` en toda operación `201`
cuyo output declare `id`, componiéndola con la URI de la petición más ese id — pero este servicio
direcciona sus recursos por **clave natural** (`code`, `templateCode`, `versionNumber`, `address`),
así que esa cabecera apuntaría a una URL que no existe. Con `200` no se emite, y el cuerpo ya
devuelve la clave natural con la que el cliente direcciona lo que acaba de crear.

### El alcance por aplicación vive en las reglas, no en `accessRule`

**Qué**: la guarda del claim `applications` se declara como `rule` en cada operación afectada.
**Por qué**: `roles`, `permissions` y `scopes` son **globales** — quien pasa la regla de acceso pasa
para todos los recursos — y el DSL no tiene sitio en `accessRule` para declarar una acotación por
dato. Sin esa guarda, un editor de plantillas de una aplicación podría leer los destinatarios y el
asunto (datos personales) de los correos de cualquier otra. `keel validate` avisa del `403` porque
no puede ver esas reglas; el aviso está aceptado a sabiendas y su respuesta vive en el bloque
`access` de la capa `security`.

### `application-operator`: el rol que hace observable el alcance por aplicación

**Qué**: un cuarto rol humano que combina `message:read` con acotación por el claim `applications`.
**Por qué**: los otros tres no podían ejercerla sobre datos personales. `template-editor` **no lee
correos** —su función es redactar contenido, y los destinatarios y el asunto renderizado no le hacen
falta— y los dos roles que sí los leen (`notifications-admin`, `notifications-auditor`) están
**exentos del alcance por diseño**. Sin este rol, la acotación por aplicación sobre los datos
personales de los mensajes no es observable desde fuera y su escenario no es ejercitable: la guarda
existiría en el spec sin que nada la ejercite. **Alternativa descartada**: dar `message:read` a
`template-editor` — más simple en la tabla de permisos, pero ensancha ese rol hasta datos personales
que su función no necesita, justo lo contrario del mínimo privilegio que el resto del diseño aplica.

### `dispatch-only` como cliente separado

**Qué**: un consumidor con `notification:send` y **sin** `message:read`. **Por qué**: pedir un envío
y leer los mensajes de una aplicación son capacidades distintas, y la segunda expone datos
personales de los destinatarios. Es mínimo privilegio aplicado de verdad, y no una hipótesis: hay un
escenario que lo verifica.

## 7. Ficha de reutilización: adoptar, derivar o evolucionar

### 7.1 Contrato estable vs adaptable

**Contrato estable** — cambiarlo rompe a quien ya lo consume, y exige versión **major**:

- Los **20 códigos de error propios** en `SCREAMING_SNAKE_CASE` con su status — 19 en la superficie
  HTTP, más `MESSAGE_NOT_QUEUED`, que solo observa la operación interna `sendQueuedMessage`. Se
  suman los **canónicos del framework** que el spec re-declara explícitamente
  (`IDEMPOTENCY_KEY_REUSED`, `IDEMPOTENCY_KEY_IN_PROGRESS`) y los que aporta el framework sin
  declararlos (`VALIDATION_ERROR`, `INVALID_STATE_TRANSITION`, `CONCURRENT_MODIFICATION`).
- Los **tres endpoints M2M** y la forma de sus payloads.
- Los **nombres y payloads** de `EmailSent` y `EmailDeliveryFailed`, declarados contrato estable
  aunque hoy no tengan consumidor.
- El payload de entrada de `NotificationRequested`: es nuestra firma, y hay emisores acoplados.
- Los **cuatro roles**, los diez permisos y los scopes, y el nombre de cada `serviceClient` (que es
  el `code` de una aplicación).
- `basePath: /api/v1` y los status de éxito, `200` en las creaciones incluido.

**Adaptable sin romper a nadie** (patch/minor): las reglas de `use-cases` que no cambian el
contrato, las cotas de los cuerpos de plantilla, el tamaño de tanda del despacho (200) y de la purga
(5000), la ventana de rescate (15 minutos), el umbral de retención (18 meses), los índices de
`persistence`, y la política de la suscripción.

### 7.2 Puntos de extensión típicos

- **Enums ampliables**: `SuppressionReason` admite motivos nuevos (`unsubscribe`, `spam-trap`) sin
  tocar nada más. `MessageStatus` **no**: añadir un estado toca el lifecycle, el ciclo de despacho y
  los escenarios.
- **`lifecycle` de `EmailMessage`**: es donde insertar un estado `retrying` si algún día se decide
  reintentar. Hoy `failed` es terminal a propósito, así que la arista de vuelta no existe y habría
  que declararla.
- **Capas ausentes**: no hay `storage` (sin adjuntos) ni `dependencies`/`http-clients` (sin
  proveedores). Un derivado que necesite adjuntos añade `storage` con su bucket, un campo `file` y
  los errores de subida; uno que necesite un proveedor de correo por API en vez de SMTP cambia la
  capa `mail` o añade `http-clients`.
- **`sendQueuedMessage` es `internal: true`**: es la pieza sustituible por excelencia. Cambiar cómo
  se entrega el correo no toca ninguna otra operación.
- **Piezas reutilizables en otro servicio**: el patrón plantilla-contenedor + versión inmutable con
  `hasActiveVersion` computed; el patrón cola + ciclo con marca `sending` y rescate por antigüedad;
  el patrón supresión con `released` que se reactiva en vez de duplicarse; y la purga que conserva
  la fila para no romper una guarda permanente.

### Supuestos y limitaciones

**Supone**:

- **Correo transaccional de volumen moderado**: decenas de miles de correos al día como mucho,
  disparados por un hecho de negocio. La cota de 200 mensajes por minuto del ciclo de despacho pone
  el techo teórico en unos 288.000 diarios y deja margen amplio para picos. Quien necesite volumen
  de campaña debe revisar esa cota y probablemente la estrategia de despacho entera.
- **Multi-aplicación con despliegue único**: una instancia sirve a todas las aplicaciones de la
  organización. `Application` es la unidad de aislamiento **lógico** (plantillas, remitente y lista
  de supresión propios), pero no hay separación física de datos: todo vive en el mismo almacén y el
  aislamiento lo garantizan el claim del token y la credencial de máquina. **La reputación del
  dominio remitente es el recurso genuinamente compartido**, y es lo que justifica que la supresión
  sea automática y no opcional.
- **Todos los emisores del canal de eventos son sistemas propios**, dentro del mismo perímetro. Es
  lo que hace aceptable que cada uno estampe su `metadata.source`.
- Que existe un proveedor de correo capaz de **notificar rebotes** contra `reportEmailBounce`: sin
  esa notificación, la supresión solo se alimenta del rechazo síncrono y de la decisión manual.
- Y, para la supresión automática del rechazo **síncrono**, que el relay **desglose por
  destinatario** cuál rebotó. No todos lo hacen —los relays de prueba, ninguno—, y con un rechazo
  sin desglose esa vía queda inerte: el mensaje pasa a `failed` y nada se suprime. No es un fallo
  del servicio, pero sí un supuesto que quien despliegue debe comprobar en su relay, porque de él
  depende que la lista se alimente sola o solo por `reportEmailBounce`. Es también la razón de que
  ese camino se verifique con **prueba unitaria** (`UT-SND-010`) y no con un flujo de caja negra.

**No cubre, a propósito**:

- **Reintento de envíos fallidos.** El fallo es terminal; se recupera pidiéndolo de nuevo con otra
  clave.
- **Seguimiento de aperturas y clics.** Se registra si el relay aceptó y si rebotó, nada más: no hay
  píxel ni reescritura de enlaces, y el modelo no tiene dónde guardarlo.
- **Programación de envíos y campañas.** No se puede pedir un correo «para el martes» ni a una lista
  de suscriptores: toda petición se encola para el ciclo siguiente y el destinatario lo aporta
  siempre quien pide el envío.
- **Adjuntos y otros canales.** `attachments: false`, y por eso no hay capa `storage`. Tampoco hay
  SMS, push ni notificación in-app, pese al nombre del servicio: **solo correo**.
- **Separación física de datos entre aplicaciones**, y por tanto un escenario SaaS multi-cliente
  entre organizaciones que no se confían entre sí.

### 7.4 Cómo reutilizarlo

Empieza por el resumen mecánico:

```bash
keel describe notifications
```

**Si te sirve tal cual** — el caso probable si tu encargo es «correo transaccional multiaplicación
con plantillas que edita negocio»: **adóptalo**. Copia `specs/notifications/` y `docs/notifications/`
a tu workspace (o `keel registry get notifications` si está publicado) y ve **directo a generar**,
sin fase de diseño. Llega con sus derivados al día.

**Si tienes que cambiarlo** — otro modelo de tenencia, reintentos, adjuntos, otro canal:
**derívalo**.

```bash
keel new <nuevo-servicio> --from notifications
/keel-design specs/<nuevo-servicio>
```

Clona solo el spec con linaje `basedOn`, y `/keel-design` arranca en modo derivación: entrevista
solo sobre lo que cambia. Los derivados de `docs/` no se heredan porque describen a este servicio;
salen del cierre normal del diseño derivado.

**Cuál esperar aquí.** Lo listado en «Contrato estable» es amplio y muy específico de este dominio
—los códigos de error, la superficie M2M, el payload de entrada del canal—, así que un encargo que
encaje en los supuestos de arriba se resuelve **adoptando**, no derivando. Derivar para acabar
usándolo sin cambios obliga a regenerar a mano todo lo que ya estaba hecho.

**Si el cambio es sobre este mismo servicio** y el diseño ya está cerrado, la puerta es
`/keel-evolve specs/notifications`, que versiona el contrato y regenera en cascada los derivados que
el cambio deje atrás.
