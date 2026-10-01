# notifications — Documento de diseño

> specs/notifications v0.1.2. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`notifications` envía **correo transaccional en nombre de varias aplicaciones consumidoras** (billing,
shipping, …). Cada aplicación pide un correo a partir de una **plantilla que mantiene negocio sin
desplegar**. Las peticiones entran por dos puertas con el mismo contrato: HTTP servidor-a-servidor y un
canal de eventos. Tras aceptar una petición el servicio la encola y la envía en el siguiente ciclo de
despacho. Todo fallo de entrega es terminal y se publica; el servicio no reintenta.

El mayor riesgo del servicio es **mandar dos veces el mismo correo a una persona real**, y cada decisión
de este documento se tomó para evitarlo: una clave idempotente permanente, una cola con reclamo único y
un rescate que nunca reenvía.

Fuera de alcance: adjuntos, envío programado a una fecha, otros canales (solo correo) y la
verificación del remitente ante el proveedor, que se hace fuera del servicio.

## 2. Modelo de dominio

### Value types

| Tipo | Qué es |
|---|---|
| `ApplicationCode` | Código estable de una aplicación (`^[a-z][a-z0-9-]{2,47}$`). Coincide con su cliente máquina y con el `source` de sus eventos. |
| `TemplateCode` | Código estable de una plantilla, único dentro de su aplicación (mismo patrón). |
| `EmailAddress` | Dirección de correo (≤ 254). Siempre se normaliza a minúsculas. |
| `IdempotencyKey` | Clave con la que la aplicación identifica un envío: `^[A-Za-z0-9._:-]{8,128}$`, segura en una URL porque viaja en la ruta de consulta M2M. |
| `VariableName` | Nombre de variable de plantilla (`^[A-Za-z][A-Za-z0-9_]{0,63}$`). |
| `TemplateVariable` | Variable declarada por una versión: `name`, `description`, `required`. |
| `VariableValue` | Valor de una variable en una petición: `name`, `value` (texto ≤ 2000, se escapa y nunca se evalúa). |
| `SenderSnapshot` | From (dirección y nombre) y Reply-To **congelados** al aceptar. |
| `FailureReason` | Motivo estable de un fallo: `delivery-rejected`, `delivery-error`, `render-error`, `application-suspended`, `recipient-suppressed` y `sending-timeout`. |

### Agregados

| Agregado | Raíz | Internas | Por qué |
|---|---|---|---|
| `Application` | `Application` | `EmailTemplate`, `EmailTemplateVersion` | La aplicación es dueña de sus plantillas. Publicar, activar y retirar cambian a la vez dos versiones y el `hasActiveVersion` de la plantilla. |
| `EmailMessage` | `EmailMessage` | — | Cada correo es su propia frontera. Congela lo que necesita de la plantilla, así que editarla después no lo afecta. |
| `SuppressedAddress` | `SuppressedAddress` | — | La supresión es por aplicación y vive aparte: la escriben el back-office, los rebotes y el despacho. |

### Entidades y ciclos de vida

- **Application** — `code` (inmutable), `name`, `senderAddress` (requerido y verificado fuera),
  `senderName`, `replyToAddress` y `status`. Ciclo de vida `active ⇄ suspended`.
- **EmailTemplate** — `code` (único en la aplicación e inmutable), `name`, `description` y
  `hasActiveVersion`, un campo **computed** y persistido que se recalcula en la misma transacción que
  publicar, activar o retirar.
- **EmailTemplateVersion** — **inmutable**. `versionNumber` es correlativo desde 1 y nunca se reutiliza.
  Lleva `subject` (≤ 300), `htmlBody` (≤ 256 KiB), `textBody` (≤ 64 KiB, obligatorio), `variables`
  (≤ 50) y `publishedAt`. Ciclo de vida `active ⇄ archived`, con **como mucho una** versión `active`
  por plantilla (índice único condicionado).
- **EmailMessage** — `idempotencyKey`, `recipients` (≤ 20, deduplicados y en minúsculas),
  `renderedSubject` (≤ 998), `variableValues` (**sensitive**), `contentFingerprint` (**sensitive,
  computed**: SHA-256 del contenido canónico) y el snapshot congelado (`sender`, `templateCode`,
  `templateVersionNumber` y `templateVersionId`). A eso se suman `failureReason`, `failureDetail` y los
  instantes `requestedAt`, `sendingSince`, `sentAt`, `failedAt` y `personalDataPurgedAt`.
  Ciclo de vida `queued → sending → sent | failed`; `sent` y `failed` son terminales.
- **SuppressedAddress** — `address`, `reason` (`hard-bounce | complaint | manual`), `notes`,
  `suppressedAt` y `releasedAt`. Ciclo de vida `active ⇄ released`. El par (aplicación, dirección) es
  único para siempre: liberar conserva el registro y suprimir de nuevo lo reactiva.

Llevan auditoría **en el contrato** (`createdAt`, `updatedAt`, `createdBy` y `updatedBy`)
Application, EmailTemplate, EmailTemplateVersion y SuppressedAddress. EmailMessage no la lleva.

## 3. Invariantes y reglas clave

- **La misma petición nunca produce dos correos.**
  - La clave `(aplicación, idempotencyKey)` es única para siempre, también después de la purga.
  - «Mismo contenido» es la misma `contentFingerprint`: `templateCode`, el conjunto de destinatarios
    normalizados y el conjunto de pares de variables **declaradas** por la versión congelada, sin
    importar el orden.
  - Una repetición con el mismo contenido devuelve el mismo mensaje, aunque desde entonces la aplicación
    se haya suspendido o la plantilla retirado. Con contenido distinto → `IDEMPOTENCY_KEY_REUSED`.
- **Orden de aceptación**: aplicación → variables repetidas → repetición de la clave → suspensión →
  plantilla → versión activa → variables required → supresiones → longitud del asunto renderizado.
- **La guarda del envío es la transición `queued → sending`**, confirmada en su propia transacción
  **antes** de contactar con el relay. Lo que se queda en `sending` más de `sendingTimeoutMinutes` (15)
  lo rescata el despacho a `failed/sending-timeout` y **nunca lo reenvía**. Ese plazo es contrato, no
  mecánica del generador: la transición del rescate lo declara como su `stalledAfter`.
- En `sendQueuedMessage` se comprueban de nuevo la suspensión y las supresiones: lo aceptado no sale si
  entre tanto la aplicación se suspendió o un destinatario quedó suprimido.
- **Correo**:
  - Un único envío `multipart/alternative` con todos los destinatarios en To.
  - From, nombre y Reply-To salen del snapshot.
  - Lleva la cabecera `X-Notification-Id` con el id del mensaje, que es la correlación de los rebotes.
- **Plantillas**:
  - Los marcadores son `{{name}}` o `{{ name }}`, sin lógica de ningún tipo.
  - Toda variable usada tiene que estar declarada.
  - Una variable opcional ausente se sustituye por la cadena vacía.
  - En HTML se escapan `& < > " '`.
  - En el asunto, los saltos de línea de un valor se sustituyen por un espacio, para que no pueda
    inyectar cabeceras.
- **Supresión**:
  - Se rechaza la petición entera si algún destinatario está suprimido en la aplicación.
  - Un rebote `permanent`, o el rechazo definitivo del relay, suprime la dirección con `hard-bounce`,
    aunque un humano la hubiera liberado.
- **Datos personales**: a los `personalDataRetentionMonths` (18) meses de calendario se vacían
  `recipients`, `renderedSubject` y `variableValues`. Se conservan la fila, la clave y la huella.

## 4. Qué hace

### Back-office (personas, OIDC, acotado por el claim `applications`)

| Área | Operaciones |
|---|---|
| Aplicaciones | `registerApplication`, `updateApplication` (sustitución completa), `suspendApplication`, `reactivateApplication`, `getApplication` y `listApplications` (filtro por estado, orden por `code`). |
| Plantillas | `createTemplate` (vacía), `updateTemplate` (nombre y descripción), `publishTemplateVersion` (crea, activa y archiva la anterior; `Idempotency-Key` opcional de 24 h), `activateTemplateVersion` (revertir, o volver a poner en servicio), `retireTemplate`, `getTemplate`, `listTemplates`, `getTemplateVersion` y `listTemplateVersions` (de la más nueva a la más antigua, sin cuerpos). |
| Supresiones | `suppressAddress` (no-op si ya está active; reactiva si está released), `releaseAddress` y `listSuppressedAddresses` (solo las active, las más recientes primero). |
| Correos | `getMessage` (fuera de alcance responde 404, igual que si no existiera) y `listMessages` (filtros por aplicación, estado, plantilla, destinatario y ventana temporal; orden `requestedAt` desc con desempate por id). |
| Datos personales | `purgeMessagePersonalData`, disparo manual (solo admin) además del diario. |

Los recursos se direccionan por su clave natural, así que **ninguna creación devuelve 201**. Todas las
listas paginan por offset (20 por defecto, 100 como máximo).

### Superficie servidor-a-servidor

| Operación | Endpoint | Quién |
|---|---|---|
| `requestNotification` | `POST /api/v1/notifications` → `202` (acepta, no promete la entrega) | las aplicaciones (`notification:send`) |
| `findMessageByIdempotencyKey` | `GET /api/v1/notifications/{applicationCode}/{idempotencyKey}` | las aplicaciones (`notification:read`), solo sobre la propia |
| `reportEmailBounce` | `POST /api/v1/messages/{messageId}/bounces` → `204` | el proveedor de correo (`bounce:report`) |

**La aplicación nunca viaja en el cuerpo**: por HTTP sale del cliente máquina (1:1 con su `code`) y
por eventos, de `metadata.source`.

### Procesos

- `dispatchQueuedMessages`, cada minuto y sin solape:
  - Primero rescata lo atascado en `sending` más de `sendingTimeoutMinutes` (`stalledAfter` de la
    transición `sending → failed`).
  - Después toma hasta 200 mensajes en `queued` por `requestedAt` y llama a `sendQueuedMessage`,
    interna y única operación que produce correo.
- `purgeMessagePersonalData`, diario a las 03:00 UTC y sin solape. Procesa lotes de 5000 hasta vaciar.
- `suppressHardBouncedAddress`, interna. La usan los rebotes y el rechazo síncrono del relay.

## 5. Fronteras e integraciones

- **Mensajería**:
  - Entrada por `notificationRequests` (suscripción `NotificationRequested`, `nature: request`,
    envoltura Keel). La identidad sale de `metadata.source`. Política de fallo: 3 reintentos
    exponenciales y después deadLetter; un emisor no registrado también va a deadLetter.
  - Salida por `notificationEvents`, en best-effort: `EmailSent` y `EmailDeliveryFailed`, **sin
    destinatarios**.
- **Correo**: SMTP con partes html+text y sin adjuntos. El remitente y el Reply-To salen de los datos,
  **sin fallback**. Las plantillas son dato del servicio (`templating: data`) y declaran sus variables.
- **Persistencia**:
  - Modelo relacional, con claves naturales `code`, `(application, code)`,
    `(template, versionNumber)`, `(application, idempotencyKey)` y `(application, address)`.
  - Índice multivalor sobre `recipients`.
  - Frontera transaccional por agregado y bloqueo optimista en todas las raíces.
- **Seguridad**:
  - OIDC para personas, con cuatro roles:
    - `notifications-admin`: todo.
    - `template-editor`: plantillas; no lee correos.
    - `application-operator`: lee los correos de sus aplicaciones y gestiona sus supresiones.
    - `notifications-auditor`: solo lectura.
  - Admin y auditor están exentos del claim `applications`.
  - Las máquinas se autentican con client_credentials con `aud=notifications`:
    - `billing` y `shipping`: pedir y consultar envíos.
    - `dispatch-only`: solo pedir.
    - `mail-provider`: solo reportar rebotes.
  - CORS habilitado para la SPA de back-office.

## 6. Decisiones de diseño

| Decisión | Elegido | Descartado | Por qué |
|---|---|---|---|
| Idempotencia de `requestNotification` (§3.2) | `payload-field idempotencyKey`, permanente | `client-key` (no llega por eventos); `payload-hash`; con TTL | Dos puertas con el mismo contrato: solo una clave en el cuerpo cubre las dos. Es permanente porque un reintento tardío no puede producir un segundo correo. La guarda es la clave natural, y la huella sobrevive a la purga. |
| Idempotencia de `publishTemplateVersion` | `client-key` 24 h por plantilla, cabecera opcional | sin idempotencia; cabecera obligatoria | Un doble clic crearía versiones duplicadas, y el número no se reutiliza. Obligarla no compensa: en el peor caso aparece una versión de más, sin ningún correo de más. |
| Rebotes y purga (§3.2) | idempotentes por naturaleza | `client-key` | Suprimir lo que ya está suprimido, o purgar lo ya purgado, no tiene efecto. |
| Caché (§3.3) | ninguna | TTL en plantillas | El estado de un mensaje y las supresiones tienen que verse al instante. |
| Superficie M2M (§3.4) | tres operaciones propias con `audience: services` | `audience: both` | Pedir, consultar por clave y reportar rebotes son contratos de máquina que evolucionan aparte del back-office. |
| Paginación (§3.8) | offset 20/100 con orden total | sin paginar | Todas las colecciones crecen sin cota. |
| Fiabilidad de publicación (§3.1) | best-effort | outbox | Los eventos no tienen consumidores hoy y la verdad del desenlace es el estado consultable por HTTP. Se revisa cuando alguien dependa de ellos. |
| Política de la suscripción (§3.5) | 3 reintentos exponenciales + deadLetter | solo retry; descartar; evento de rechazo | Nada se pierde en silencio y reintentar no duplica nada. El emisor que quiera saber si su petición se aceptó consulta por su clave. |
| Frontera transaccional (§3.7) | por agregado | por operación | Ninguna operación escribe en dos agregados. La supresión tras un rechazo del relay va aparte, con un reintento; el rebote posterior del proveedor sirve de respaldo. |
| Concurrencia (§3.9) | bloqueo optimista en todas las raíces, con `CONCURRENT_MODIFICATION` canónico | último gana | Es lo que hace que un solo envío gane la transición `queued → sending` y que nunca haya dos versiones activas. |
| Auditoría (§3.9b) | `declared` en las cuatro entidades de back-office | `all`; `none` | El back-office muestra quién publicó, suspendió o liberó. |
| Repetir un cambio de estado | back-office: `409 INVALID_STATE_TRANSITION`; supresión y rebotes: no-op real (arista `active → active`) | no-op en todo; 409 en todo | Para una persona, suspender dos veces es un error honesto, y es lo que el generador deriva del lifecycle. Suprimir dos veces es un no-op por encargo, y el proveedor reintenta los rebotes: ahí un 409 sería ruido. |
| Solape de los barridos | sin solape, y la transición como guarda | solape guardado solo por la transición | Evita conflictos y lotes pisados; la transición sigue siendo la red de seguridad. |
| Remitente | de los datos, congelado y sin fallback | dirección de respaldo | Antes que enviar desde una dirección que nadie verificó, no se envía. El Reply-To también se congela. |
| Rechazo parcial del relay | `sent` y se suprimen los rechazados | `failed`; solo log | El correo sí salió. Las direcciones que no existen no se vuelven a intentar. |
| Destinatarios | todos en To, un envío | una copia por destinatario; Bcc | La aplicación los agrupa a sabiendas, y un mensaje tiene un solo desenlace. |
| Correlación de rebotes | cabecera `X-Notification-Id` | dentro del `Message-ID` | Es explícita y no depende de un dominio configurado. |
| `getMessage` fuera de alcance | 404 | 403 `APPLICATION_FORBIDDEN` | Un 403 sobre un id confirmaría que el correo existe. |
| Huella de contenido | interna, SHA-256 sobre forma canónica | expuesta; algoritmo libre | Es un dato personal reidentificable, y migrar entre stacks no debe romper la idempotencia. |
| Identidad por eventos | `metadata.source` con la asunción escrita | cabecera del broker | La ACL del broker sobre el canal hace de scope. Se acepta a sabiendas que un emisor registrado podría suplantar a otro, con efecto irreversible; se revisa en cuanto publique un tercero. |
| Umbrales | parámetros `sendingTimeoutMinutes` y `personalDataRetentionMonths` | constantes | Son ajustables sin recompilar, y la purga queda verificable en pruebas (valor 0). |

Riesgos aceptados por escrito (`gaps.yaml`):
- Las quejas del proveedor se registran a mano.
- Traducir el webhook del proveedor al contrato de rebotes es un adaptador de despliegue.
- Una aplicación nueva exige aprovisionar su cliente máquina, que es una evolución del diseño.
- Un payload inválido por eventos gasta los reintentos antes de ir a deadLetter.
- La deadLetter la reinyecta operación.
- Las supresiones liberadas no se consultan por API.
- Los escenarios exigen que el arnés siembre el claim `applications` de los usuarios de prueba con
  `[billing]`: con un generador que publique otro valor, esos flujos no se pueden alcanzar.

## 7. Ficha de reutilización

### Contrato estable vs adaptable

- **Estable**:
  - Los códigos de error: `APPLICATION_*`, `TEMPLATE_*`, `IDEMPOTENCY_KEY_*`, `RECIPIENT_SUPPRESSED`,
    `MESSAGE_*`, `BOUNCE_RECIPIENT_UNKNOWN`, `SUPPRESSED_ADDRESS_NOT_FOUND`,
    `RENDERED_SUBJECT_TOO_LONG`, `INVALID_TIME_WINDOW` y los canónicos.
  - Las rutas M2M y el cuerpo de `requestNotification`, idéntico por las dos puertas.
  - Los eventos `EmailSent` y `EmailDeliveryFailed` con su payload.
  - `FailureReason`, la sintaxis de plantilla y el escapado.
  - La cabecera `X-Notification-Id`.
  - Roles, permisos y scopes.
- **Adaptable sin romper a nadie**: los umbrales (parámetros), el tamaño de los lotes del despacho y de
  la purga, los índices, las reglas internas del despacho y el catálogo de `serviceClients` (añadir un
  cliente).
- Versionado según `docs/methodology.md`:
  - **Patch**: aclarar prosa.
  - **Minor**: añadir operaciones, filtros, campos opcionales o motivos de fallo nuevos al final del enum.
  - **Major**: cambiar la semántica de la clave idempotente o el payload de los eventos.

### Puntos de extensión típicos

- Una operación M2M `reportEmailComplaint` para las quejas del proveedor (`reason: complaint`).
- Pasar la publicación a `outbox` cuando exista un consumidor de los eventos.
- Pasar la identidad por eventos a `location: header` cuando publique un tercero.
- Un filtro por dirección en `listSuppressedAddresses`, o la consulta de las liberadas.
- Reutilizables en otros servicios: el patrón de versión inmutable con una sola activa
  (índice condicionado + recálculo en la raíz), la clave `payload-field` permanente con huella que
  sobrevive a la purga, y el rescate que no reenvía.

### Supuestos y limitaciones

- Una sola organización con varias aplicaciones propias: todos los consumidores y emisores de eventos
  están dentro del mismo perímetro de confianza. No es un servicio multi-organización.
- Un solo relay SMTP para todo el servicio. Cada aplicación verifica su remitente ante ese proveedor,
  fuera del servicio.
- Plantillas monoidioma: la localización se hace con plantillas distintas. No hay locale en el modelo.
- Volumen de hasta decenas de miles de correos al día. El despacho (200 por minuto) tiene holgura; no es
  un servicio de envío masivo.
- Sin adjuntos ni envío programado a una fecha: todo sale en el siguiente ciclo del despacho
  (limitación deliberada).
- Sin reintentos de entrega: la aplicación decide si vuelve a pedir el correo con una clave nueva.
- El diseño no cubre el seguimiento de aperturas y clics ni otros canales (SMS, push).
  > supuesto pendiente: el diseñador no confirmó si estas dos ausencias son deliberadas.

### Cómo reutilizarlo

`keel describe notifications` da el resumen mecánico.

- **Si sirve tal cual**, adóptalo con `keel registry get notifications`, que lo trae con sus derivados
  al día, y ve directo a `keel-<tech> build specs/notifications`.
- **Si hay que cambiarlo**, derívalo con `keel new <nuevo> --from registry:notifications`, que clona
  solo el spec, y `/keel-design` entrevista únicamente sobre lo que cambia.

Lo esperado es **adoptarlo**. Lo que suele variar entre organizaciones (umbrales, clientes máquina,
lotes) es adaptable sin tocar el contrato. Derivar solo compensa si cambian la tenancy, el canal (no
correo) o la política de reintentos.
