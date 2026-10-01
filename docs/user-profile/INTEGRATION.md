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
  - name: resolveDeliveryProfileForServices
    method: GET
    path: /services/profiles/{subject}/delivery
    access: service profile-delivery:read
  - name: resolveContactsBatchForServices
    method: POST
    path: /services/profiles/contact/batch
    access: service profile-contact:read
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

`user-profile` (dominio `identity`) es la fuente de verdad del **perfil de negocio** de cada persona
autenticada: datos de contacto y direcciones postales. Lo indexa por el `sub` del token OIDC (el
`subject`), que es la única clave que comparte con el servidor de identidad. A otros servidores les
ofrece dos cosas:

- **Resolver un perfil por subject**, en una de dos proyecciones separadas por contenido y por scope.
  La de **contacto** no lleva dirección: nombre, email, teléfono y status. La de **entrega** añade la
  dirección shipping predeterminada.
  - Se pueden pedir de una en una (con caché de 60 s invalidada por cada cambio) o por lote de hasta
    100 subjects. Los lotes nunca fallan enteros y sirven también para arrancar o reconciliar una
    réplica.
  - El scope autoriza sobre **cualquier** subject.
- **Mantener una copia local** a partir de diez eventos. Cada uno lleva el estado que cambió y una
  `version`, el `lockVersion` del perfil, monotónica por subject.
  - El email de contacto es una réplica del claim del servidor de identidad. Este servicio no lo
    cambia nunca por su API.
  - Un perfil `deactivated` conserva sus datos y se sigue resolviendo, con su status.

Contratos formales: [`openapi.yaml`](openapi.yaml) (HTTP) y [`asyncapi.yaml`](asyncapi.yaml) (eventos).

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con un token de usuario. Para obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y el scope que tu cliente
   tiene concedido. Fija la audiencia `aud: user-profile`, porque se valida: un token emitido para
   otro servicio responde `403 ACCESS_DENIED`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=profile-delivery:read&audience=user-profile
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`. Sin
   token, o con uno caducado, la respuesta es `401 UNAUTHENTICATED`. Con un scope que no cubre el
   endpoint, `403 ACCESS_DENIED`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| order-service | profile-delivery:read | Resuelve a quién y dónde entregar un pedido. |
| notification-service | profile-contact:read | Resuelve cómo contactar a una persona. Sin acceso a direcciones, ni pidiéndolas. |

Un consumidor nuevo necesita su propio cliente máquina con el scope mínimo, y eso lo da de alta el
dueño del servicio.

**Convenciones comunes a los cuatro endpoints:**

- **Subject en la ruta:** viaja codificado como segmento de URL (`%2F` para `/`).
- **Campos sin valor:** todo campo declarado viaja; si no tiene valor, como `null`.
- **Instantes:** van en UTC ISO-8601 con milisegundos y `Z`.
- **`status`:** es `draft`, `complete` o `deactivated`.
- **`version`:** es el `lockVersion` del perfil, el mismo número que llevan los eventos.
- **`updatedAt`:** es el `updatedAt` del perfil.
- **Errores:** el cuerpo es `{timestamp, status, error, code, message, details, correlationId}`. Solo
  `code` y status son contrato.

`PostalAddress`, la forma de la dirección en la proyección de entrega y en los eventos:

| Campo | Tipo | Notas |
|---|---|---|
| line1 | string (1–140) | requerido |
| line2 | string (1–140) \| null | |
| line3 | string (1–140) \| null | |
| postalCode | string (1–16) \| null | sin patrón: depende del país |
| countryCode | string | requerido; ISO 3166-1 alpha-2 |
| adminAreaCode | string \| null | ISO 3166-2 (`CO-ANT`); su prefijo es `countryCode` |
| adminAreaName | string (1–120) \| null | presente si hay `adminAreaCode`; copiado al guardar |
| localityCode | string (1–32) \| null | opaco, se interpreta según el país |
| localityName | string (1–120) | requerido; copiado al guardar |

Los códigos territoriales solo se validan en forma, nunca en existencia. Los nombres se copiaron al
guardar y no se vuelven a resolver.

### resolveContactForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/profiles/{subject}/contact` |
| Acceso | `service` — scope `profile-contact:read` |
| Idempotencia | no aplica (query) |
| Caché | 60 s por `subject`, invalidada por cualquiera de los diez eventos; el TTL es solo el respaldo |

**Request** — path `subject: string` (1–255, requerido). Sin cuerpo.

**Response** — `200`

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| displayName | string \| null | |
| givenName | string \| null | |
| familyName | string \| null | |
| contactEmail | string \| null | réplica del claim email del servidor de identidad |
| contactPhone | string \| null | E.164 |
| status | string | `draft` \| `complete` \| `deactivated` |
| version | int | lockVersion del perfil |
| updatedAt | timestamp | |

```json
{
  "subject": "sub-ana-001",
  "displayName": "Ana Pérez",
  "givenName": "Ana",
  "familyName": "Pérez",
  "contactEmail": "ana@example.com",
  "contactPhone": "+573001234567",
  "status": "complete",
  "version": 2,
  "updatedAt": "2026-10-01T09:21:07.482Z"
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PROFILE_NOT_FOUND | 404 | No existe perfil ni lápida para ese subject: la persona nunca accedió. | No reintentar; el perfil se crea cuando la persona accede por primera vez. |
| PROFILE_DELETED | 410 | El subject tiene lápida: su perfil se borró. | No reintentar; borra lo que tengas de ese subject. |

### resolveDeliveryProfileForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/profiles/{subject}/delivery` |
| Acceso | `service` — scope `profile-delivery:read` |
| Idempotencia | no aplica (query) |
| Caché | 60 s por `subject`, invalidada por cualquiera de los diez eventos; el TTL es solo el respaldo |

**Request** — path `subject: string` (1–255, requerido). Sin cuerpo.

**Response** — `200`. Son los campos de contacto con `displayName`, `contactEmail` y `contactPhone`
obligatorios, más `shippingAddress`.

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| displayName | string | requerido |
| givenName | string \| null | |
| familyName | string \| null | |
| contactEmail | string | requerido |
| contactPhone | string | requerido; E.164 |
| status | string | `draft` \| `complete` \| `deactivated` |
| version | int | lockVersion del perfil |
| updatedAt | timestamp | |
| shippingAddress | PostalAddress | requerido; la dirección shipping predeterminada, sin id, label ni isDefault |

```json
{
  "subject": "sub-ana-001",
  "displayName": "Ana Pérez",
  "givenName": "Ana",
  "familyName": "Pérez",
  "contactEmail": "ana@example.com",
  "contactPhone": "+573001234567",
  "status": "complete",
  "version": 2,
  "updatedAt": "2026-10-01T09:21:07.482Z",
  "shippingAddress": {
    "line1": "Calle 10 # 43-12", "line2": "Apto 301", "line3": null, "postalCode": "050021",
    "countryCode": "CO", "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
    "localityCode": "05001", "localityName": "Medellín"
  }
}
```

Un perfil es **entregable** si tiene `contactEmail`, `contactPhone`, `displayName` y una dirección
shipping predeterminada. Se evalúa sobre los datos, no sobre el status: un perfil `deactivated` con esos
datos se resuelve, con `status: "deactivated"`, y tu servicio decide qué hacer con él.

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PROFILE_NOT_FOUND | 404 | No existe perfil ni lápida para ese subject. | No reintentar. |
| PROFILE_DELETED | 410 | El subject tiene lápida. | No reintentar; borra lo que tengas de ese subject. |
| PROFILE_INCOMPLETE | 409 | El perfil no es entregable. `details` enumera lo que falta, con estos nombres y en este orden: `contactEmail`, `contactPhone`, `displayName`, `defaultShippingAddress`. | No reintentar ahora; se resolverá cuando la persona complete su perfil (lo verás en el evento con `status: "complete"`). |

### resolveContactsBatchForServices

| | |
|---|---|
| Endpoint | `POST /api/v1/services/profiles/contact/batch` |
| Acceso | `service` — scope `profile-contact:read` |
| Idempotencia | no aplica (lectura; POST solo porque los subjects no caben con garantías en una query) |
| Caché | sin caché: refleja el estado real |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| subjects | string[] | requerido, entre 1 y 100 elementos, contados tal como llegan (antes de deduplicar); cada uno de 1 a 255 caracteres |

```json
{ "subjects": ["sub-beto-002", "sub-nadie-999", "sub-ana-001", "sub-beto-002", "sub-cleo-003"] }
```

**Response** — `200`

| Campo | Tipo | Notas |
|---|---|---|
| profiles | ContactDetails[] | la proyección de contacto de `resolveContactForServices`, una por subject resuelto |
| unresolved | { subject, reason }[] | `reason`: `not-found` (ni perfil ni lápida) \| `deleted` (lápida) |

- **Nunca falla entero por un subject:** cada subject sale una sola vez, en `profiles` o en
  `unresolved`.
- **Repetidos:** los subjects repetidos se deduplican.
- **Orden:** las dos listas conservan el orden de la **primera aparición** de cada subject en la
  petición.

```json
{
  "profiles": [
    { "subject": "sub-beto-002", "displayName": "Beto", "givenName": null, "familyName": null,
      "contactEmail": "beto@example.com", "contactPhone": null, "status": "deactivated", "version": 1,
      "updatedAt": "2026-10-01T09:22:10.120Z" },
    { "subject": "sub-ana-001", "displayName": "Ana Pérez", "givenName": "Ana", "familyName": "Pérez",
      "contactEmail": "ana@example.com", "contactPhone": "+573001234567", "status": "complete", "version": 2,
      "updatedAt": "2026-10-01T09:21:07.482Z" }
  ],
  "unresolved": [
    { "subject": "sub-nadie-999", "reason": "not-found" },
    { "subject": "sub-cleo-003", "reason": "deleted" }
  ]
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| TOO_MANY_SUBJECTS | 422 | La petición trae más de 100 subjects (antes de deduplicar). | Corregir input: parte el lote en trozos de 100 como mucho. |
| VALIDATION_ERROR | 400 | Lote vacío, sin `subjects`, o algún subject vacío o de más de 255 caracteres: se rechaza el lote entero. | Corregir input. |

### resolveDeliveryProfilesBatchForServices

| | |
|---|---|
| Endpoint | `POST /api/v1/services/profiles/delivery/batch` |
| Acceso | `service` — scope `profile-delivery:read` |
| Idempotencia | no aplica (lectura; POST solo porque los subjects no caben con garantías en una query) |
| Caché | sin caché: refleja el estado real |

**Request** — igual que el lote de contacto: `subjects: string[]`, de 1 a 100.

```json
{ "subjects": ["sub-cleo-003", "sub-beto-002", "sub-ana-001", "sub-nadie-999"] }
```

**Response** — `200`

| Campo | Tipo | Notas |
|---|---|---|
| profiles | DeliveryProfile[] | la proyección de entrega de `resolveDeliveryProfileForServices`, una por subject entregable |
| unresolved | { subject, reason }[] | `reason`: `not-found` \| `deleted` \| `incomplete` (perfil no entregable) |

Rige lo mismo que en el lote de contacto: nunca falla entero, deduplica y conserva el orden de la
primera aparición. Un perfil `deactivated` entregable va a `profiles`, con su status.

```json
{
  "profiles": [
    { "subject": "sub-ana-001", "displayName": "Ana Pérez", "givenName": "Ana", "familyName": "Pérez",
      "contactEmail": "ana@example.com", "contactPhone": "+573001234567", "status": "complete", "version": 2,
      "updatedAt": "2026-10-01T09:21:07.482Z",
      "shippingAddress": { "line1": "Calle 10 # 43-12", "line2": "Apto 301", "line3": null, "postalCode": "050021",
        "countryCode": "CO", "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
        "localityCode": "05001", "localityName": "Medellín" } }
  ],
  "unresolved": [
    { "subject": "sub-cleo-003", "reason": "deleted" },
    { "subject": "sub-beto-002", "reason": "incomplete" },
    { "subject": "sub-nadie-999", "reason": "not-found" }
  ]
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| TOO_MANY_SUBJECTS | 422 | La petición trae más de 100 subjects (antes de deduplicar). | Corregir input: parte el lote en trozos de 100 como mucho. |
| VALIDATION_ERROR | 400 | Lote vacío, sin `subjects`, o algún subject de forma inválida. | Corregir input. |

## Eventos

Los publica `user-profile` en el canal lógico `profileEvents` (el topic o cola físico es parámetro de
despliegue). No se suscribe a nada. Hoy no hay consumidores declarados: un servidor que quiera mantener
una réplica del perfil se suscribe a este canal.

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProfileContactDetailsChanged",
    "eventVersion": 1,
    "occurredAt": "2026-10-01T09:21:07.482Z",
    "source": "user-profile",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": {
    "subject": "sub-ana-001", "version": 1, "status": "draft", "givenName": "Ana", "familyName": "Pérez",
    "displayName": "Ana Pérez", "contactPhone": "+573001234567"
  }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`ProfileContactDetailsChanged`). Discriminador: el canal transporta los diez tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| metadata.source | string | Servicio emisor: `user-profile`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| metadata.traceparent | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Reglas comunes a los diez eventos:**

- **Garantía de entrega:** outbox. Ningún evento se pierde si la transacción confirma, y cada
  transacción publica como mucho un evento.
- **Orden y versión:** todos llevan `subject` y `version`, el `lockVersion` del perfil tras el cambio.
  Para una réplica, aplica un evento solo si su `version` es mayor que la última que guardaste para ese
  subject, y descarta los demás (duplicados o desordenados). Una mutación sin cambios no publica nada.
- **Estado, no solo el aviso:** cada evento lleva los campos que cambió y el `status` recalculado (salvo
  `ProfileDeleted`). Los cambios de completitud no tienen evento propio: se ven en el `status` del
  evento que los causó.
- **Encarnación:** `ProfileDeleted` **cierra la encarnación** del subject. Borra todo lo que tengas de
  él, **incluida la última `version` vista**. Si después se levanta la lápida, el perfil nuevo es otra
  encarnación y su `ProfileProvisioned` empieza otra vez en `version: 0`.
- **PII:** los eventos llevan nombres, email, teléfono y direcciones. Trata el canal como dato personal.
  `ProfileDeleted` no lleva PII.

### ProfileProvisioned

Se creó el perfil de un subject en su primera petición autenticada. Siempre nace en `draft` con
`version: 0`. Emitido por `provisionProfileFromIdentity`, el aprovisionamiento just-in-time interno.

```json
{ "subject": "sub-ana-001", "version": 0, "status": "draft", "contactEmail": "ana@example.com",
  "givenName": "Ana", "familyName": "Pérez", "displayName": "Ana Pérez" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido; 0 |
| status | string | requerido |
| contactEmail | string \| null | del claim del token |
| givenName | string \| null | del claim del token |
| familyName | string \| null | del claim del token |
| displayName | string \| null | del claim del token |

### ProfileContactEmailRefreshed

El aprovisionamiento refrescó `contactEmail` porque el claim `email` del token cambió. Puede ocurrir
también en un perfil `deactivated`. Emitido por `provisionProfileFromIdentity`.

```json
{ "subject": "sub-ana-001", "version": 1, "status": "draft", "contactEmail": "ana.perez@example.org" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |
| contactEmail | string | requerido |

### ProfileContactDetailsChanged

El titular reemplazó su bloque de contacto. Un campo a `null` significa que se borró. Emitido por
`updateMyContactDetails`.

```json
{ "subject": "sub-ana-001", "version": 1, "status": "draft", "givenName": "Ana", "familyName": "Pérez",
  "displayName": "Ana Pérez", "contactPhone": "+573001234567" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |
| givenName | string \| null | |
| familyName | string \| null | |
| displayName | string \| null | |
| contactPhone | string \| null | E.164 |

### AddressAdded

El titular añadió una dirección. Si llegó como predeterminada, `previousDefaultAddressId` dice cuál dejó
de serlo, y no se publica además `DefaultAddressChanged`. Emitido por `addMyAddress`.

```json
{ "subject": "sub-ana-001", "version": 3, "status": "complete", "addressId": "6b1f0c2e-6a35-4d6e-9d55-3f1a0e8b7c21",
  "label": "Casa", "type": "shipping", "isDefault": true,
  "postal": { "line1": "Calle 10 # 43-12", "line2": "Apto 301", "line3": null, "postalCode": "050021",
    "countryCode": "CO", "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
    "localityCode": "05001", "localityName": "Medellín" },
  "previousDefaultAddressId": null }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |
| addressId | uuid | requerido |
| label | string \| null | |
| type | string | requerido; `shipping` \| `billing` |
| isDefault | boolean | requerido |
| postal | PostalAddress | requerido |
| previousDefaultAddressId | uuid \| null | la predeterminada de ese tipo que quedó desmarcada |

### AddressChanged

El titular reemplazó una dirección. Si cambió de tipo, llega con `isDefault: false`. Emitido por
`updateMyAddress`.

```json
{ "subject": "sub-ana-001", "version": 4, "status": "complete", "addressId": "6b1f0c2e-6a35-4d6e-9d55-3f1a0e8b7c21",
  "label": "Casa principal", "type": "shipping", "isDefault": true,
  "postal": { "line1": "Calle 10 # 43-12", "line2": "Apto 302", "line3": null, "postalCode": "050021",
    "countryCode": "CO", "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
    "localityCode": "05001", "localityName": "Medellín" } }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |
| addressId | uuid | requerido |
| label | string \| null | |
| type | string | requerido |
| isDefault | boolean | requerido |
| postal | PostalAddress | requerido |

### AddressRemoved

El titular quitó una dirección. Si era la predeterminada, su tipo se queda sin predeterminada: no se
promociona otra. Emitido por `removeMyAddress`.

```json
{ "subject": "sub-ana-001", "version": 5, "status": "draft", "addressId": "6b1f0c2e-6a35-4d6e-9d55-3f1a0e8b7c21",
  "type": "shipping", "wasDefault": true }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |
| addressId | uuid | requerido |
| type | string | requerido |
| wasDefault | boolean | requerido |

### DefaultAddressChanged

El titular marcó otra dirección como predeterminada de su tipo. Emitido por `setMyDefaultAddress`.

```json
{ "subject": "sub-ana-001", "version": 6, "status": "complete", "addressId": "0e9d4a71-2b8c-4f3e-a1d6-5c7b9e2f8a40",
  "type": "shipping", "previousAddressId": "6b1f0c2e-6a35-4d6e-9d55-3f1a0e8b7c21" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |
| addressId | uuid | requerido |
| type | string | requerido |
| previousAddressId | uuid \| null | la que dejó de ser predeterminada |

### ProfileDeactivated

El back-office desactivó el perfil. Conserva sus datos y no admite mutaciones de su titular. Emitido por
`deactivateProfile`.

```json
{ "subject": "sub-ana-001", "version": 7, "status": "deactivated", "deactivatedBy": "sub-op-support",
  "deactivatedAt": "2026-10-01T10:02:44.913Z" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido; `deactivated` |
| deactivatedBy | string | requerido; subject del operador |
| deactivatedAt | timestamp | requerido |

### ProfileReactivated

El back-office reactivó el perfil. `status` es el recalculado (`draft` o `complete`). Emitido por
`reactivateProfile`.

```json
{ "subject": "sub-ana-001", "version": 8, "status": "complete" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido |
| status | string | requerido |

### ProfileDeleted

El perfil se borró, por su titular o desde back-office, y quedó su lápida. No lleva PII. **Borra todo lo
que tengas de ese subject**, incluida la última `version` vista. Emitido por `deleteMyProfile` y
`deleteProfile`.

```json
{ "subject": "sub-ana-001", "version": 9, "deletedAt": "2026-10-01T10:15:03.207Z", "reason": "self-service" }
```

| Campo | Tipo | Notas |
|---|---|---|
| subject | string | requerido |
| version | int | requerido; el último `lockVersion` más uno |
| deletedAt | timestamp | requerido; el mismo instante que la lápida |
| reason | string | requerido; `self-service` \| `back-office` |

### Suscripciones

`user-profile` no se suscribe a ningún evento. No tiene puertas de entrada por evento, y en particular
no consume los eventos del servidor de identidad: el email se refresca desde el token en la siguiente
petición del titular.
