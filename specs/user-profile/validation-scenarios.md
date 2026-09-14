# user-profile — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/user-profile v0.1.1. Contrato de validación para la fase de generación.

## Convenciones de determinación

Valen para **todo** el servicio y ningún escenario las repite.

- **Instantes**: UTC, ISO-8601 con milisegundos y sufijo `Z` (`createdAt`, `updatedAt`, `deletedAt`, y el `occurredAt` de la envoltura). No hay ninguna fecha de negocio local, así que el servicio no declara zona horaria de negocio. Se verifican **por forma o por rango**, nunca por valor exacto.
- **Ausencia vs nulo**: todo campo declarado en un output o en un payload de evento **está presente**; el que no tiene valor viaja como `null`, nunca omitido. Un `Then` que dice «`contactPhone` es nulo» exige la clave presente con valor nulo.
- **Identificadores**: `id` y `addressId` se verifican por forma (identificador) y por reutilización simbólica dentro del flujo, jamás por valor literal. El `subject` sí se verifica por valor: lo fija el token del `Given`.
- **Mayúsculas y acentos**: la unicidad de `subject` es **exacta y sensible** (es un identificador opaco, no un nombre). El filtro `search` de `listProfiles` es **insensible** a mayúsculas y acentos y coincide por **prefijo**.
- **Forma del cuerpo de error**: la impone el generador (en keel-spring, `{timestamp, status, error, code, message, details}` más `correlationId`). Los escenarios solo fijan el **`code`** y el **status**; el **texto del mensaje no es contrato** y ningún `Then` lo afirma.
- **Sobre de paginación**: el canónico del DSL — `{ items, page, size, totalElements, totalPages }`, con los query params `page` (base 0) y `size`.
- **Cabecera de idempotencia**: `Idempotency-Key`, solo en `addMyAddress`. Se declara pero no se exige: sin ella la operación se ejecuta sin deduplicar.
- **Cabecera `Location`**: **ningún** endpoint de este servicio devuelve `201`, así que no hay ninguna `Location` que afirmar. Es consecuencia de que las cinco mutaciones de `/me` devuelvan `200` con el perfil.
- **Concurrencia**: `optimisticLocking: declared` sobre `UserProfile`, así que dos escrituras concurrentes sobre la misma raíz producen **conflicto** `409 CONCURRENT_MODIFICATION`, nunca un último-gana silencioso.
- **Identidades**: los tokens de usuario se emiten con el `sub` que cada `Given` nombre; los roles son los declarados (`profile-support`, `profile-admin`); las credenciales de máquina son las de los `serviceClients` declarados (`order-service`, `notification-service`). Ninguna otra identidad aparece en este documento.
- **El subject nunca viaja en el cuerpo ni en la ruta de `/me`**: lo estampa el servidor desde el claim `sub`. Los `When` de esa superficie no lo incluyen.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|---|---|---|
| `provisionProfileFromIdentity` | FL-PRV-001, FL-PRV-002, FL-PRV-003, FL-PRV-004, FL-DEL-003, FL-DEL-004 | interna (se ejecuta al validar el token de cada petición) |
| `getMyProfile` | FL-PRV-001, FL-ME-001, FL-DEL-001, FL-SEC-001 | usuarios |
| `updateMyContactDetails` | FL-ME-001, FL-ME-002, FL-ME-003 | usuarios |
| `addMyAddress` | FL-ADR-001, FL-ADR-003, FL-ADR-004, FL-ADR-005, FL-TER-001, FL-TER-002, FL-ME-003 | usuarios |
| `updateMyAddress` | FL-ADR-002, FL-ADR-005, FL-ME-003 | usuarios |
| `removeMyAddress` | FL-ADR-002, FL-ADR-005, FL-ME-003 | usuarios |
| `setMyDefaultAddress` | FL-ADR-002, FL-ADR-005, FL-ME-003 | usuarios |
| `deleteMyProfile` | FL-DEL-001, FL-DEL-005 | usuarios |
| `listProfiles` | FL-BO-001, FL-SEC-001 | back-office |
| `getProfile` | FL-BO-002, FL-DEL-002 | back-office |
| `deactivateProfile` | FL-BO-003, FL-ME-003 | back-office |
| `reactivateProfile` | FL-BO-004 | back-office |
| `deleteProfile` | FL-DEL-002, FL-DEL-005, FL-SEC-001 | back-office |
| `reinstateSubject` | FL-DEL-004 | back-office |
| `resolveContactForServices` | FL-M2M-001, FL-M2M-004, FL-CCH-001 | **servidores (M2M)** |
| `resolveContactsBatchForServices` | FL-M2M-003, FL-M2M-004 | **servidores (M2M)** |
| `resolveDeliveryProfileForServices` | FL-M2M-002, FL-M2M-004, FL-CCH-002 | **servidores (M2M)** |
| `resolveDeliveryProfilesBatchForServices` | FL-M2M-003, FL-M2M-004 | **servidores (M2M)** |

**Cobertura de errores** — los 16 códigos del diseño:

| `code` | Status | Flujo que lo ejercita |
|---|---|---|
| `SUBJECT_NOT_PROVISIONABLE` | 403 | FL-PRV-002 |
| `PROFILE_DELETED` | 410 | FL-DEL-001, FL-DEL-003 |
| `PROFILE_DEACTIVATED` | 409 | FL-ME-003 |
| `PROFILE_NOT_FOUND` | 404 | FL-BO-002, FL-DEL-005, FL-M2M-001 |
| `ADDRESS_NOT_FOUND` | 404 | FL-ADR-005 |
| `ADDRESS_LIMIT_REACHED` | 409 | FL-ADR-005 |
| `SUBDIVISION_CODE_COUNTRY_MISMATCH` | 422 | FL-TER-002 |
| `TERRITORY_NAME_MISSING` | 422 | FL-TER-002 |
| `PROFILE_ALREADY_DEACTIVATED` | 409 | FL-BO-003 |
| `PROFILE_NOT_DEACTIVATED` | 409 | FL-BO-004 |
| `SUBJECT_NOT_DELETED` | 404 | FL-DEL-004 |
| `TOO_MANY_SUBJECTS` | 422 | FL-M2M-003 |
| `PROFILE_INCOMPLETE` | 409 | FL-M2M-002 |
| `CONCURRENT_MODIFICATION` | 409 | FL-ME-002 |
| `IDEMPOTENCY_KEY_IN_PROGRESS` | 409 | FL-ADR-004 |
| `IDEMPOTENCY_KEY_REUSED` | 409 | FL-ADR-003 |

**Cobertura de eventos** — los 10 de `messaging`:

| Evento | Flujo que afirma su publicación |
|---|---|
| `ProfileProvisioned` | FL-PRV-001, FL-PRV-004, FL-DEL-004 |
| `ProfileContactEmailRefreshed` | FL-PRV-003 |
| `ProfileContactDetailsChanged` | FL-ME-001 |
| `AddressAdded` | FL-ADR-001, FL-TER-001 |
| `AddressChanged` | FL-ADR-002 |
| `AddressRemoved` | FL-ADR-002 |
| `DefaultAddressChanged` | FL-ADR-002 |
| `ProfileDeactivated` | FL-BO-003 |
| `ProfileReactivated` | FL-BO-004 |
| `ProfileDeleted` | FL-DEL-001, FL-DEL-002 |

**Cobertura del lifecycle** — las 6 aristas de `UserProfile`:

| Arista | Operación | Flujo |
|---|---|---|
| `draft → complete` | `addMyAddress` | FL-ADR-001 |
| `complete → draft` | `removeMyAddress` | FL-ADR-002 |
| `complete → draft` | `updateMyAddress` (cambio de `type`) | FL-ADR-002 |
| `draft → deactivated` | `deactivateProfile` | FL-BO-003 |
| `complete → deactivated` | `deactivateProfile` | FL-ME-003 |
| `deactivated → draft` | `reactivateProfile` | FL-BO-004 |
| `deactivated → complete` | `reactivateProfile` | FL-BO-004 |
| transición inválida | `reactivateProfile` desde `complete` | FL-BO-004 |

Los tres estados (`draft`, `complete`, `deactivated`) los alcanza algún flujo.

**Proyección de `UserProfile`** — el cuerpo que devuelven `getMyProfile`, `updateMyContactDetails`, `addMyAddress`, `updateMyAddress`, `removeMyAddress`, `setMyDefaultAddress`, `getProfile`, `deactivateProfile` y `reactivateProfile`, derivada del artefacto y **idéntica en todos los `Then`** de este documento:

```
id, subject, contactEmail, givenName, familyName, displayName, contactPhone,
status, lockVersion, createdAt, updatedAt,
addresses: [ { id, label, type, isDefault, createdAt, updatedAt,
               postal: { line1, line2, line3, postalCode, countryCode,
                         adminAreaCode, adminAreaName, localityCode, localityName } } ]
```

`listProfiles` devuelve esa misma proyección **sin** `addresses` (`exclude`).

---

## Aprovisionamiento just-in-time

### FL-PRV-001: un token válido sin perfil crea el perfil en draft en cualquier endpoint

**Given**: no existe perfil ni lápida para el subject `sub-alice`. El canal `profileEvents` está purgado.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con token de usuario de `sub-alice`, cuyos claims son `email: "alice@example.com"`, `given_name: "Alice"`, `family_name: "Ruiz"`, `name: "Alice Ruiz"`.

**Then**:
1. Status `200`.
2. El cuerpo trae la proyección completa de `UserProfile` y ningún campo adicional.
3. `subject: "sub-alice"`, `contactEmail: "alice@example.com"`, `givenName: "Alice"`, `familyName: "Ruiz"`, `displayName: "Alice Ruiz"`.
4. `contactPhone` es **nulo** y `addresses` es una lista **vacía**.
5. `status: "draft"` — nacer incompleto es un estado válido, no un error.
6. `id` y `createdAt`/`updatedAt` tienen forma válida; `lockVersion` es un entero.
7. Se publica **exactamente un** `ProfileProvisioned` en el canal `profileEvents`, con `subject: "sub-alice"`, `contactEmail: "alice@example.com"`, `givenName: "Alice"`, `familyName: "Ruiz"`, `displayName: "Alice Ruiz"`, `contactPhone: null`, `status: "draft"` y el `version` del perfil devuelto.
8. `listProfiles` con rol `profile-support` devuelve ese perfil: existe de verdad, no solo en la respuesta.

**When** (segundo escenario del flujo): la misma petición `GET /api/v1/me/profile` con el mismo token.

**Then**:
9. Status `200` con el **mismo** `id` y el **mismo** `subject`.
10. El canal `profileEvents` **no** ha recibido un segundo `ProfileProvisioned`.

**Notas de determinación**: es el escenario que hace verificable el patrón entero. La petición es un `GET` y **escribe**: la aserción 8 lo comprueba desde fuera.

### FL-PRV-002: un token de máquina no aprovisiona nada

**Given**: no existe perfil ni lápida para el subject del cliente máquina `order-service`. El canal `profileEvents` está purgado.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con la credencial de máquina del cliente `order-service`.

**Then**:
1. Status `403` con `code: SUBJECT_NOT_PROVISIONABLE`.
2. `listProfiles` con rol `profile-admin` devuelve `totalElements: 0`: no se creó ningún perfil fantasma.
3. El canal `profileEvents` no ha recibido ningún mensaje.

**Notas de determinación**: sin esta guarda se crearía un perfil por cada servicio que nos llame. El `Then` 2 es el que lo mide; el status por sí solo no distinguiría un rechazo de autorización de esta guarda.

### FL-PRV-003: el refresco del contactEmail publica su propio evento, también sobre un perfil desactivado

**Given**: `sub-bob` tiene perfil aprovisionado con `contactEmail: "bob@example.com"`. El canal está purgado.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con token de `sub-bob` cuyo claim `email` es ahora `"bob.nuevo@example.com"`.

**Then**:
1. Status `200` con `contactEmail: "bob.nuevo@example.com"`.
2. Se publica **exactamente un** `ProfileContactEmailRefreshed` con `subject: "sub-bob"`, `contactEmail: "bob.nuevo@example.com"` y el `version` nuevo.
3. No se publica `ProfileProvisioned` ni `ProfileContactDetailsChanged`.

**When**: la misma petición otra vez, con el mismo claim.

**Then**:
4. Status `200` y el canal **no** recibe un segundo `ProfileContactEmailRefreshed`: solo se publica cuando el claim difiere de lo almacenado.

**When**: `deactivateProfile` sobre `sub-bob` con rol `profile-admin`, y después `GET /api/v1/me/profile` con token de `sub-bob` cuyo claim `email` es `"bob.tercero@example.com"`.

**Then**:
5. Status `200` con `contactEmail: "bob.tercero@example.com"` y `status: "deactivated"`.
6. Se publica un `ProfileContactEmailRefreshed`.

**Notas de determinación**: el refresco ocurre en cualquier estado porque no es una edición del usuario sino el mantenimiento de una réplica cuyo dueño es el servidor de identidad. Es también el fallo más silencioso del diseño si el evento faltara: no hay error ni traza, solo un correo que no llega.

### FL-PRV-004: tres peticiones simultáneas del mismo subject producen un solo perfil

**Given**: no existe perfil ni lápida para `sub-carol`. El canal está purgado.

**When**: tres `GET /api/v1/me/profile` **a la vez**, las tres con el token de `sub-carol`. Las tres disparan `provisionProfileFromIdentity` con la **misma clave de idempotencia** — el `callerSubject`, que es `payload-field` sobre la clave natural del agregado.

**Then**:
1. Las tres responden `200` con el **mismo** `id` y el mismo `subject`.
2. `listProfiles` con rol `profile-admin` devuelve **exactamente un** perfil con `subject: "sub-carol"` — la aserción que no depende de quién gane la carrera.
3. El canal recibe **exactamente un** `ProfileProvisioned`.

**Notas de determinación**: es la carrera de `idempotency: { keySource: payload-field, keyField: callerSubject }`. La clave es la clave natural del agregado, así que la guarda es la restricción de unicidad y no hay registro de claves: el desenlace no admite `IDEMPOTENCY_KEY_IN_PROGRESS`, las tres reciben el perfil. La aserción 2 es la única que puede ponerse roja si la colisión se propagara en vez de releerse.

---

## El perfil propio

### FL-ME-001: completar el contacto publica su evento y el perfil sigue en draft sin dirección

**Given**: `sub-dana` recién aprovisionada (`status: draft`, sin teléfono, sin direcciones). Canal purgado.

**When**: `updateMyContactDetails` — `PUT /api/v1/me/profile/contact-details`
```json
{ "givenName": "Dana", "familyName": "López", "displayName": "Dana L.", "contactPhone": "+573001112233" }
```

**Then**:
1. Status `200` con la proyección completa de `UserProfile` y ningún campo adicional.
2. `givenName: "Dana"`, `familyName: "López"`, `displayName: "Dana L."`, `contactPhone: "+573001112233"`.
3. `contactEmail` **no** cambió: sigue siendo el del claim del token. El cuerpo no acepta ese campo.
4. `status: "draft"` — hay contacto completo pero no hay dirección de envío por defecto, y la regla exige las dos cosas.
5. `lockVersion` es mayor que el que tenía antes de la llamada.
6. Se publica **exactamente un** `ProfileContactDetailsChanged` con `subject: "sub-dana"`, los cuatro campos del cuerpo, `status: "draft"` y el `version` nuevo.
7. `getMyProfile` devuelve el mismo cuerpo.

**Orden de evaluación**:
1. El subject no tiene lápida → `PROFILE_DELETED` (`410`).
2. El perfil no está `deactivated` → `PROFILE_DEACTIVATED` (`409`).
3. La versión leída sigue vigente → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**:
- `contactPhone: "600111222"` (sin prefijo internacional) → `400`: el value type `PhoneNumber` exige E.164.
- `displayName` de 201 caracteres → `400` (`maxLength: 200`).

### FL-ME-002: el reemplazo es completo y dos escrituras concurrentes dan conflicto

**Given**: `sub-dana` con el contacto completo de FL-ME-001 dentro de este mismo flujo (el primer escenario lo crea).

**When**: `updateMyContactDetails` — `PUT /api/v1/me/profile/contact-details`
```json
{ "givenName": "Dana", "displayName": "Dana L." }
```

**Then**:
1. Status `200`.
2. `familyName` es **nulo** y `contactPhone` es **nulo**: es un reemplazo completo del bloque, no un parche, y un campo que no venga se borra.
3. `contactEmail` sigue intacto.
4. `status: "draft"`.

**When**: la **misma** petición otra vez.

**Then**:
5. Status `200` con el mismo cuerpo salvo `lockVersion` y `updatedAt`: fijar el bloque a un valor es naturalmente idempotente, y por eso la operación no declara `idempotency`.

**When**: dos `PUT /api/v1/me/profile/contact-details` **a la vez** con cuerpos distintos (`displayName: "A"` y `displayName: "B"`).

**Then**:
6. Una responde `200` y la otra `409` con `code: CONCURRENT_MODIFICATION`.
7. `getMyProfile` devuelve `displayName` igual a `"A"` o a `"B"`, y el resto del cuerpo es coherente con la ganadora.
8. `lockVersion` avanzó **exactamente una** vez respecto al valor previo a la carrera — la aserción que no depende de quién gane.

### FL-ME-003: un perfil desactivado rechaza las cinco mutaciones de /me

**Given**: `sub-erik` con contacto completo y una dirección `shipping` por defecto (`status: complete`), desactivado con `deactivateProfile` desde el rol `profile-admin` dentro de este flujo.

**Then** (del `deactivateProfile` del `Given`):
1. `status: "deactivated"` — la arista `complete → deactivated`.

**When**: cada una de las cinco mutaciones de `/me`, con el token de `sub-erik`:
`PUT /me/profile/contact-details`, `POST /me/profile/addresses`, `PUT /me/profile/addresses/{addressId}`, `DELETE /me/profile/addresses/{addressId}`, `PUT /me/profile/addresses/{addressId}/default`.

**Then**:
2. Las cinco responden `409` con `code: PROFILE_DEACTIVATED`.
3. `getMyProfile` responde `200`: leer sí se puede, y el cuerpo es el de antes de los cinco intentos.
4. El canal no ha recibido ningún evento durante los cinco intentos.

**Casos borde**:
- `deleteMyProfile` sobre el mismo perfil desactivado responde su éxito normal, **no** `PROFILE_DEACTIVATED`: el derecho de supresión no depende de que alguien haya desactivado antes la cuenta. Se ejercita en FL-DEL-001.

---

## Direcciones

### FL-ADR-001: añadir la dirección de envío por defecto completa el perfil

**Given**: `sub-fran` con contacto completo (`contactEmail`, `contactPhone` y `displayName` con valor) y sin direcciones, `status: draft`. Canal purgado.

**When**: `addMyAddress` — `POST /api/v1/me/profile/addresses` con cabecera `Idempotency-Key: key-adr-1`
```json
{ "label": "Casa", "type": "shipping", "isDefault": true,
  "postal": { "line1": "Calle 10 # 43-25", "line2": "Apto 501", "line3": null,
              "postalCode": null, "countryCode": "CO",
              "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
              "localityCode": "05001", "localityName": "Medellín" } }
```

**Then**:
1. Status `200` — **no** `201`, y por tanto **no** hay cabecera `Location`.
2. El cuerpo trae la proyección completa de `UserProfile` y ningún campo adicional.
3. `addresses` tiene **un** elemento, con `label: "Casa"`, `type: "shipping"`, `isDefault: true` y el `postal` exacto del cuerpo, con `line3`, `postalCode` **nulos** y presentes.
4. `status: "complete"` — la arista `draft → complete`: hay contacto completo y dirección `shipping` por defecto.
5. Se publica **exactamente un** `AddressAdded` con `subject: "sub-fran"`, el `addressId` devuelto, `label`, `type: "shipping"`, el `postal` completo, `isDefault: true`, `status: "complete"` y el `version` nuevo.
6. **No** se publica `DefaultAddressChanged`: añadir marcando por defecto emite un solo evento, para que `version` siga siendo única por agregado.

**Orden de evaluación**:
1. La clave de idempotencia no está en curso → `IDEMPOTENCY_KEY_IN_PROGRESS` (`409`).
2. La clave no se reutiliza con otro contenido → `IDEMPOTENCY_KEY_REUSED` (`409`).
3. El subject no tiene lápida → `PROFILE_DELETED` (`410`).
4. El perfil no está `deactivated` → `PROFILE_DEACTIVATED` (`409`).
5. El perfil tiene menos de veinte direcciones → `ADDRESS_LIMIT_REACHED` (`409`).
6. El `adminAreaCode`, si viene, es un ISO 3166-2 del `countryCode` → `SUBDIVISION_CODE_COUNTRY_MISMATCH` (`422`).
7. Ningún código territorial viene sin su nombre → `TERRITORY_NAME_MISSING` (`422`).
8. La versión leída sigue vigente → `CONCURRENT_MODIFICATION` (`409`).

### FL-ADR-002: cambiar, reordenar y eliminar direcciones, con sus tres eventos

**Given**: `sub-fran` con el estado que deja FL-ADR-001 reproducido dentro de este flujo: contacto completo, una dirección `shipping` por defecto `a1`, `status: complete`. Canal purgado.

**When**: `addMyAddress` una segunda dirección `a2` (`label: "Oficina"`, `type: "shipping"`, `isDefault: false`).

**Then**:
1. Status `200`, `addresses` con dos elementos, `a1.isDefault: true` y `a2.isDefault: false`.
2. `status` sigue en `"complete"`.

**When**: `setMyDefaultAddress` — `PUT /api/v1/me/profile/addresses/{a2}/default`

**Then**:
3. Status `200`, `a2.isDefault: true` y `a1.isDefault: false`: como mucho una por defecto por `type`.
4. Se publica **exactamente un** `DefaultAddressChanged` con `subject`, `type: "shipping"`, `addressId: {a2}`, `previousAddressId: {a1}`, `status: "complete"` y el `version` nuevo.

**When**: la **misma** petición otra vez.

**Then**:
5. Status `200` con el mismo estado: fijar un valor es naturalmente idempotente, y por eso la operación no declara `idempotency`.

**When**: `updateMyAddress` — `PUT /api/v1/me/profile/addresses/{a2}` cambiando `type` a `"billing"` y el resto igual.

**Then**:
6. Status `200`, `a2.type: "billing"` y `a2.isDefault: true` (sigue siendo la por defecto, ahora de `billing`).
7. `status: "draft"` — la arista `complete → draft`. Al pasar `a2` a `billing`, el perfil se queda sin ninguna dirección `shipping` marcada por defecto: `a1` es `shipping` pero no está marcada.
8. Se publica **exactamente un** `AddressChanged` con el `postal` completo, `isDefault: true`, `status: "draft"` y el `version` nuevo.

**When**: `setMyDefaultAddress` sobre `{a1}` y después `removeMyAddress` — `DELETE /api/v1/me/profile/addresses/{a1}`

**Then**:
9. Status `200` — **no** `204`: eliminar recalcula el `status` del perfil, un cambio que el cliente no puede predecir.
10. El cuerpo es la proyección completa de `UserProfile` con `addresses` de un solo elemento (`a2`).
11. `status: "draft"` — la arista `complete → draft` por la vía del borrado.
12. Se publica **exactamente un** `AddressRemoved` con `subject`, `addressId: {a1}`, `type: "shipping"`, `wasDefault: true`, `status: "draft"` y el `version` nuevo.
13. Ninguna otra dirección se promocionó a por defecto: `a2.isDefault` sigue siendo `true` **para su propio type** (`billing`), y no hay ninguna `shipping` por defecto.

**Notas de determinación**: la aserción 13 fija la regla de que no se promociona nada automáticamente, que sin escenario se «arregla» sola en la implementación.

### FL-ADR-003: el reintento con la misma clave no crea una segunda dirección

**Given**: `sub-gaby` con contacto completo y sin direcciones.

**When**: `addMyAddress` con `Idempotency-Key: key-g1` y un cuerpo válido.

**Then**:
1. Status `200` y `addresses` con un elemento, `addressId` = `a1`.

**When**: la **misma** petición, con la **misma** clave `key-g1` y el **mismo** cuerpo.

**Then**:
2. Status `200` con el mismo cuerpo que la primera respuesta.
3. `getMyProfile` devuelve `addresses` con **un** solo elemento: no hay segunda dirección.
4. El canal recibió **exactamente un** `AddressAdded`.

**When**: la misma petición con clave **distinta** (`key-g2`) y el mismo cuerpo.

**Then**:
5. Status `200` y `addresses` con **dos** elementos: contenido idéntico no es la misma intención, y la clave es lo que las distingue.

**When**: `Idempotency-Key: key-g1` con un cuerpo **distinto** (`label: "Otra"`).

**Then**:
6. Status `409` con `code: IDEMPOTENCY_KEY_REUSED`.
7. `getMyProfile` sigue devolviendo dos direcciones.

### FL-ADR-004: dos altas con la misma clave a la vez crean una sola dirección

**Given**: `sub-hugo` con contacto completo y sin direcciones.

**When**: dos `POST /api/v1/me/profile/addresses` **a la vez**, las dos con `Idempotency-Key: key-h1` y el mismo cuerpo.

**Then**:
1. O las dos responden `200` con el mismo cuerpo, o una responde `200` y la otra `409` con `code: IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `getMyProfile` devuelve `addresses` con **exactamente un** elemento — la aserción que no depende de quién gane la carrera.
3. El canal recibió **exactamente un** `AddressAdded`.

**Notas de determinación**: no es FL-ADR-003 con otras palabras. El reintento secuencial encuentra el registro de la clave ya confirmado y lo resuelve una lectura; el simultáneo cae en la ventana en la que todavía no lo está, que es donde vive el fallo real y donde un servicio replicado pasa la mayor parte de su vida. Sin la aserción 2, el `Then` solo enumeraría desenlaces admisibles y no podría ponerse rojo.

### FL-ADR-005: los rechazos de las operaciones de dirección

**Given**: `sub-iris` con contacto completo y una dirección `a1`.

**When / Then**:
1. `updateMyAddress` sobre un `addressId` que no existe → `404` con `code: ADDRESS_NOT_FOUND`.
2. `removeMyAddress` sobre un `addressId` que no existe → `404` con `code: ADDRESS_NOT_FOUND`.
3. `setMyDefaultAddress` sobre un `addressId` que no existe → `404` con `code: ADDRESS_NOT_FOUND`.
4. `updateMyAddress` sobre el `addressId` de **otro** perfil → `404` con `code: ADDRESS_NOT_FOUND`, **no** `403`: el 404 no revela que esa dirección exista en otra cuenta.
5. Con veinte direcciones ya creadas, `addMyAddress` → `409` con `code: ADDRESS_LIMIT_REACHED`.
6. `addMyAddress` sin `postal.line1` → `400`.
7. `addMyAddress` sin `postal.localityName` → `400` (`required`), no `TERRITORY_NAME_MISSING`: el nombre de la localidad es obligatorio siempre, así que la validación de forma corre antes.
8. `addMyAddress` con `postal.countryCode: "COL"` → `400` (el value type `CountryCode` exige dos letras).
9. `addMyAddress` sin `label` → `400`.

**Casos borde** (precedencia):
- Con veinte direcciones **y** un `adminAreaCode` incoherente en la misma llamada → `409 ADDRESS_LIMIT_REACHED`: la guarda 5 precede a la 6 del orden de evaluación de FL-ADR-001.
- Sobre un perfil desactivado **y** con veinte direcciones → `409 PROFILE_DEACTIVATED`: la guarda 4 precede a la 5.

---

## La frontera entre la forma y la existencia territorial

### FL-TER-001: una localidad que no existe se guarda tal cual

**Given**: `sub-juan` con contacto completo y sin direcciones. Canal purgado.

**When**: `addMyAddress` — `POST /api/v1/me/profile/addresses`
```json
{ "label": "Casa", "type": "shipping", "isDefault": true,
  "postal": { "line1": "Carrera 70 # 1-20", "line2": null, "line3": null,
              "postalCode": null, "countryCode": "CO",
              "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
              "localityCode": "99999", "localityName": "Medallo" } }
```

**Then**:
1. Status `200`. El servidor **no** rechaza la dirección.
2. `addresses[0].postal.localityName` es exactamente `"Medallo"` y `localityCode` exactamente `"99999"`, sin normalizar, sin corregir y sin sustituir por ningún valor de catálogo.
3. `status: "complete"`.
4. Se publica **exactamente un** `AddressAdded` con ese `postal` literal.
5. `getMyProfile` devuelve los mismos valores.

**Notas de determinación**: este escenario es lo que hace **verificable** la decisión de no validar contra ningún catálogo territorial. Sin él, la regla se queda en prosa y el primero que pase por aquí la «arregla» metiendo una comprobación que el diseño no pidió — y este escenario se pondría rojo. Que el territorio exista, que pertenezca a esa división de primer nivel y que el nombre sea el oficial es responsabilidad del llamante, que lo resuelve en el borde contra su propio catálogo.

### FL-TER-002: la forma sí se valida

**Given**: `sub-juan` con contacto completo.

**When / Then**:
1. `addMyAddress` con `countryCode: "ES"` y `adminAreaCode: "CO-ANT"` → `422` con `code: SUBDIVISION_CODE_COUNTRY_MISMATCH`: el prefijo del código de primer nivel no coincide con el país declarado.
2. `addMyAddress` con `countryCode: "CO"`, `adminAreaCode: "CO-ANT"` y `adminAreaName: null` → `422` con `code: TERRITORY_NAME_MISSING`: nunca se acepta un código sin su nombre.
3. `addMyAddress` con `adminAreaCode: "ANTIOQUIA"` → `400`: el value type `SubdivisionCode` exige la forma ISO 3166-2.
4. `addMyAddress` con `countryCode: "MC"`, `adminAreaCode: null`, `adminAreaName: null`, `localityCode: null`, `localityName: "Monaco"` → `200`: el nivel 1 es opcional y hay países que no lo tienen.

**Notas de determinación**: FL-TER-001 y FL-TER-002 son pareja. Entre los dos queda dibujada la frontera exacta: la **forma** se valida, la **existencia** no.

---

## Back-office

### FL-BO-001: el listado ordena, filtra, busca y pagina

**Given**: creados por la API dentro del flujo, treinta perfiles con `displayName` distinguible (`"Perfil 01"` … `"Perfil 30"`), en orden de alta conocido; el último creado es `"Perfil 30"`. Dos de ellos desactivados. El rol es `profile-support`.

**When**: `listProfiles` — `GET /api/v1/management/profiles`

**Then**:
1. Status `200` con el sobre canónico `{ items, page, size, totalElements, totalPages }`.
2. `page: 0`, `size: 25` (el `defaultSize`), `totalElements: 30`, `totalPages: 2`.
3. `items` tiene 25 elementos, ordenados por `createdAt` **descendente**: el primero es `"Perfil 30"` y el último de la página, `"Perfil 06"`.
4. Cada elemento trae la proyección de `UserProfile` **sin** `addresses`, y ningún campo adicional.

**When**: `GET /api/v1/management/profiles?page=1`

**Then**:
5. `items` tiene 5 elementos, de `"Perfil 05"` a `"Perfil 01"`, sin repetir ninguno de la página anterior.

**When**: `GET /api/v1/management/profiles?page=5`

**Then**:
6. Status `200` con `items` vacío, `totalElements: 30` y `totalPages: 2`.

**When**: `GET /api/v1/management/profiles?size=500`

**Then**:
7. `size: 100` — el `maxSize` recorta, no da error.

**When**: `GET /api/v1/management/profiles?status=deactivated`

**Then**:
8. `totalElements: 2` y los dos elementos tienen `status: "deactivated"`.

**When**: `GET /api/v1/management/profiles?search=PERFIL%200`

**Then**:
9. Devuelve los perfiles cuyo `displayName` **empieza** por `"Perfil 0"`, ignorando mayúsculas.

**When**: `GET /api/v1/management/profiles?search=erfil`

**Then**:
10. `totalElements: 0` — la coincidencia es por **prefijo**, no por subcadena.

**Notas de determinación**: la aserción 3 usa datos que hacen el orden distinguible de cualquier otro orden posible; el `id` del agregado desempata siempre, así que dos páginas consecutivas no repiten ni omiten. La 10 es la que fija que la búsqueda es por prefijo — sin ella, una implementación por subcadena pasaría la 9 igual.

### FL-BO-002: la ficha por subject, y sus dos ausencias

**Given**: `sub-kira` con perfil y dos direcciones. Rol `profile-support`.

**When**: `getProfile` — `GET /api/v1/management/profiles/sub-kira`

**Then**:
1. Status `200` con la proyección completa de `UserProfile`, `addresses` incluido, y ningún campo adicional.

**When**: `GET /api/v1/management/profiles/sub-inexistente`

**Then**:
2. Status `404` con `code: PROFILE_NOT_FOUND` — nunca existió.

**When**: se borra `sub-kira` con `deleteProfile` (rol `profile-admin`) y se repite `GET /api/v1/management/profiles/sub-kira`

**Then**:
3. Status `410` con `code: PROFILE_DELETED` — existió y se borró.

**Notas de determinación**: las aserciones 2 y 3 son el contrato que permite distinguir «nunca existió» de «se borró». Un servidor que respondiera `404` en los dos casos pasaría la 2 y fallaría la 3, que es exactamente lo que se quiere detectar.

### FL-BO-003: desactivar, y no poder desactivar dos veces

**Given**: `sub-luis` con perfil en `draft` (sin direcciones). Canal purgado. Rol `profile-admin`.

**When**: `deactivateProfile` — `POST /api/v1/management/profiles/sub-luis/deactivate`

**Then**:
1. Status `200` con la proyección completa de `UserProfile`.
2. `status: "deactivated"` — la arista `draft → deactivated`.
3. `contactEmail`, `contactPhone` y `addresses` siguen intactos: desactivar no borra ni purga nada.
4. Se publica **exactamente un** `ProfileDeactivated` con `subject: "sub-luis"`, `status: "deactivated"` y el `version` nuevo.

**When**: la misma petición otra vez.

**Then**:
5. Status `409` con `code: PROFILE_ALREADY_DEACTIVATED`.
6. El canal **no** recibe un segundo `ProfileDeactivated`: la transición es irrepetible por construcción, y esa irrepetibilidad es la guarda que sustituye a `idempotency`.

**Casos borde**:
- `deactivateProfile` sobre un subject inexistente → `404 PROFILE_NOT_FOUND`.
- `deactivateProfile` sobre un subject con lápida → `410 PROFILE_DELETED`.

### FL-BO-004: reactivar recalcula la completitud, y no restaura ningún estado guardado

**Given**: dentro del flujo se crean dos perfiles por la API: `sub-mara` con contacto completo y dirección `shipping` por defecto (llega a `complete`) y `sub-nico` sin teléfono ni direcciones (`draft`). Los dos se desactivan. Canal purgado. Rol `profile-admin`.

**When**: `reactivateProfile` — `POST /api/v1/management/profiles/sub-mara/reactivate`

**Then**:
1. Status `200` con `status: "complete"` — la arista `deactivated → complete`.
2. Se publica **exactamente un** `ProfileReactivated` con `subject: "sub-mara"`, `status: "complete"` y el `version` nuevo.

**When**: `POST /api/v1/management/profiles/sub-nico/reactivate`

**Then**:
3. Status `200` con `status: "draft"` — la arista `deactivated → draft`. El estado sale de los datos que el perfil tiene **ahora**, no de ningún estado anterior guardado.
4. Se publica un `ProfileReactivated` con `status: "draft"`.

**When**: `POST /api/v1/management/profiles/sub-mara/reactivate` otra vez (el perfil está en `complete`).

**Then**:
5. Status `409` con `code: PROFILE_NOT_DEACTIVATED` — es la transición inválida: `complete` no está en el `from` de esta operación.
6. El canal no recibe un segundo `ProfileReactivated`.

**Notas de determinación**: sin `ProfileReactivated`, un consumidor dejaría al usuario desactivado para siempre en su réplica. Las aserciones 1 y 3 fijan que el estado se recalcula: un servidor que restaurase un estado guardado devolvería `complete` también para `sub-nico`.

---

## Borrado y lápida

### FL-DEL-001: la baja del propio usuario purga el perfil y publica la señal

**Given**: `sub-olga` con contacto completo y dos direcciones, desactivada dentro del flujo con `deactivateProfile` (rol `profile-admin`). Canal purgado.

**When**: `deleteMyProfile` — `DELETE /api/v1/me/profile` con el token de `sub-olga`.

**Then**:
1. Status `204` sin cuerpo — el borrado del propio usuario no devuelve recibo, y funciona **estando desactivado**: el derecho de supresión no depende del estado.
2. Se publica **exactamente un** `ProfileDeleted` con `subject: "sub-olga"`, `reason: "self-service"`, `deletedAt` con forma de instante UTC y el `version` que tenía el agregado al borrarse.
3. El payload de `ProfileDeleted` **no** trae `contactEmail`, `contactPhone`, `givenName`, `familyName`, `displayName` ni ninguna dirección.
4. `getProfile` sobre `sub-olga` con rol `profile-admin` responde `410 PROFILE_DELETED`.
5. `listProfiles` no lo devuelve en ninguna página: una lápida no es un perfil.
6. `getMyProfile` con el token de `sub-olga` responde `410` con `code: PROFILE_DELETED`.

**Notas de determinación**: la aserción 3 es contrato, no estilo: mandar el contacto de quien acaba de ejercer su derecho de supresión sería lo contrario de lo que el evento pide hacer. La 6 es la puerta del escenario siguiente.

### FL-DEL-002: el borrado administrativo devuelve la lápida, y repetirlo devuelve la misma

**Given**: `sub-pilar` con perfil y una dirección. Canal purgado. Rol `profile-admin`.

**When**: `deleteProfile` — `DELETE /api/v1/management/profiles/sub-pilar`

**Then**:
1. Status `200` con el cuerpo de `DeletedSubject`: `subject: "sub-pilar"`, `deletedAt` (instante UTC), `deletedBy` (el operador autenticado) y `reason: "back-office"`. Ningún campo adicional.
2. Se publica **exactamente un** `ProfileDeleted` con `reason: "back-office"`.

**When**: la **misma** petición otra vez.

**Then**:
3. Status `200` con el **mismo** cuerpo: el mismo `deletedAt` y el mismo `deletedBy` que la primera vez. La lápida es la guarda de repetición y no se reescribe.
4. El canal **no** recibe un segundo `ProfileDeleted`.

**Notas de determinación**: la aserción 3 es lo que permite a un operador distinguir su propio reintento tras un timeout de un borrado ajeno, en la operación más irreversible del servicio. Un servidor que respondiera `410` la segunda vez, o que reescribiera `deletedAt`, fallaría aquí.

### FL-DEL-003: un subject borrado no se vuelve a aprovisionar

**Given**: `sub-quique` tuvo perfil y se borró con `deleteProfile` dentro de este flujo. El canal está purgado **después** del borrado.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con un token válido y vigente de `sub-quique`. La identidad sigue viva en el servidor de identidad: nunca la tocamos.

**Then**:
1. Status `410` con `code: PROFILE_DELETED`.
2. `listProfiles` con rol `profile-admin` **no** devuelve ningún perfil con `subject: "sub-quique"`: no se creó ninguno.
3. El canal **no** ha recibido ningún `ProfileProvisioned`.

**When**: tres peticiones más con el mismo token, a tres endpoints distintos de `/me`.

**Then**:
4. Las tres responden `410 PROFILE_DELETED` y no se crea ningún perfil.

**Notas de determinación**: **este es el escenario que separa un diseño correcto de uno que lo parece.** Como este servicio nunca llama al servidor de identidad, la identidad sobrevive al borrado; sin la lápida, el aprovisionamiento just-in-time le crearía un perfil nuevo en `draft` y el borrado se habría deshecho solo, sin error y sin traza. Las aserciones 2 y 3 son las que lo miden: el status por sí solo no distinguiría un rechazo de un re-aprovisionamiento seguido de un rechazo.

### FL-DEL-004: levantar la lápida es el único acto que permite volver

**Given**: `sub-rosa` borrada dentro del flujo con `deleteProfile`. Canal purgado. Rol `profile-admin`.

**When**: `reinstateSubject` — `POST /api/v1/management/deleted-subjects/sub-rosa/reinstate`

**Then**:
1. Status `204` sin cuerpo.
2. El canal **no** recibe ningún evento: levantar la lápida no cambia ningún perfil.
3. `getProfile` sobre `sub-rosa` responde `404 PROFILE_NOT_FOUND` — ya no hay lápida, y tampoco perfil.

**When**: `GET /api/v1/me/profile` con el token de `sub-rosa`.

**Then**:
4. Status `200` con un perfil **nuevo** en `status: "draft"`, sin direcciones y con `contactPhone` nulo: no se restaura nada de lo que se borró.
5. Se publica **exactamente un** `ProfileProvisioned`.

**When**: `POST /api/v1/management/deleted-subjects/sub-rosa/reinstate` otra vez.

**Then**:
6. Status `404` con `code: SUBJECT_NOT_DELETED`.

**Notas de determinación**: la aserción 4 fija que el perfil no vuelve: el subject empieza de cero. Si un servidor restaurase el contacto o las direcciones, el borrado no habría purgado nada.

### FL-DEL-005: borrar lo que no existe

**Given**: no existe perfil ni lápida para `sub-nadie`. Rol `profile-admin`.

**When / Then**:
1. `DELETE /api/v1/management/profiles/sub-nadie` → `404` con `code: PROFILE_NOT_FOUND`.
2. `DELETE /api/v1/me/profile` con el token de un subject cuyo aprovisionamiento acaba de crear el perfil → `204`: siempre hay perfil que borrar tras una petición autenticada, salvo lápida.

---

## Superficie servidor a servidor

### FL-M2M-001: resolver el contacto de una persona

**Given**: `sub-sara` con `contactEmail: "sara@example.com"`, `contactPhone: "+34600111222"`, `givenName: "Sara"`, `familyName: "Gil"`, `displayName: "Sara Gil"`, una dirección `shipping` por defecto, `status: complete`.

**When**: `resolveContactForServices` — `GET /api/v1/services/profiles/sub-sara/contact` con la credencial de máquina del cliente `notification-service`.

**Then**:
1. Status `200`.
2. El cuerpo es `{ "profile": { … } }` y `profile` trae exactamente `subject`, `displayName`, `givenName`, `familyName`, `contactEmail`, `contactPhone`, `status`, `version` y `updatedAt`. Ningún campo adicional.
3. **No** trae ninguna dirección, ni `shippingAddress`, ni `addresses`, ni ningún campo territorial. `notification-service` no puede ver un domicilio ni pidiéndolo.
4. `status: "complete"`, `version` es un entero y `updatedAt` tiene forma de instante UTC.

**When**: `GET /api/v1/services/profiles/sub-nadie/contact` (nunca existió).

**Then**:
5. Status `404` con `code: PROFILE_NOT_FOUND`.

**When**: se borra `sub-sara` y se repite la llamada.

**Then**:
6. Status `410` con `code: PROFILE_DELETED` — el consumidor debe purgar sus propias copias.

**Notas de determinación**: las aserciones 5 y 6 son contrato de integración, no detalle: un consumidor tiene que poder distinguir «nunca existió» de «se borró» para saber si purgar.

### FL-M2M-002: resolver el perfil de entrega, y el perfil incompleto

**Given**: `sub-tomas` con contacto completo y una dirección `shipping` por defecto con todos los campos territoriales. Y `sub-ulises` con contacto completo pero **sin** ninguna dirección `shipping` por defecto (`status: draft`).

**When**: `resolveDeliveryProfileForServices` — `GET /api/v1/services/profiles/sub-tomas/delivery` con la credencial de máquina del cliente `order-service`.

**Then**:
1. Status `200`.
2. El cuerpo es `{ "profile": { … } }` con `subject`, `displayName`, `givenName`, `familyName`, `contactEmail`, `contactPhone`, `status`, `version`, `updatedAt` y `shippingAddress`. Ningún campo adicional.
3. `shippingAddress` trae `line1`, `line2`, `line3`, `postalCode`, `countryCode`, `adminAreaCode`, `adminAreaName`, `localityCode` y `localityName`, con los valores exactos con que se guardó — **códigos incluidos**.
4. `shippingAddress` **no** trae `id`, ni `label`, ni `isDefault`: son de la UX con la que el usuario la eligió, no del consumidor.

**When**: `GET /api/v1/services/profiles/sub-ulises/delivery` con la misma credencial.

**Then**:
5. Status `409` con `code: PROFILE_INCOMPLETE` — **no** `404` ni `500`: la persona existe, lo que falta es la dirección, y el llamante tiene que poder saberlo para pedirle al usuario que la complete.

**Notas de determinación**: la aserción 3 es lo que hace útil el snapshot: order-service congela esta dirección como dato del pedido y no puede volver a pedirla, así que un snapshot sin códigos no serviría después para liquidar impuestos ni enrutar reparto.

### FL-M2M-003: los lotes, su orden, sus no resueltos y su cota

**Given**: `sub-v1` (perfil completo con dirección por defecto), `sub-v2` (perfil completo con dirección), `sub-v3` (perfil sin dirección `shipping` por defecto), `sub-v4` (borrado, con lápida) y `sub-v5` (nunca existió).

**When**: `resolveDeliveryProfilesBatchForServices` — `POST /api/v1/services/profiles/delivery/batch` con la credencial de máquina del cliente `order-service`
```json
{ "subjects": ["sub-v5", "sub-v1", "sub-v4", "sub-v3", "sub-v2"] }
```

**Then**:
1. Status `200`. El lote **no** falla entero por los tres que no resuelven.
2. `profiles` tiene dos elementos, en el orden de la petición: `sub-v1` y luego `sub-v2`.
3. `unresolved` tiene tres elementos, en el orden de la petición: `{subject: "sub-v5", reason: "not-found"}`, `{subject: "sub-v4", reason: "deleted"}`, `{subject: "sub-v3", reason: "incomplete"}`.
4. Cada elemento de `profiles` tiene la misma forma que el `profile` de FL-M2M-002.

**When**: el mismo lote con `sub-v1` repetido tres veces.

**Then**:
5. `profiles` trae `sub-v1` **una sola vez**.

**When**: `POST /api/v1/services/profiles/delivery/batch` con 101 subjects.

**Then**:
6. Status `422` con `code: TOO_MANY_SUBJECTS`.

**When**: `POST /api/v1/services/profiles/delivery/batch` con `subjects: []`.

**Then**:
7. Status `400`: una lista vacía es una petición mal construida, no una decisión de negocio, así que la rechaza la validación de forma y no lleva `code` propio.

**When**: `resolveContactsBatchForServices` — `POST /api/v1/services/profiles/contact/batch` con la credencial de `notification-service` y los mismos cinco subjects.

**Then**:
8. Status `200`, `profiles` con **tres** elementos (`sub-v1`, `sub-v3`, `sub-v2` en el orden de la petición): para la familia de contacto, `sub-v3` **sí** resuelve — no le falta contacto, le falta dirección.
9. `unresolved` trae solo `sub-v5` (`not-found`) y `sub-v4` (`deleted`); `incomplete` no aparece en esta familia.
10. Ningún elemento de `profiles` trae `shippingAddress`.

**When**: dos peticiones al lote de entrega, una con 2 subjects y otra con 100.

**Then**:
11. El trabajo de la operación **no crece con el tamaño de la lista**: cien subjects no cuestan por elemento lo que cuestan dos. La afirmación es de forma, no de un número absoluto.

**Notas de determinación**: la aserción 11 es la única de coste y está aquí porque ninguna aserción funcional la ve — resolver cien perfiles uno a uno devuelve exactamente el mismo cuerpo que resolverlos en lote, así que un bucle pasaría todos los demás escenarios. La 8 y la 9 fijan que las dos familias tienen contratos distintos y no uno con campos de menos.

### FL-M2M-004: autenticación y mínimo privilegio de la superficie de máquina

**Given**: `sub-sara` con perfil completo.

**When / Then**:
1. `GET /api/v1/services/profiles/sub-sara/contact` **sin credencial** → `401`.
2. `GET /api/v1/services/profiles/sub-sara/delivery` con la credencial de máquina del cliente `notification-service` → `403`: solo tiene `profile-contact:read`, y no hay ninguna operación que le devuelva una dirección.
3. `GET /api/v1/services/profiles/sub-sara/contact` con la credencial de máquina del cliente `order-service` → `403`: solo tiene `profile-delivery:read`. El mínimo privilegio corta en las dos direcciones.
4. `POST /api/v1/services/profiles/delivery/batch` con la credencial de `notification-service` → `403`.
5. `GET /api/v1/services/profiles/sub-sara/contact` con un token de máquina válido **emitido para otra audiencia** → `403`, no `401`: el token es legítimo, simplemente no está emitido para este servicio.
6. `GET /api/v1/management/profiles` con la credencial de máquina de `order-service` → `403`: la superficie de back-office exige privilegio de usuario, no de servicio.

**Notas de determinación**: la aserción 3 no es simétrica por adorno — es lo que comprueba que el scope acota de verdad y no solo etiqueta. El status de la 5 lo fija el generador, no el diseño.

---

## Caché de la superficie de máquina

### FL-CCH-001: las diez vías de mutación invalidan la caché del contacto

**Given**: `sub-wanda` con perfil completo y una dirección `shipping` por defecto. La credencial de máquina es la del cliente `notification-service`.

**When / Then**: por **cada** una de las diez vías de `invalidatedBy`, en este orden, el ciclo es siempre el mismo — se lee `GET /api/v1/services/profiles/{subject}/contact` (la caché se puebla), se muta, y se vuelve a leer:

1. `ProfileProvisioned`: se lee un subject inexistente (`404`), se provoca su aprovisionamiento con una petición autenticada, y la lectura siguiente devuelve `200` con el perfil nuevo.
2. `ProfileContactEmailRefreshed`: cambia el claim `email` del token y una petición a `/me` lo refresca; la lectura siguiente devuelve el `contactEmail` nuevo.
3. `ProfileContactDetailsChanged`: `updateMyContactDetails` cambia `contactPhone`; la lectura siguiente devuelve el teléfono nuevo.
4. `AddressAdded`: se añade una dirección y el perfil pasa a `complete`; la lectura siguiente devuelve `status: "complete"`.
5. `AddressChanged`: `updateMyAddress` cambia el `type` a `billing` y el perfil pasa a `draft`; la lectura siguiente devuelve `status: "draft"`.
6. `AddressRemoved`: se elimina la dirección por defecto; la lectura siguiente devuelve el `status` recalculado.
7. `DefaultAddressChanged`: `setMyDefaultAddress` sobre otra dirección `shipping` devuelve el perfil a `complete`; la lectura siguiente lo refleja.
8. `ProfileDeactivated`: la lectura siguiente devuelve `status: "deactivated"`.
9. `ProfileReactivated`: la lectura siguiente devuelve el `status` recalculado.
10. `ProfileDeleted`: la lectura siguiente devuelve `410 PROFILE_DELETED` — la vía más importante de las diez: seguir sirviendo datos de quien pidió el borrado es el peor desenlace posible de una invalidación incompleta.

11. En los diez ciclos, el `version` que devuelve la lectura posterior es **mayor** que el de la anterior (salvo en el 10, que no devuelve cuerpo).

**Notas de determinación**: los eventos de dirección están en la lista aunque no toquen el contacto, porque cambian el `status`, que **sí** viaja en esta respuesta. Basta una vía no listada para servir datos rancios indefinidamente, y el `200` no distingue una respuesta vieja de una fresca.

### FL-CCH-002: las diez vías invalidan también la caché del perfil de entrega

**Given**: `sub-xavi` con perfil completo y dirección `shipping` por defecto. La credencial de máquina es la del cliente `order-service`.

**When / Then**: el mismo recorrido de FL-CCH-001, esta vez sobre `GET /api/v1/services/profiles/{subject}/delivery`, con dos diferencias propias de esta familia:

1. Las diez vías invalidan, con los mismos ciclos leer → mutar → releer.
2. Tras `AddressChanged` que deja el perfil sin dirección `shipping` por defecto, la lectura siguiente devuelve `409 PROFILE_INCOMPLETE`, no el cuerpo viejo con la dirección anterior.
3. Tras `AddressChanged` sobre la dirección por defecto sin cambiar su `type`, la lectura siguiente devuelve el `shippingAddress` **nuevo**, campo a campo.
4. Tras `ProfileDeleted`, la lectura siguiente devuelve `410 PROFILE_DELETED`.

**Notas de determinación**: la aserción 2 es la que más fácil se escapa: la respuesta pasa de `200` con cuerpo a `409`, y una invalidación incompleta seguiría entregando una dirección que el usuario ya no tiene marcada.

---

## Fiabilidad de la publicación

### FL-OBX-001: el evento sobrevive a un canal indisponible

**Given**: `sub-yago` con perfil completo, el canal `profileEvents` sin mensajes y el canal de eventos **indisponible**.

**When**: `updateMyContactDetails` con un cuerpo válido.

**Then**:
1. Status `200` con el cuerpo completo del perfil: la indisponibilidad del canal **no llega al cliente**.
2. `getMyProfile` devuelve el contacto nuevo: el cambio está confirmado.
3. El canal `profileEvents` **no** ha recibido ningún mensaje todavía.

**When**: el canal vuelve a estar disponible.

**Then**:
4. En ≤ 10 s el canal `profileEvents` recibe **exactamente un** `ProfileContactDetailsChanged`.
5. Su payload es el del perfil modificado, y su `correlationId` el de la petición del paso anterior.
6. El servidor no se ha rendido con ningún evento: ningún evento abandonado.

**Notas de determinación**: las dos aserciones que impiden que este escenario pase por accidente son la **3** y el «exactamente un» de la **4**. La 3 separa el outbox de publicar en línea dentro de la operación —un servicio que publica directo pasaría el resto del escenario igual, porque el mensaje también acaba llegando—; el «exactamente uno» separa un relay que marca lo entregado de uno que reentrega para siempre.

### FL-OBX-002: el evento que el relay abandona no se pierde en silencio

**Given**: el canal indisponible y una mutación ya ejecutada sobre `sub-zoe`, con su evento pendiente de salir.

**When**: se agota el presupuesto de reintentos de ese evento.

**Then**:
1. El servidor lo dice: informa de **un** evento abandonado.
2. Restablecido el canal, ese evento **no** se publica — el relay respeta que se rindió.
3. Y el canal no recibe ninguna otra cosa.

**Notas de determinación**: el punto 2 es el que lo hace algo más que una prueba de la señal: sin él, un relay que ignorase su propio presupuesto reintentaría para siempre una fila ya dada por perdida y pasaría igual. El escenario habla del **presupuesto de reintentos**, nunca de su número ni del nombre de la métrica: los dos son del generador.

---

## Seguridad y consumo desde el navegador

### FL-SEC-001: cada superficie exige lo suyo

**Given**: `sub-ana` con perfil completo.

**When / Then**:
1. `GET /api/v1/me/profile` **sin credencial** → `401`.
2. `PUT /api/v1/me/profile/contact-details` sin credencial → `401`.
3. `GET /api/v1/management/profiles` sin credencial → `401`.
4. `GET /api/v1/management/profiles` con un token de usuario **sin ningún rol** → `403`.
5. `GET /api/v1/management/profiles` con el rol `profile-support` → `200`.
6. `DELETE /api/v1/management/profiles/sub-ana` con el rol `profile-support` → `403`: `profile-support` no tiene `profile:delete`. El corte entre los dos roles está donde está la irreversibilidad.
7. `POST /api/v1/management/deleted-subjects/sub-ana/reinstate` con el rol `profile-support` → `403`.
8. `DELETE /api/v1/management/profiles/sub-ana` con el rol `profile-admin` → `200`.
9. `GET /api/v1/me/profile` con el token de `sub-ana` devuelve **su** perfil, y no hay ninguna forma de pedir el de otro: el subject no viaja ni en la ruta ni en el cuerpo, lo estampa el servidor desde el claim `sub`.

**Cobertura por operación** — las 17 operaciones con endpoint, cada una llamada **sin credencial** y **con una credencial autenticada que no tiene lo que la operación exige**:

| Operaciones | Sin credencial | Credencial insuficiente |
|---|---|---|
| las 7 de `/me` | `401` | — (no exigen permiso: basta un token de persona; el alcance lo da el claim `sub`) |
| `listProfiles`, `getProfile` | `401` | token de usuario sin rol → `403` |
| `deactivateProfile`, `reactivateProfile` | `401` | token de usuario sin rol → `403` |
| `deleteProfile`, `reinstateSubject` | `401` | rol `profile-support` → `403` |
| las 4 de `/services` | `401` | `403` con la credencial de máquina del cliente `order-service` sobre la familia de contacto, y con la del cliente `notification-service` sobre la de entrega (FL-M2M-004) |

`provisionProfileFromIdentity` no aparece: es `internal: true` y no tiene endpoint propio — su control de acceso se ejercita en FL-PRV-002, donde una credencial de máquina no aprovisiona.

**Notas de determinación**: la aserción 6 es la que mide que el reparto de privilegio no es decorativo. La 9 es la autorización a nivel de dato de toda la superficie `/me`: no la da un rol, la da que el cliente no pueda elegir en nombre de quién actúa.

### FL-SEC-002: el preflight no muere en la cadena de seguridad

**Given**: el servicio en marcha. La petición viene de un origen web.

**When**: `OPTIONS /api/v1/me/profile/addresses` **sin credencial**, con `Origin` de un cliente web, `Access-Control-Request-Method: POST` y `Access-Control-Request-Headers: Authorization, Content-Type, Idempotency-Key`.

**Then**:
1. La respuesta acepta el método `POST`.
2. Acepta las tres cabeceras solicitadas — incluida `Idempotency-Key`, sin la cual la idempotencia de `addMyAddress` no funcionaría desde un navegador.
3. Declara el tiempo de cacheo del preflight en `3600` segundos.
4. **No** admite credenciales (`allowCredentials: false`).

**Notas de determinación**: el escenario habla de la **política** y de «un origen web», nunca de los orígenes concretos: esos son despliegue y no están en el diseño.

### FL-SEC-003: la respuesta llega utilizable al navegador

**Given**: `sub-ana` con perfil.

**When**: `GET /api/v1/me/profile` cross-origin, con token válido y `Origin` de un cliente web.

**Then**:
1. Status `200` con el cuerpo completo del perfil.
2. El navegador puede leer la cabecera `X-Correlation-Id` de la respuesta (`exposedHeaders`).

**Notas de determinación**: no es FL-SEC-002 con otras palabras. Un servicio que contesta al preflight y no expone sus cabeceras pasa aquel y rompe al cliente igual.

---

## Lo que no tiene escenario, y por qué

Un hueco declarado es honesto; uno tapado con un escenario decorativo es peor, porque además apaga la sospecha.

- **La retención de la caché.** El formato pide un escenario que mute el dato por una vía que **no** esté en `invalidatedBy` y compruebe que la lectura sigue sirviendo el valor viejo dentro del TTL — es lo que distingue una caché sana de una que no cachea nada. **Aquí no existe esa vía**: `invalidatedBy` enumera las **diez** vías de mutación del servicio, que son todas, y no hay ninguna forma de cambiar el dato sin publicar uno de esos diez eventos. Es una consecuencia directa de que la lista de eventos sea completa sobre la superficie de mutación, que es justamente la propiedad que el diseño quería. La contrapartida, escrita para que nadie la descubra tarde: **ningún escenario de este documento distingue una caché que funciona de la ausencia de caché**, porque sin caché el dato también sale fresco. El `ttlSeconds: 60` queda verificado estáticamente, contra el artefacto, y no por ejecución.
- **La retención de perfiles.** El diseño declara que no hay ninguna: un perfil vive mientras nadie lo borre. No hay barrido que ejercitar porque no hay barrido.
- **Ninguna operación declara `schedule`**, así que no hay escenarios de solapamiento, de recuperación tras parada ni de clúster.
- **No hay capa `subscriptions`, `storage`, `mail`, `dependencies` ni `http-clients`**, así que no hay consumo de eventos ajenos, ni archivos, ni correo, ni compensaciones, ni reconciliación, ni resiliencia de llamadas salientes que validar.
- **Ninguna creación devuelve `201`**, así que no hay ninguna cabecera `Location` que afirmar.
