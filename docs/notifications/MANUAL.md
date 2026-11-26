# Manual de uso por actor de notifications

> specs/notifications v0.3.1. Generado con `keel describe notifications --manual`: no se edita, se cambia el diseño y se vuelve a generar.

## Actores

| Actor | Quién es | Operaciones |
|---|---|---|
| Rol notifications-admin | Administra el servicio completo, incluido el remitente y la purga manual. | 17 |
| Rol template-editor | Gestiona plantillas y sus versiones. No lee correos enviados. | 8 |
| Rol notifications-operator | Soporte diario; consulta correos y gestiona supresiones. | 10 |
| Rol notifications-auditor | Solo lectura de todo el servicio. | 9 |
| Cliente máquina billing | Facturación; pide correos y reconcilia sus peticiones. | 2 |
| Cliente máquina shipping | Envíos; pide correos y reconcilia sus peticiones. | 2 |
| Cliente máquina dispatch-only | Sistema que solo pide correos, con el mínimo privilegio. | 1 |
| Cliente máquina mail-provider | Proveedor de correo; notifica rebotes. | 1 |

## Rol notifications-admin

Administra el servicio completo, incluido el remitente y la purga manual.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [configureSenderSettings](#configuresendersettings) | `PUT /api/v1/sender-settings` | Crea o reemplaza el remitente del servicio. Los mensajes ya aceptados conservan el remitente que congelaron. | — | — |
| [getSenderSettings](#getsendersettings) | `GET /api/v1/sender-settings` | Devuelve el remitente configurado. | — | — |
| [createTemplate](#createtemplate) | `POST /api/v1/templates` | Da de alta una plantilla vacía, sin versiones; no se puede usar hasta publicar la primera. | — | — |
| [publishTemplateVersion](#publishtemplateversion) | `POST /api/v1/templates/{templateCode}/versions` | Publica contenido nuevo para una plantilla. La versión nueva queda active y la que lo estaba pasa a archived, en la misma transacción. | EmailTemplateVersion: active → archived | — |
| [activateTemplateVersion](#activatetemplateversion) | `POST /api/v1/templates/{templateCode}/versions/{versionNumber}/activate` | Vuelve a poner en active una versión archivada (revertir) y archiva la que estuviera active. | EmailTemplateVersion: archived → active; EmailTemplateVersion: active → archived | — |
| [retireTemplate](#retiretemplate) | `POST /api/v1/templates/{templateCode}/retire` | Retira una plantilla archivando su versión active; conserva el historial y se puede volver a activar o publicar. | EmailTemplateVersion: active → archived | — |
| [getTemplate](#gettemplate) | `GET /api/v1/templates/{templateCode}` | Devuelve una plantilla sin su contenido. | — | — |
| [listTemplates](#listtemplates) | `GET /api/v1/templates` | Lista las plantillas, opcionalmente filtradas por si tienen versión active. | — | — |
| [getTemplateVersion](#gettemplateversion) | `GET /api/v1/templates/{templateCode}/versions/{versionNumber}` | Devuelve una versión concreta de una plantilla, con su contenido completo. | — | — |
| [listTemplateVersions](#listtemplateversions) | `GET /api/v1/templates/{templateCode}/versions` | Historial de versiones de una plantilla, de la más nueva a la más antigua, sin los cuerpos. | — | — |
| [getMessage](#getmessage) | `GET /api/v1/messages/{id}` | Devuelve un mensaje por su id, sin los valores de las variables. | — | — |
| [listMessages](#listmessages) | `GET /api/v1/messages` | Lista los mensajes con filtros, del más reciente al más antiguo. | — | — |
| [suppressAddress](#suppressaddress) | `PUT /api/v1/suppressions/{address}` | Suprime una dirección a mano. Reactiva el registro si estaba released; si ya está active, no cambia nada. | SuppressedAddress: released → active | — |
| [releaseAddress](#releaseaddress) | `POST /api/v1/suppressions/{address}/release` | Libera una dirección suprimida; el registro se conserva. Liberar una dirección ya released no cambia nada. | SuppressedAddress: active → released | — |
| [getSuppressedAddress](#getsuppressedaddress) | `GET /api/v1/suppressions/{address}` | Devuelve el registro de supresión de una dirección, esté active o released. | — | — |
| [listSuppressedAddresses](#listsuppressedaddresses) | `GET /api/v1/suppressions` | Lista las direcciones con supresión active, de la más reciente a la más antigua. | — | — |
| [purgeMessagePersonalData](#purgemessagepersonaldata) | `POST /api/v1/maintenance/purge-personal-data` | Vacía los datos personales de los mensajes más antiguos que el plazo de retención (personalDataRetentionMonths) y conserva la fila y su clave para siempre. Diario a las 03:00 UTC y a demanda del administrador. | — | — |

## Rol template-editor

Gestiona plantillas y sus versiones. No lee correos enviados.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [createTemplate](#createtemplate) | `POST /api/v1/templates` | Da de alta una plantilla vacía, sin versiones; no se puede usar hasta publicar la primera. | — | — |
| [publishTemplateVersion](#publishtemplateversion) | `POST /api/v1/templates/{templateCode}/versions` | Publica contenido nuevo para una plantilla. La versión nueva queda active y la que lo estaba pasa a archived, en la misma transacción. | EmailTemplateVersion: active → archived | — |
| [activateTemplateVersion](#activatetemplateversion) | `POST /api/v1/templates/{templateCode}/versions/{versionNumber}/activate` | Vuelve a poner en active una versión archivada (revertir) y archiva la que estuviera active. | EmailTemplateVersion: archived → active; EmailTemplateVersion: active → archived | — |
| [retireTemplate](#retiretemplate) | `POST /api/v1/templates/{templateCode}/retire` | Retira una plantilla archivando su versión active; conserva el historial y se puede volver a activar o publicar. | EmailTemplateVersion: active → archived | — |
| [getTemplate](#gettemplate) | `GET /api/v1/templates/{templateCode}` | Devuelve una plantilla sin su contenido. | — | — |
| [listTemplates](#listtemplates) | `GET /api/v1/templates` | Lista las plantillas, opcionalmente filtradas por si tienen versión active. | — | — |
| [getTemplateVersion](#gettemplateversion) | `GET /api/v1/templates/{templateCode}/versions/{versionNumber}` | Devuelve una versión concreta de una plantilla, con su contenido completo. | — | — |
| [listTemplateVersions](#listtemplateversions) | `GET /api/v1/templates/{templateCode}/versions` | Historial de versiones de una plantilla, de la más nueva a la más antigua, sin los cuerpos. | — | — |

## Rol notifications-operator

Soporte diario; consulta correos y gestiona supresiones.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [getTemplate](#gettemplate) | `GET /api/v1/templates/{templateCode}` | Devuelve una plantilla sin su contenido. | — | — |
| [listTemplates](#listtemplates) | `GET /api/v1/templates` | Lista las plantillas, opcionalmente filtradas por si tienen versión active. | — | — |
| [getTemplateVersion](#gettemplateversion) | `GET /api/v1/templates/{templateCode}/versions/{versionNumber}` | Devuelve una versión concreta de una plantilla, con su contenido completo. | — | — |
| [listTemplateVersions](#listtemplateversions) | `GET /api/v1/templates/{templateCode}/versions` | Historial de versiones de una plantilla, de la más nueva a la más antigua, sin los cuerpos. | — | — |
| [getMessage](#getmessage) | `GET /api/v1/messages/{id}` | Devuelve un mensaje por su id, sin los valores de las variables. | — | — |
| [listMessages](#listmessages) | `GET /api/v1/messages` | Lista los mensajes con filtros, del más reciente al más antiguo. | — | — |
| [suppressAddress](#suppressaddress) | `PUT /api/v1/suppressions/{address}` | Suprime una dirección a mano. Reactiva el registro si estaba released; si ya está active, no cambia nada. | SuppressedAddress: released → active | — |
| [releaseAddress](#releaseaddress) | `POST /api/v1/suppressions/{address}/release` | Libera una dirección suprimida; el registro se conserva. Liberar una dirección ya released no cambia nada. | SuppressedAddress: active → released | — |
| [getSuppressedAddress](#getsuppressedaddress) | `GET /api/v1/suppressions/{address}` | Devuelve el registro de supresión de una dirección, esté active o released. | — | — |
| [listSuppressedAddresses](#listsuppressedaddresses) | `GET /api/v1/suppressions` | Lista las direcciones con supresión active, de la más reciente a la más antigua. | — | — |

## Rol notifications-auditor

Solo lectura de todo el servicio.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [getSenderSettings](#getsendersettings) | `GET /api/v1/sender-settings` | Devuelve el remitente configurado. | — | — |
| [getTemplate](#gettemplate) | `GET /api/v1/templates/{templateCode}` | Devuelve una plantilla sin su contenido. | — | — |
| [listTemplates](#listtemplates) | `GET /api/v1/templates` | Lista las plantillas, opcionalmente filtradas por si tienen versión active. | — | — |
| [getTemplateVersion](#gettemplateversion) | `GET /api/v1/templates/{templateCode}/versions/{versionNumber}` | Devuelve una versión concreta de una plantilla, con su contenido completo. | — | — |
| [listTemplateVersions](#listtemplateversions) | `GET /api/v1/templates/{templateCode}/versions` | Historial de versiones de una plantilla, de la más nueva a la más antigua, sin los cuerpos. | — | — |
| [getMessage](#getmessage) | `GET /api/v1/messages/{id}` | Devuelve un mensaje por su id, sin los valores de las variables. | — | — |
| [listMessages](#listmessages) | `GET /api/v1/messages` | Lista los mensajes con filtros, del más reciente al más antiguo. | — | — |
| [getSuppressedAddress](#getsuppressedaddress) | `GET /api/v1/suppressions/{address}` | Devuelve el registro de supresión de una dirección, esté active o released. | — | — |
| [listSuppressedAddresses](#listsuppressedaddresses) | `GET /api/v1/suppressions` | Lista las direcciones con supresión active, de la más reciente a la más antigua. | — | — |

## Cliente máquina billing

Facturación; pide correos y reconcilia sus peticiones.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [requestNotification](#requestnotification) | `POST /api/v1/notifications` | Acepta una petición de envío de un sistema llamante, la congela en un EmailMessage queued y responde sin prometer la entrega. Entra igual por HTTP y por el canal de eventos. | — | — |
| [findMessageByIdempotencyKey](#findmessagebyidempotencykey) | `GET /api/v1/notifications/{idempotencyKey}` | Permite a un sistema llamante reconciliar el estado de una petición suya a partir de su idempotencyKey. | — | — |

## Cliente máquina shipping

Envíos; pide correos y reconcilia sus peticiones.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [requestNotification](#requestnotification) | `POST /api/v1/notifications` | Acepta una petición de envío de un sistema llamante, la congela en un EmailMessage queued y responde sin prometer la entrega. Entra igual por HTTP y por el canal de eventos. | — | — |
| [findMessageByIdempotencyKey](#findmessagebyidempotencykey) | `GET /api/v1/notifications/{idempotencyKey}` | Permite a un sistema llamante reconciliar el estado de una petición suya a partir de su idempotencyKey. | — | — |

## Cliente máquina dispatch-only

Sistema que solo pide correos, con el mínimo privilegio.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [requestNotification](#requestnotification) | `POST /api/v1/notifications` | Acepta una petición de envío de un sistema llamante, la congela en un EmailMessage queued y responde sin prometer la entrega. Entra igual por HTTP y por el canal de eventos. | — | — |

## Cliente máquina mail-provider

Proveedor de correo; notifica rebotes.

| Operación | Ruta | Qué hace | Cambia de estado | Condición |
|---|---|---|---|---|
| [reportEmailBounce](#reportemailbounce) | `POST /api/v1/messages/{messageId}/bounces` | El proveedor de correo notifica un rebote de un destinatario de un mensaje. Un rebote permanent suprime la dirección; uno transient solo se registra en el log. | SuppressedAddress: released → active | — |

## Lo que hace el servicio sin que nadie lo pida

| Operación | La dispara | Cambia de estado |
|---|---|---|
| [dispatchQueuedMessages](#dispatchqueuedmessages) | reloj `* * * * *` | EmailMessage: sending → failed |
| [sendQueuedMessage](#sendqueuedmessage) | la despacha dispatchQueuedMessages | EmailMessage: queued → sending; EmailMessage: sending → sent; EmailMessage: sending → failed; EmailMessage: queued → failed |

## Operaciones

### configureSenderSettings

Crea o reemplaza el remitente del servicio. Los mensajes ya aceptados conservan el remitente que congelaron.

- Tipo: command · `PUT /api/v1/sender-settings`
- Regla: senderAddress y replyToAddress se normalizan a minúsculas.
- Regla: Si no existe SenderSettings se crea; si existe, se reemplazan sus tres campos (un campo omitido queda vacío).
- Regla: Si dos primeras configuraciones simultáneas chocan en la clave constante, la segunda recibe CONCURRENT_MODIFICATION.
- Regla: La dirección tiene que estar verificada ante el proveedor de correo; el servicio no puede comprobarlo y confía en el administrador.

| Error | HTTP | Cuándo |
|---|---|---|
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó el remitente entre la lectura y la escritura. |

### getSenderSettings

Devuelve el remitente configurado.

- Tipo: query · `GET /api/v1/sender-settings`

| Error | HTTP | Cuándo |
|---|---|---|
| `SENDER_SETTINGS_NOT_CONFIGURED` | 404 | Todavía no se ha configurado el remitente. |

### createTemplate

Da de alta una plantilla vacía, sin versiones; no se puede usar hasta publicar la primera.

- Tipo: command · `POST /api/v1/templates`
- Regla: La plantilla nace con hasActiveVersion false.

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_ALREADY_EXISTS` | 409 | Ya existe una plantilla con ese code. |

### publishTemplateVersion

Publica contenido nuevo para una plantilla. La versión nueva queda active y la que lo estaba pasa a archived, en la misma transacción.

- Tipo: command · `POST /api/v1/templates/{templateCode}/versions`
- Cambia de estado: EmailTemplateVersion: active → archived
- Regla: versionNumber es el mayor versionNumber de la plantilla más uno, o 1 si no tiene versiones.
- Regla: Si la plantilla tenía una versión active, pasa a archived.
- Regla: hasActiveVersion de la plantilla queda true.
- Regla: El motor de plantillas solo sustituye variables por nombre; no evalúa expresiones, condicionales ni llamadas.
- Regla: Toda aparición de {{ en subject, htmlBody o textBody abre un marcador {{name}} bien formado: tras {{ vienen cero o más espacios (solo U+0020; un tabulador o un salto de línea no cuentan), un VariableName, cero o más espacios y }}. El texto se lee de izquierda a derecha y solo {{ abre un marcador: un }} fuera de un marcador es texto literal, así que {{name}}} es un marcador seguido de '}'. Cualquier otra forma, incluida la triple llave {{{name}}}, se rechaza con MALFORMED_TEMPLATE_MARKER: no existe forma de escribir un valor sin escapar.
- Regla: Orden de evaluación: plantilla existente (TEMPLATE_NOT_FOUND), después nombres repetidos en variables (DUPLICATE_TEMPLATE_VARIABLES), después marcadores mal formados (MALFORMED_TEMPLATE_MARKER), después marcadores sin declarar (UNDECLARED_TEMPLATE_VARIABLES).

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |
| `DUPLICATE_TEMPLATE_VARIABLES` | 422 | variables declara dos veces el mismo name. |
| `MALFORMED_TEMPLATE_MARKER` | 422 | subject, htmlBody o textBody contienen un {{ que no abre un marcador {{name}} bien formado, como la triple llave {{{name}}}. |
| `UNDECLARED_TEMPLATE_VARIABLES` | 422 | subject, htmlBody o textBody usan una variable que no está en variables. |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó la plantilla a la vez. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en active. (lo pone el generador) |

### activateTemplateVersion

Vuelve a poner en active una versión archivada (revertir) y archiva la que estuviera active.

- Tipo: command · `POST /api/v1/templates/{templateCode}/versions/{versionNumber}/activate`
- Cambia de estado: EmailTemplateVersion: archived → active; EmailTemplateVersion: active → archived
- Regla: Si la versión indicada ya está active, la operación no cambia nada y la devuelve.
- Regla: Una versión archived que no cumple la regla de marcadores bien formados de publishTemplateVersion (solo puede ser una publicada antes de la v0.2.0) no se reactiva y se rechaza con MALFORMED_TEMPLATE_MARKER.
- Regla: Orden de evaluación: plantilla existente (TEMPLATE_NOT_FOUND), después versión existente (TEMPLATE_VERSION_NOT_FOUND), después versión ya active (no-op), después marcadores bien formados (MALFORMED_TEMPLATE_MARKER).
- Regla: La versión que estaba active, si había una, pasa a archived.
- Regla: hasActiveVersion de la plantilla queda true.

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |
| `TEMPLATE_VERSION_NOT_FOUND` | 404 | La plantilla no tiene una versión con ese versionNumber. |
| `MALFORMED_TEMPLATE_MARKER` | 422 | La versión archived contiene un {{ que no abre un marcador {{name}} bien formado (publicada antes de la v0.2.0). |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó la plantilla a la vez. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en archived, active. (lo pone el generador) |

### retireTemplate

Retira una plantilla archivando su versión active; conserva el historial y se puede volver a activar o publicar.

- Tipo: command · `POST /api/v1/templates/{templateCode}/retire`
- Cambia de estado: EmailTemplateVersion: active → archived
- Regla: hasActiveVersion de la plantilla queda false.
- Regla: Los mensajes ya aceptados con esta plantilla se envían igualmente; congelaron su versión.

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |
| `TEMPLATE_NOT_PUBLISHED` | 409 | La plantilla no tiene ninguna versión active. |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó la plantilla a la vez. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en active. (lo pone el generador) |

### getTemplate

Devuelve una plantilla sin su contenido.

- Tipo: query · `GET /api/v1/templates/{templateCode}`

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |

### listTemplates

Lista las plantillas, opcionalmente filtradas por si tienen versión active.

- Tipo: query · `GET /api/v1/templates`

### getTemplateVersion

Devuelve una versión concreta de una plantilla, con su contenido completo.

- Tipo: query · `GET /api/v1/templates/{templateCode}/versions/{versionNumber}`

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |
| `TEMPLATE_VERSION_NOT_FOUND` | 404 | La plantilla no tiene una versión con ese versionNumber. |

### listTemplateVersions

Historial de versiones de una plantilla, de la más nueva a la más antigua, sin los cuerpos.

- Tipo: query · `GET /api/v1/templates/{templateCode}/versions`

| Error | HTTP | Cuándo |
|---|---|---|
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |

### requestNotification

Acepta una petición de envío de un sistema llamante, la congela en un EmailMessage queued y responde sin prometer la entrega. Entra igual por HTTP y por el canal de eventos.

- Tipo: command · `POST /api/v1/notifications`
- Regla: Orden de evaluación: (1) forma de la petición y variables repetidas; (2) comprobar la repetición; (3) remitente, plantilla, variables, asunto renderizado (vacío, después longitud) y supresiones.
- Regla: Los destinatarios se normalizan a minúsculas y se deduplican antes de cualquier comparación; recipients conserva el orden de llegada y, ante duplicados, la primera aparición.
- Regla: La comprobación de supresiones se hace para todos los destinatarios a la vez, no una consulta por destinatario.
- Regla: Mismo contenido significa mismo templateCode, mismo conjunto de destinatarios normalizados y mismo conjunto de pares (name, value) de las variables declaradas por la versión congelada, sin importar el orden.
- Regla: Si el mensaje existente ya está purgado (personalDataPurgedAt con valor), mismo contenido significa solo mismo templateCode, porque es lo único que queda para comparar; nunca se envía un segundo correo.
- Regla: Dos valores para el mismo name en variables son DUPLICATE_TEMPLATE_VARIABLES, y se comprueba antes de la repetición.
- Regla: Si ya existe un mensaje con el mismo (requestedBy, idempotencyKey) y el mismo contenido, se devuelve ese mensaje sin crear otro, aunque desde entonces la plantilla se haya retirado, el remitente haya cambiado o un destinatario se haya suprimido.
- Regla: Las variables que la versión active no declara se descartan en silencio y no se guardan.
- Regla: El asunto se renderiza al aceptar con la versión active, sustituyendo cada marcador {{name}} por su valor (cadena vacía si es una opcional sin valor) y reemplazando por un espacio cada salto de línea de los valores (\r\n, \n y un \r suelto cuentan cada uno como un solo salto); los cuerpos se renderizan al enviar con esa misma versión, que es inmutable.
- Regla: El mensaje congela: el remitente (address, name y replyToAddress de SenderSettings), templateCode, versionNumber e id de la versión active.
- Regla: El mensaje queda en queued con requestedAt; la entrega ocurre después, en sendQueuedMessage.

| Error | HTTP | Cuándo |
|---|---|---|
| `DUPLICATE_TEMPLATE_VARIABLES` | 422 | variables trae dos valores con el mismo name. |
| `IDEMPOTENCY_KEY_REUSED` | 409 | Ya existe un mensaje con el mismo (requestedBy, idempotencyKey) y distinto contenido. |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | 409 | Otra petición con el mismo (requestedBy, idempotencyKey) se está aceptando a la vez. |
| `SENDER_SETTINGS_NOT_CONFIGURED` | 404 | Todavía no se ha configurado el remitente. |
| `TEMPLATE_NOT_FOUND` | 404 | No existe una plantilla con ese templateCode. |
| `TEMPLATE_NOT_PUBLISHED` | 409 | La plantilla no tiene ninguna versión active. |
| `MISSING_TEMPLATE_VARIABLES` | 422 | Falta alguna variable que la versión active declara required. |
| `RENDERED_SUBJECT_TOO_LONG` | 422 | El asunto renderizado pasa de 300 caracteres; no se recorta y no se acepta el mensaje. |
| `EMPTY_RENDERED_SUBJECT` | 422 | El asunto renderizado queda vacío o solo con espacios (p. ej. un subject que es solo el marcador de una variable opcional que no llega); no se acepta un correo sin asunto. |
| `RECIPIENT_SUPPRESSED` | 422 | Algún destinatario tiene una supresión active. |

### dispatchQueuedMessages

Barrido de envío, cada minuto. Primero rescata los mensajes atascados en sending y los da por fallidos sin reenviarlos; después envía los queued más antiguos.

- Tipo: command · no se llama por HTTP
- Cambia de estado: EmailMessage: sending → failed
- Regla: Rescate: todo mensaje en sending con sendingSince anterior a ahora menos sendingTimeoutMinutes pasa a failed con failureReason stuck-in-sending y failedAt, y publica EmailDeliveryFailed. Nunca se reenvía: no se sabe si el correo llegó a salir.
- Regla: Solo lo dispara el reloj; no tiene puerta HTTP ni de eventos.
- Regla: Después toma hasta 200 mensajes queued, por requestedAt ascendente y id ascendente, y ejecuta sendQueuedMessage con cada uno.
- Regla: Si una ejecución se solapa con la anterior, cada mensaje lo envía como mucho una de las dos; la guarda es la transición queued → sending de sendQueuedMessage.
- Regla: Lo que no cabe en un ciclo queda queued para el siguiente; tras una parada, el primer ciclo recupera los queued pendientes y rescata los sending atascados.

| Error | HTTP | Cuándo |
|---|---|---|
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en sending. (lo pone el generador) |

### sendQueuedMessage

Envía un mensaje queued. Es la única operación que produce correo.

- Tipo: command · no se llama por HTTP
- Cambia de estado: EmailMessage: queued → sending; EmailMessage: sending → sent; EmailMessage: sending → failed; EmailMessage: queued → failed
- Regla: Si algún destinatario tiene ahora una supresión active, el mensaje pasa de queued a failed con failureReason recipient-suppressed y failedAt, se publica EmailDeliveryFailed y no se envía nada.
- Regla: En otro caso pasa a sending con sendingSince, y esa transición se confirma antes de entregar el correo al relay. Quien pierde la transición no envía nada.
- Regla: Las dos salidas de queued (a failed por supresión y a sending) compiten por la misma transición condicionada a queued. Si dos ejecuciones procesan el mismo mensaje a la vez, gana la primera que la confirma; la otra la pierde, termina como MESSAGE_NOT_QUEUED y no envía ni publica nada.
- Regla: Renderiza los cuerpos con la versión congelada y variableValues, y entrega un único correo html+text con todos los recipients en To, el remitente y el reply-to congelados, y un Message-ID cuya parte local (lo que va antes de la arroba) es el id del EmailMessage, para que el proveedor pueda asociar los rebotes; el dominio del Message-ID es despliegue.
- Regla: La sustitución es de una sola pasada, igual que la del asunto al aceptar: el valor insertado nunca se vuelve a interpretar, aunque contenga {{otra}}. Solo se sustituyen los marcadores bien formados; un {{ mal formado de una versión anterior a la v0.2.0 que siga active va como texto literal (escapado en htmlBody), nunca sin escapar.
- Regla: Cada marcador {{name}} se sustituye por su valor; una variable opcional sin valor se sustituye por cadena vacía. En htmlBody los valores se escapan como texto HTML escapando solo los caracteres & < > " ' (como &amp; &lt; &gt; &quot; &#39;); el resto, incluidos acentos y símbolos como €, va literal. El asunto ya viene renderizado desde la aceptación (renderedSubject); el To lleva los recipients en su orden.
- Regla: El intento contra el relay tiene un tope de 1 minuto, muy por debajo de sendingTimeoutMinutes.
- Regla: Si el relay acepta el correo para al menos un destinatario, pasa a sent con sentAt y publica EmailSent; los destinatarios que rechazó se tratan como rebotes y llegan por reportEmailBounce.
- Regla: Si el relay lo rechaza para todos los destinatarios, pasa a failed con delivery-rejected; si el intento falla o agota su tope sin respuesta concluyente, con delivery-error. En ambos casos fija failedAt, publica EmailDeliveryFailed y no se reintenta.
- Regla: Si el relay acepta después de que el rescate ya cerrara el mensaje como failed, el mensaje sigue failed y no se publica EmailSent; se acepta ese desajuste a cambio de no reenviar nunca.
- Regla: Si el proceso cae tras confirmar sending, el mensaje queda en sending hasta que el rescate lo da por fallido.

| Error | HTTP | Cuándo |
|---|---|---|
| `MESSAGE_NOT_QUEUED` | — | El mensaje no existe o ya no está en queued; el barrido lo salta sin hacer nada. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en queued, sending. (lo pone el generador) |

### getMessage

Devuelve un mensaje por su id, sin los valores de las variables.

- Tipo: query · `GET /api/v1/messages/{id}`

| Error | HTTP | Cuándo |
|---|---|---|
| `MESSAGE_NOT_FOUND` | 404 | No existe un mensaje con ese id. |

### listMessages

Lista los mensajes con filtros, del más reciente al más antiguo.

- Tipo: query · `GET /api/v1/messages`
- Regla: recipient casa con los mensajes cuya lista recipients contiene esa dirección; un mensaje purgado no casa nunca.

| Error | HTTP | Cuándo |
|---|---|---|
| `INVALID_TIME_WINDOW` | 422 | requestedFrom es posterior o igual a requestedTo. |

### findMessageByIdempotencyKey

Permite a un sistema llamante reconciliar el estado de una petición suya a partir de su idempotencyKey.

- Tipo: query · `GET /api/v1/notifications/{idempotencyKey}`
- Regla: Solo encuentra mensajes del propio llamante.

| Error | HTTP | Cuándo |
|---|---|---|
| `MESSAGE_NOT_FOUND` | 404 | El llamante no tiene ningún mensaje con esa idempotencyKey. |

### suppressAddress

Suprime una dirección a mano. Reactiva el registro si estaba released; si ya está active, no cambia nada.

- Tipo: command · `PUT /api/v1/suppressions/{address}`
- Cambia de estado: SuppressedAddress: released → active
- Regla: address se normaliza a minúsculas.
- Regla: Si no existe registro, se crea active con suppressedAt ahora.
- Regla: Si existe released, pasa a active con el reason nuevo, suppressedAt ahora y releasedAt vacío; notes se reemplaza por el nuevo (omitido deja notes vacío).
- Regla: Si existe active, se devuelve sin cambios.
- Regla: reason complaint solo entra por aquí, a mano; no hay canal de quejas del proveedor.

| Error | HTTP | Cuándo |
|---|---|---|
| `SUPPRESSED_ADDRESS_ALREADY_EXISTS` | 409 | Otra supresión de la misma dirección creó el registro a la vez. |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó el registro a la vez. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en released. (lo pone el generador) |

### releaseAddress

Libera una dirección suprimida; el registro se conserva. Liberar una dirección ya released no cambia nada.

- Tipo: command · `POST /api/v1/suppressions/{address}/release`
- Cambia de estado: SuppressedAddress: active → released
- Regla: address se normaliza a minúsculas.
- Regla: releasedAt queda con la fecha de la liberación.
- Regla: Si el registro ya está released, se devuelve sin cambios.

| Error | HTTP | Cuándo |
|---|---|---|
| `SUPPRESSED_ADDRESS_NOT_FOUND` | 404 | No existe ningún registro de supresión para esa dirección. |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó el registro a la vez. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en active. (lo pone el generador) |

### getSuppressedAddress

Devuelve el registro de supresión de una dirección, esté active o released.

- Tipo: query · `GET /api/v1/suppressions/{address}`
- Regla: address se normaliza a minúsculas.

| Error | HTTP | Cuándo |
|---|---|---|
| `SUPPRESSED_ADDRESS_NOT_FOUND` | 404 | No existe ningún registro de supresión para esa dirección. |

### listSuppressedAddresses

Lista las direcciones con supresión active, de la más reciente a la más antigua.

- Tipo: query · `GET /api/v1/suppressions`

### reportEmailBounce

El proveedor de correo notifica un rebote de un destinatario de un mensaje. Un rebote permanent suprime la dirección; uno transient solo se registra en el log.

- Tipo: command · `POST /api/v1/messages/{messageId}/bounces`
- Cambia de estado: SuppressedAddress: released → active
- Regla: recipient se normaliza a minúsculas.
- Regla: Con bounceType transient no cambia ningún estado persistente; solo se registra en el log.
- Regla: Con bounceType permanent, si no existe registro se crea active con reason hard-bounce; si existe released se reactiva con reason hard-bounce aunque lo hubiera liberado una persona; si ya está active no cambia nada.
- Regla: Al crear o reactivar por rebote, suppressedAt es ahora, releasedAt queda vacío y notes se reemplaza por el id del mensaje, un espacio y el diagnostic; si no cabe en 1000 caracteres, se recorta el diagnostic por el final, de modo que el id siempre queda: un diagnostic largo nunca impide la supresión.
- Regla: El proveedor real llega a través de un adaptador de despliegue que traduce su webhook a este contrato; el messageId es la parte local del Message-ID del correo rebotado.
- Regla: El rebote no cambia el estado del mensaje.

| Error | HTTP | Cuándo |
|---|---|---|
| `MESSAGE_NOT_FOUND` | 404 | No existe un mensaje con ese messageId. |
| `RECIPIENT_NOT_IN_MESSAGE` | 422 | recipient no está entre los recipients del mensaje, incluido el caso de un mensaje ya purgado. |
| `SUPPRESSED_ADDRESS_ALREADY_EXISTS` | 409 | Otra supresión de la misma dirección creó el registro a la vez. |
| `CONCURRENT_MODIFICATION` | 409 | Otra escritura modificó el registro a la vez. |
| `INVALID_STATE_TRANSITION` | 409 | La entidad no está en released. (lo pone el generador) |

### purgeMessagePersonalData

Vacía los datos personales de los mensajes más antiguos que el plazo de retención (personalDataRetentionMonths) y conserva la fila y su clave para siempre. Diario a las 03:00 UTC y a demanda del administrador.

- Tipo: command · `POST /api/v1/maintenance/purge-personal-data`
- Regla: Selecciona hasta 5000 mensajes en sent o failed con requestedAt anterior a ahora menos personalDataRetentionMonths meses de calendario y personalDataPurgedAt vacío, por requestedAt ascendente e id ascendente.
- Regla: Si dos ejecuciones se solapan (la programada y la manual), ninguna falla por ello; cada una salta los mensajes que la otra ya purgó o tiene en curso, y purgedCount cuenta solo los suyos.
- Regla: A cada uno le vacía recipients, renderedSubject y variableValues y fija personalDataPurgedAt.
- Regla: Repetirla no tiene efectos sobre mensajes ya purgados; lo que no cabe en un ciclo lo recoge el siguiente.
- Regla: La clave (requestedBy, idempotencyKey) se conserva, así que una petición repetida tras la purga sigue sin enviar un segundo correo.
