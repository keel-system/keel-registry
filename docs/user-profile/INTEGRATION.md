---
service: user-profile
version: 0.1.0
domain: identity
basePath: /api/v1
m2mAuth:
  protocol: client-credentials
  audience: user-profile
  validateAudience: true
endpoints:
  - name: resolveContactForServices
    method: GET
    path: /services/profiles/{subject}/contact
    access: service profile-contact:read
  - name: resolveContactsBatchForServices
    method: POST
    path: /services/profiles/contact/batch
    access: service profile-contact:read
  - name: resolveDeliveryProfileForServices
    method: GET
    path: /services/profiles/{subject}/delivery
    access: service profile-delivery:read
  - name: resolveDeliveryProfilesBatchForServices
    method: POST
    path: /services/profiles/delivery/batch
    access: service profile-delivery:read
events:
  envelope: keel
  published:
    - name: ProfileProvisioned
      channel: profileEvents
    - name: ProfileContactEmailRefreshed
      channel: profileEvents
    - name: ProfileContactDetailsChanged
      channel: profileEvents
    - name: AddressAdded
      channel: profileEvents
    - name: AddressChanged
      channel: profileEvents
    - name: AddressRemoved
      channel: profileEvents
    - name: DefaultAddressChanged
      channel: profileEvents
    - name: ProfileDeactivated
      channel: profileEvents
    - name: ProfileReactivated
      channel: profileEvents
    - name: ProfileDeleted
      channel: profileEvents
  consumed: []
errors:
  - code: PROFILE_NOT_FOUND
    http: 404
  - code: PROFILE_DELETED
    http: 410
  - code: PROFILE_INCOMPLETE
    http: 409
  - code: TOO_MANY_SUBJECTS
    http: 422
---

# Integración con user-profile

## Resumen

`user-profile` posee el perfil de negocio de cada persona —contacto y direcciones— y adopta, por aprovisionamiento just-in-time, la identidad que crea un servidor de identidad. A otros servidores les ofrece **dos familias de consulta separadas por scope** —el contacto de una persona, y el contacto más su dirección de envío por defecto— en versión individual y por lotes de hasta cien; y **diez eventos** que cubren toda la superficie de mutación del perfil, para que quien lo necesite pueda mantener una réplica de solo lectura sin llamarnos en cada cambio.

Tres cosas que conviene saber antes de integrarse, porque cambian lo que puedes esperar:

- **Este servicio nunca llama al servidor de identidad.** Valida la firma del token en local contra las claves públicas del emisor. La consecuencia que te afecta: una identidad borrada en la consola del servidor de identidad **no** llega aquí, y el procedimiento correcto es el contrario — la baja se inicia contra esta API y deshabilitar la identidad es un acto aparte.
- **El perfil incompleto es un estado válido**, no un fallo. Una persona puede existir sin teléfono y sin ninguna dirección. Las operaciones que exigen dirección te lo dicen con un error de dominio propio y nunca con un `500`.
- **Este servicio no valida la división territorial contra ningún catálogo.** Ver [§Sobre los datos territoriales](#sobre-los-datos-territoriales): es contrato, y afecta a lo que recibes.

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client credentials), no con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del endpoint de token del servidor de identidad (`tokenUrl`), que varía por entorno y no forma parte del diseño.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los `scopes` que tu cliente tiene concedidos (ver tabla). Fija la audiencia `aud: user-profile`: **se valida**, así que un token legítimo emitido para otro servicio responde `403` y no `401`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=profile-delivery:read&audience=user-profile
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| `order-service` | `profile-delivery:read` | Resuelve contacto y dirección de envío por defecto para congelarlos como snapshot del pedido, de uno en uno o por lotes al pintar un listado. |
| `notification-service` | `profile-contact:read` | Resuelve el contacto de la persona a la que va dirigido un aviso; no necesita ni recibe direcciones postales. |

**Las dos familias están separadas en la ruta y en el scope, y eso es deliberado.** El mínimo privilegio a nivel de campo solo se puede expresar como operación aparte: un consumidor con `profile-contact:read` **no puede recibir una dirección postal ni pidiéndola**, porque no existe ninguna operación que se la devuelva. Si tu servicio necesita las dos cosas, pide los dos scopes; si solo necesita contacto, no pidas el de entrega.

**El scope alcanza a cualquier subject.** No hay acotación por consumidor: quien tiene `profile-contact:read` puede resolver el contacto de **cualquier** persona del padrón, no solo de las que le conciernen. Acotarlo exigiría que este servicio supiera qué pedidos tienes o a quién notificas, y eso es dato tuyo: nos volvería dependientes de nuestros propios consumidores. Trata la credencial en consecuencia.

**La clave es siempre el `subject`**, el identificador opaco que emite el servidor de identidad y que viaja en el claim `sub` del token de la persona. Nunca el email: el email cambia de dueño, y uno reutilizado reasignaría el perfil a otra persona. Es opaco: no lo interpretes, no lo parsees y percent-encodéalo al ponerlo en una ruta.

### resolveContactForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/profiles/{subject}/contact` |
| Acceso | `service` — scopes `profile-contact:read` |
| Idempotencia | no aplica (query) |
| Caché del servidor | 60 s, invalidada por los diez eventos del perfil |

**Request** — path `subject: string` (requerido, opaco, percent-encoded).

**Response**

| Campo | Tipo | Notas |
|---|---|---|
| `profile.subject` | string | requerido |
| `profile.displayName` | string \| null | nombre con el que dirigirse a la persona |
| `profile.givenName` | string \| null | |
| `profile.familyName` | string \| null | |
| `profile.contactEmail` | string \| null | réplica del claim del token; **puede ser nulo** si el token nunca trajo ese claim |
| `profile.contactPhone` | string \| null | E.164; dato de negocio, nunca una credencial |
| `profile.status` | enum | requerido — `draft` \| `complete` \| `deactivated` |
| `profile.version` | int | requerido — versión del agregado al responder |
| `profile.updatedAt` | timestamp | requerido — ISO-8601 UTC |

```json
{
  "profile": {
    "subject": "sub-sara",
    "displayName": "Sara Gil",
    "givenName": "Sara",
    "familyName": "Gil",
    "contactEmail": "sara@example.com",
    "contactPhone": "+34600111222",
    "status": "complete",
    "version": 7,
    "updatedAt": "2026-03-14T09:21:07.482Z"
  }
}
```

**Todo campo declarado viaja, nulo si no tiene valor; nunca omitido.** No distingas «no vino» de «vino vacío»: la segunda es la única que ocurre.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `PROFILE_NOT_FOUND` | 404 | No existe perfil con ese subject y tampoco hay lápida: esa persona nunca ha entrado. | No reintentar. Puede aparecer más adelante, cuando entre por primera vez. |
| `PROFILE_DELETED` | 410 | Ese subject tuvo perfil y fue borrado. | No reintentar. **Purga tus propias copias de esa persona.** |

Distinguir `404` de `410` es contrato y no detalle de implementación: es lo que te permite purgar las copias de quien se borró sin purgar las de quien todavía no ha entrado.

### resolveContactsBatchForServices

| | |
|---|---|
| Endpoint | `POST /api/v1/services/profiles/contact/batch` |
| Acceso | `service` — scopes `profile-contact:read` |
| Idempotencia | no aplica (query) |
| Caché del servidor | ninguna — una lista de hasta cien subjects no es una clave de caché útil |
| Cota | de 1 a 100 subjects |

Es `POST` y no `GET` porque cien subjects opacos no caben con garantías en una query string. **Es también el arranque y la reconciliación de una réplica**: los eventos solo dan el delta desde que te suscribiste, así que si llegas después, si pierdes mensajes o si sospechas que tu copia diverge, cargas y reconcilias por aquí.

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| `subjects` | string[] | requerido, de 1 a 100 elementos |

```json
{ "subjects": ["sub-sara", "sub-tomas", "sub-nadie"] }
```

**Response** — `profiles[]` con la misma forma que el `profile` de [`resolveContactForServices`](#resolvecontactforservices), y `unresolved[]` con los que no se pudieron resolver.

| Campo | Tipo | Notas |
|---|---|---|
| `profiles` | objeto[] | requerido; **en el orden de la petición** |
| `unresolved` | objeto[] | requerido; **en el orden de la petición** |
| `unresolved[].subject` | string | requerido |
| `unresolved[].reason` | enum | requerido — `not-found` \| `deleted` (`incomplete` no aplica a esta familia) |

```json
{
  "profiles": [
    {
      "subject": "sub-sara", "displayName": "Sara Gil", "givenName": "Sara", "familyName": "Gil",
      "contactEmail": "sara@example.com", "contactPhone": "+34600111222",
      "status": "complete", "version": 7, "updatedAt": "2026-03-14T09:21:07.482Z"
    }
  ],
  "unresolved": [
    { "subject": "sub-tomas", "reason": "deleted" },
    { "subject": "sub-nadie", "reason": "not-found" }
  ]
}
```

**El lote nunca falla entero por un subject que no resuelve.** Un subject repetido en la petición se resuelve una sola vez y aparece una sola vez en la respuesta.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `TOO_MANY_SUBJECTS` | 422 | La petición trae más de cien subjects. | Corregir input: parte el lote en trozos de 100. |

Una lista **vacía** no lleva código propio: la rechaza la validación de forma con un `400` genérico, porque es una petición mal construida y no una decisión de negocio.

### resolveDeliveryProfileForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/profiles/{subject}/delivery` |
| Acceso | `service` — scopes `profile-delivery:read` |
| Idempotencia | no aplica (query) |
| Caché del servidor | 60 s, invalidada por los diez eventos del perfil |

**Request** — path `subject: string` (requerido, opaco, percent-encoded).

**Response** — los nueve campos de `resolveContactForServices` más `shippingAddress`.

| Campo | Tipo | Notas |
|---|---|---|
| `profile.shippingAddress` | objeto | requerido — la dirección `shipping` marcada por defecto |
| `…shippingAddress.line1` | string | requerido — vía y número |
| `…shippingAddress.line2` | string \| null | complemento (piso, puerta, torre) |
| `…shippingAddress.line3` | string \| null | referencias adicionales de entrega |
| `…shippingAddress.postalCode` | string \| null | opcional: hay países donde no existe y países donde nadie lo escribe |
| `…shippingAddress.countryCode` | string | requerido — ISO 3166-1 alpha-2 |
| `…shippingAddress.adminAreaCode` | string \| null | división de primer nivel, ISO 3166-2 (departamento, provincia, state) |
| `…shippingAddress.adminAreaName` | string \| null | su nombre; viene siempre que venga el código |
| `…shippingAddress.localityCode` | string \| null | división de segundo nivel en la **codificación nacional del país declarado** |
| `…shippingAddress.localityName` | string | requerido — no hay país donde una dirección postal no nombre una localidad |

```json
{
  "profile": {
    "subject": "sub-tomas",
    "displayName": "Tomás Rivas",
    "givenName": "Tomás",
    "familyName": "Rivas",
    "contactEmail": "tomas@example.com",
    "contactPhone": "+573001112233",
    "status": "complete",
    "version": 12,
    "updatedAt": "2026-03-14T09:21:07.482Z",
    "shippingAddress": {
      "line1": "Calle 10 # 43-25",
      "line2": "Apto 501",
      "line3": null,
      "postalCode": null,
      "countryCode": "CO",
      "adminAreaCode": "CO-ANT",
      "adminAreaName": "Antioquia",
      "localityCode": "05001",
      "localityName": "Medellín"
    }
  }
}
```

**La dirección viaja entera, códigos incluidos, y sin `id`, `label` ni `isDefault`.** Los códigos están porque tú la vas a congelar como dato del pedido y no vas a poder volver a pedirla: un snapshot sin códigos no sirve después para liquidar impuestos ni para enrutar reparto. Los tres campos que faltan son de la UX con la que el usuario eligió esa dirección, no tuyos.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `PROFILE_NOT_FOUND` | 404 | No existe perfil con ese subject y tampoco hay lápida. | No reintentar. |
| `PROFILE_DELETED` | 410 | Ese subject tuvo perfil y fue borrado. | No reintentar. **Purga tus propias copias.** |
| `PROFILE_INCOMPLETE` | 409 | El perfil existe pero no tiene dirección de tipo `shipping` marcada por defecto; la respuesta enumera qué falta. | No reintentar con los mismos datos. Pídele al usuario que complete su perfil y vuelve a intentarlo después. |

`PROFILE_INCOMPLETE` no es un fallo del servidor y por eso no es un `500`: el perfilado progresivo es parte del diseño, y este código es lo que te permite saber qué modal abrir en vez de recibir errores aleatorios.

### resolveDeliveryProfilesBatchForServices

| | |
|---|---|
| Endpoint | `POST /api/v1/services/profiles/delivery/batch` |
| Acceso | `service` — scopes `profile-delivery:read` |
| Idempotencia | no aplica (query) |
| Caché del servidor | ninguna |
| Cota | de 1 a 100 subjects |

Para que un listado de cien pedidos no sean cien llamadas. Sirve también de arranque y reconciliación de una réplica de solo lectura.

**Request** — idéntico al del lote de contacto.

**Response** — `profiles[]` con la forma de [`resolveDeliveryProfileForServices`](#resolvedeliveryprofileforservices), y `unresolved[]`.

| Campo | Tipo | Notas |
|---|---|---|
| `profiles` | objeto[] | requerido; en el orden de la petición |
| `unresolved` | objeto[] | requerido; en el orden de la petición |
| `unresolved[].reason` | enum | `not-found` \| `deleted` \| **`incomplete`** |

```json
{
  "profiles": [ { "subject": "sub-tomas", "…": "…", "shippingAddress": { "…": "…" } } ],
  "unresolved": [
    { "subject": "sub-nadie", "reason": "not-found" },
    { "subject": "sub-quique", "reason": "deleted" },
    { "subject": "sub-ulises", "reason": "incomplete" }
  ]
}
```

**Un perfil sin dirección de envío por defecto sale en `unresolved` con `reason: incomplete`, no se omite en silencio.** Tienes que poder distinguir «no hay nadie» de «hay alguien al que le falta la dirección»: la segunda se arregla pidiéndole al usuario que complete su perfil, y la primera no.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `TOO_MANY_SUBJECTS` | 422 | La petición trae más de cien subjects. | Corregir input: parte el lote en trozos de 100. |

### Sobre los datos territoriales

**Este servicio valida la FORMA de los datos territoriales y nunca su EXISTENCIA, y es una decisión, no un olvido.** Comprueba que el código de primer nivel, cuando viene, sea un ISO 3166-2 del país declarado; que los nombres no vengan vacíos; que no llegue un código sin su nombre. Y nada más.

Lo que eso significa para ti, dicho sin rodeos: **el servidor acepta `"localityName": "Medallo"` sin rechistar**, y te lo devolverá tal cual. Que el territorio exista de verdad, que pertenezca a ese departamento y que el nombre sea el oficial es responsabilidad de quien escribió la dirección, que lo resuelve en el borde contra su propio catálogo territorial — que es además donde está la UX de elegir municipio.

Tres consecuencias prácticas:

- **No asumas que un `localityCode` resuelve contra tu catálogo.** El país es el esquema implícito: `localityCode` se interpreta contra la codificación nacional del país de `countryCode`, y la dirección no lleva ningún identificador de esquema aparte. En un país con más de una codificación en circulación, el código puede quedar huérfano — y por eso el **nombre viaja siempre al lado**, denormalizado, para que la dirección siga siendo imprimible aunque el código deje de ser resoluble.
- **El nombre es un hecho histórico, no una referencia viva.** Se copió en el momento de guardar y no se vuelve a resolver nunca, aunque el territorio se renombre o se fusione después. Que código y nombre diverjan con los años no es una inconsistencia que haya que arreglar: es la historia, y es lo que permite reimprimir una dirección vieja como se emitió.
- **Los niveles tienen nombres genéricos a propósito.** `adminArea` es departamento en Colombia, provincia en España y state en EE. UU.; `locality` es municipio, ayuntamiento o city. Hay países que no tienen el primero, y por eso `adminAreaCode` y `adminAreaName` son opcionales mientras que `localityName` no lo es.

### Qué puedes hacer con estos datos, y qué no

Un consumidor **no puede mantener su propia versión mutable del perfil**: con dos escritores no hay fuente de verdad. Pero sí hay copias legítimas, y el contrato distingue las tres:

| | Qué es | ¿Legítima? |
|---|---|---|
| **Snapshot inmutable** | La dirección congelada en el momento del pedido. No es una copia del perfil: es un dato del pedido, y no cambia aunque el perfil cambie. | **Sí.** Es lo que `resolveDeliveryProfileForServices` existe para darte, y por eso la dirección viaja entera. |
| **Réplica de solo lectura** | Una copia alimentada por los eventos de este servicio, donde **user-profile sigue siendo el único escritor**. | **Sí**, y lo es porque la lista de eventos cubre toda la superficie de mutación. Arranca y reconcilia por el lote; aplica `ProfileDeleted` purgando; usa `version` para descartar lo que llegue tarde. |
| **Réplica mutable** | Una copia que tu servicio edita por su cuenta. | **No.** Con dos escritores no hay fuente de verdad, y el perfil deja de tener dueño. |

## Eventos

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProfileProvisioned",
    "eventVersion": 1,
    "occurredAt": "2026-03-14T09:21:07.482Z",
    "source": "user-profile",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": { "subject": "sub-sara", "status": "draft", "version": 1 }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `metadata.eventId` | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| `metadata.eventType` | string | Nombre del evento (`ProfileProvisioned`). Discriminador: los diez viajan por el mismo canal. |
| `metadata.eventVersion` | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| `metadata.occurredAt` | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. Con outbox, los dos pueden distar. |
| `metadata.source` | string | Servicio emisor: `user-profile`. Es procedencia declarada, **no identidad verificada**: no autorices con ella. |
| `metadata.correlationId` | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| `data` | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Canal lógico: `profileEvents`.** Los diez eventos viajan por él. El broker concreto y el nombre físico del topic o cola son parámetro de despliegue y no forman parte del diseño; pídeselos al dueño del servicio.

**Garantía de entrega: `outbox`.** Ningún evento se pierde si la transacción confirma: la fila del evento se escribe en la misma transacción que el cambio del agregado y un relay la publica después. Lo fija `ProfileDeleted`, que es la señal con la que el resto del sistema purga datos personales.

**Tres propiedades del conjunto que te conviene conocer antes de leer los diez:**

1. **La lista es completa sobre la superficie de mutación.** Toda vía por la que el dato cambia emite un evento; ninguna queda fuera. De eso depende que una réplica de solo lectura no se pudra, y por eso la lista no se recorta sin subir la versión mayor.
2. **Todos llevan `version`**, que es la versión del agregado tras el cambio y el mismo entero que devuelven los endpoints M2M. Úsala para descartar un evento más viejo que el estado que ya aplicaste: dos eventos del mismo perfil pueden llegar desordenados, y sin esa comparación tu copia queda mal para siempre y sin síntoma.
3. **Ninguna operación publica dos eventos a la vez**, así que `version` es única por agregado y el orden entre eventos del mismo perfil es total.

El contrato formal de estos mensajes está en [`asyncapi.yaml`](asyncapi.yaml) (y su visor, [`asyncapi.html`](asyncapi.html)); lo de abajo es la prosa para integrarse.

### ProfileProvisioned

Nació un perfil al adoptar una identidad en la primera petición autenticada de esa persona. Es el primer hecho de todo perfil y el que arranca cualquier réplica.

**Emitido por**: `provisionProfileFromIdentity`.

El perfil nace con lo que el token traiga: solo el `subject` es obligatorio, así que `contactEmail` puede llegar nulo si el cliente no pidió ese scope o el proveedor no lo emite. `contactPhone` nunca viene en el alta.

```json
{
  "subject": "sub-sara",
  "contactEmail": "sara@example.com",
  "givenName": "Sara",
  "familyName": "Gil",
  "displayName": "Sara Gil",
  "contactPhone": null,
  "status": "draft",
  "version": 1
}
```

### ProfileContactEmailRefreshed

El correo de contacto se refrescó desde el claim del token durante un aprovisionamiento.

**Emitido por**: `provisionProfileFromIdentity`.

Este evento existe porque el aprovisionamiento corre en **toda** petición autenticada y refresca el `contactEmail` cuando el claim difiere del almacenado: el dato cambia sin pasar por ningún command. Si lo ignoras, tu copia conservará el correo viejo indefinidamente — sin error, sin traza, y lo descubrirás el día que un correo no llegue a quien debía. Ocurre en cualquier estado del perfil, incluido `deactivated`.

```json
{ "subject": "sub-sara", "contactEmail": "sara.nueva@example.com", "version": 8 }
```

### ProfileContactDetailsChanged

La persona reemplazó su bloque de contacto editable. No incluye el correo, que no se edita por la API.

**Emitido por**: `updateMyContactDetails`.

Es un reemplazo completo, no un delta: un campo que llegue nulo está **vacío**, no «sin cambios». El `status` viaja aquí porque los cambios de completitud (`draft` ⇄ `complete`) no tienen evento propio.

```json
{
  "subject": "sub-sara",
  "givenName": "Sara",
  "familyName": "Gil",
  "displayName": "Sara Gil",
  "contactPhone": "+34600111222",
  "status": "complete",
  "version": 9
}
```

### AddressAdded

La persona añadió una dirección a su perfil.

**Emitido por**: `addMyAddress`.

Si nace marcada por defecto, `isDefault` viene en verdadero y **no** se publica además `DefaultAddressChanged`: serían dos eventos con la misma `version` y la regla «descarta el más viejo» dejaría de desempatarlos. Que la anterior por defecto de ese `type` deja de serlo lo derivas tú de la invariante publicada: **como mucho una por defecto por tipo**.

```json
{
  "subject": "sub-tomas",
  "addressId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "label": "Casa",
  "type": "shipping",
  "postal": {
    "line1": "Calle 10 # 43-25", "line2": "Apto 501", "line3": null,
    "postalCode": null, "countryCode": "CO",
    "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
    "localityCode": "05001", "localityName": "Medellín"
  },
  "isDefault": true,
  "status": "complete",
  "version": 12
}
```

### AddressChanged

La persona reemplazó el contenido de una de sus direcciones. Reemplazo completo: el payload lleva la dirección entera tal como quedó, no el delta.

**Emitido por**: `updateMyAddress`.

```json
{
  "subject": "sub-tomas",
  "addressId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "label": "Casa",
  "type": "billing",
  "postal": { "…": "…" },
  "isDefault": true,
  "status": "draft",
  "version": 13
}
```

### AddressRemoved

La persona eliminó una de sus direcciones.

**Emitido por**: `removeMyAddress`.

`wasDefault` dice si la eliminada era la de por defecto de su `type`. En ese caso el perfil queda **sin ninguna** para ese tipo: no se promociona otra automáticamente, porque elegir cuál es decisión del usuario y no del servidor. No inventes una sustituta en tu réplica.

```json
{
  "subject": "sub-tomas",
  "addressId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "type": "shipping",
  "wasDefault": true,
  "status": "draft",
  "version": 14
}
```

### DefaultAddressChanged

La persona eligió otra dirección por defecto para un `type`.

**Emitido por**: `setMyDefaultAddress`. Y **solo** por él: ver `AddressAdded`.

`previousAddressId` viene nulo si no había ninguna marcada antes para ese tipo.

```json
{
  "subject": "sub-tomas",
  "type": "shipping",
  "addressId": "7a1b2c3d-4e5f-6071-8293-a4b5c6d7e8f9",
  "previousAddressId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "status": "complete",
  "version": 15
}
```

### ProfileDeactivated

Un administrador desactivó el perfil. Conserva todos sus datos y todas sus direcciones: desactivar no es borrar, y no toca la identidad en el servidor de identidad.

**Emitido por**: `deactivateProfile`.

```json
{ "subject": "sub-luis", "status": "deactivated", "version": 4 }
```

### ProfileReactivated

Un administrador reactivó el perfil, que vuelve a `draft` o a `complete` según los datos que tenga en ese momento.

**Emitido por**: `reactivateProfile`.

Sin este evento dejarías al usuario desactivado **para siempre** en tu réplica. El `status` del payload es el recalculado, no un estado anterior restaurado: aquí no se guarda ninguno.

```json
{ "subject": "sub-luis", "status": "complete", "version": 5 }
```

### ProfileDeleted

Se borró el perfil y su información personal quedó purgada.

**Emitido por**: `deleteMyProfile` y `deleteProfile`.

Es la señal con la que cada consumidor purga sus propias copias, así que su fiabilidad fija la de la capa entera. **No lleva ningún dato personal a propósito**: mandar el contacto de quien acaba de ejercer su derecho de supresión sería lo contrario de lo que el evento pide hacer.

Sí lleva `version`, y no por simetría: sin ella, un evento de contacto que llegue tarde se aplicaría después del borrado y **resucitaría el perfil en tu réplica**.

Qué hacer al recibirlo: si mantienes una **réplica de solo lectura**, borra tu copia. Si tienes un **snapshot inmutable** ya congelado en un pedido, no lo toques: no es una copia del perfil, es un dato del pedido.

```json
{
  "subject": "sub-quique",
  "version": 21,
  "deletedAt": "2026-03-14T09:21:07.482Z",
  "reason": "back-office"
}
```

`reason` es `self-service` cuando lo pidió la propia persona y `back-office` cuando lo ejecutó un operador.

### Suscripciones

**Ninguna, y es una decisión, no un olvido.** Este servicio **solo publica**: no consume ningún evento de nadie, así que no hay ninguna puerta de entrada por evento y **no puedes encargarle trabajo publicando un mensaje**. Todo lo que este servicio hace entra por su API HTTP.

Tres consecuencias para quien se integre:

- **No nos suscribimos a los eventos del servidor de identidad.** El aprovisionamiento just-in-time ya cubre todas las vías de alta sin enumerarlas, y un usuario deshabilitado allí deja de recibir tokens, así que no llega hasta aquí.
- **La baja la inicia un administrador contra nuestra API**, y por esa vía `ProfileDeleted` se emite y la cadena de purga funciona. Ese es el procedimiento correcto: se borra aquí primero, y deshabilitar la identidad en el servidor de identidad es un acto aparte, fuera de este contrato.
- **Consecuencia aceptada a sabiendas**: si alguien borra la identidad directamente en la consola del servidor de identidad, este servicio no se entera y quedan datos personales de alguien que ya no existe. Cerrar ese hueco es reconciliación periódica, que es despliegue y no diseño.
