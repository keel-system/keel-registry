# user-profile — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/user-profile v0.1.1. Contrato de validación para la fase de generación.

## Convenciones de determinación

- **Aislamiento**: el estado se reinicia antes de cada flujo: cada flujo arranca **sin perfiles ni
  lápidas** y con el canal `profileEvents` vacío. Dentro del flujo, los pasos encadenan estado.
- **Rutas**: todas bajo `/api/v1`. Un `subject` en la ruta viaja codificado como segmento de URL.
- **Ausencia vs nulo** (declarado en el manifiesto, `conventions.nulls: include`): todo campo declarado
  viaja; sin valor, como `null`, en respuestas y en eventos. Una colección vacía viaja como `[]`.
- **Fechas**: todo instante es ISO-8601 en UTC con milisegundos y sufijo `Z`
  (`2026-10-01T09:21:07.482Z`). Se verifica por forma y por relación (`updatedAt` ≥ `createdAt`; un
  valor «que avanza» es estrictamente mayor que el leído antes; uno «que no cambia» es igual), nunca
  por valor exacto.
- **Identificadores generados**: forma uuid, verificados por reutilización simbólica (`p1`, `a1`…),
  nunca por valor literal. El `subject` no es generado: es el claim `sub` del token.
- **version / lockVersion**: un perfil nace con `lockVersion: 0`, y cada mutación confirmada del
  agregado (también las de sus direcciones) la sube exactamente en 1. Una mutación cuyo resultado es
  igual a lo guardado no es una mutación: no sube `lockVersion`, no cambia `updatedAt` ni publica evento.
  El `version` de cada evento es el `lockVersion` tras el cambio; en `ProfileDeleted`, el último más uno.
- **Mayúsculas y acentos** (declarado en el YAML: `listProfiles.search` con `match: prefix`,
  `compare: ignore-case-accents`): `search` casa por **prefijo** (nunca subcadena) en `contactEmail` o
  `displayName`, sin distinguir mayúsculas ni acentos. El resto de comparaciones son exactas
  (`subject` byte a byte; el refresco de `contactEmail` compara el claim tal cual).
- **Orden**: `addresses` viaja en orden de alta (de la más antigua a la más reciente); la última
  añadida es siempre la última. `listProfiles` ordena por `createdAt` descendente con desempate por
  `id`. Los lotes M2M conservan el orden de la primera aparición de cada subject en la petición.
- **Proyecciones** (derivadas del YAML; toda enumeración de este documento las respeta):
  - `UserProfile` (`getMyProfile`, `getProfile` y las mutaciones de `/me`, de-/reactivación):
    `{ id, subject, contactEmail, givenName, familyName, displayName, contactPhone, status,
    deactivatedBy, deactivatedAt, lockVersion, createdAt, updatedAt, addresses }`. Fuera de
    `deactivated`, `deactivatedBy` y `deactivatedAt` viajan a `null`.
  - `Address` (dentro de `addresses`): `{ id, userProfileId, label, type, postal, isDefault,
    createdAt, updatedAt }`; `userProfileId` es el `id` del perfil.
  - `PostalAddress` (`postal`, `shippingAddress`): `{ line1, line2, line3, postalCode, countryCode,
    adminAreaCode, adminAreaName, localityCode, localityName }`.
  - `listProfiles`: `UserProfile` **sin** `addresses`.
  - `DeletedSubject` (`deleteProfile`): `{ subject, deletedAt, deletedBy, reason }`.
  - Contacto M2M: `{ subject, displayName, givenName, familyName, contactEmail, contactPhone, status,
    version, updatedAt }`. Entrega M2M: lo mismo más `shippingAddress`. `version` = `lockVersion` y
    `updatedAt` = `updatedAt` del perfil.
  - Ninguna respuesta lleva `createdBy`/`updatedBy` (la autoría es `all`: existe fuera del contrato).
- **Cuerpo de error**: forma fija del generador —`{timestamp, status, error, code, message,
  details}` más `correlationId`—. Los escenarios fijan `code` y status; `message` no es contrato.
  `details` solo es contrato en `PROFILE_INCOMPLETE`: la lista de lo que falta, con los nombres
  estables `contactEmail`, `contactPhone`, `displayName`, `defaultShippingAddress` y en ese orden.
- **Errores del mecanismo** (canónicos de `framework-errors.md`): entrada fuera de cotas →
  `400 VALIDATION_ERROR`; sin credencial → `401 UNAUTHENTICATED`; sin permiso, sin scope o token de
  otra audiencia → `403 ACCESS_DENIED`; misma clave a la vez → `409 IDEMPOTENCY_KEY_IN_PROGRESS`; misma
  clave con otro cuerpo → `409 IDEMPOTENCY_KEY_REUSED`; escritura concurrente sobre el mismo perfil →
  `409 CONCURRENT_MODIFICATION`. La validación de forma (`400`) precede a cualquier otra guarda.
- **Idempotencia**: solo `addMyAddress`, con la cabecera opcional `Idempotency-Key`, acotada al
  titular y válida 24 h; sin cabecera se ejecuta sin deduplicar.
- **Paginación** (sobre canónico): `{ items, page, size, totalElements, totalPages }`, `page` base 0,
  `size` 25 por defecto y tope 100 (un `size` mayor se recorta a 100). Sin elementos: `items: []`,
  `totalElements: 0`, `totalPages: 0`. Página fuera de rango: `items: []` con `totalElements` y
  `totalPages` reales.
- **Eventos**: se observan en el canal `profileEvents`, con la envoltura Keel (`metadata` + `data`);
  los escenarios afirman `data` completo. «Ningún evento» significa que el canal no recibe nada
  durante el escenario.
- **Usuarios de prueba** (token de persona, OIDC, audiencia del servicio):
  - `ana`: `sub: "sub-ana-001"`, `email: "ana@example.com"`, `given_name: "Ana"`,
    `family_name: "Pérez"`, `name: "Ana Pérez"`.
  - `beto`: `sub: "sub-beto-002"`, `email: "beto@example.com"`, `name: "Beto"`, sin `given_name` ni
    `family_name`.
  - `cleo`: `sub: "sub-cleo-003"`, `email: "cleo@example.com"`, `given_name: "Cleo"`,
    `family_name: "Ruiz"`, `name: "Cleo Ruiz"`.
  - `dora`: `sub: "sub-dora-004"`, `email: "dora@example.com"`, `given_name: "Ángela"`,
    `family_name: "Dora"`, `name: "Ángela Dora"`.
  - `eva`: `sub: "sub-eva-005"`, `email: "eva@example.com"`, `name: "Eva"`.
  - `dani`: `sub: "sub-dani-006"`, `name: "Dani"`, **sin** `email`.
  - `admin`: rol `profile-admin`, `sub: "sub-op-admin"`. `support`: rol `profile-support`,
    `sub: "sub-op-support"`. Ninguna persona de las anteriores tiene roles.
  - Cuando un escenario cambia un claim («token de `ana` con `email: …`»), el resto de claims son los
    de arriba.
- **Clientes máquina de prueba**: credencial de máquina del cliente `order-service` (scope
  `profile-delivery:read`) y credencial de máquina del cliente `notification-service` (scope
  `profile-contact:read`), ambas con audiencia `user-profile` salvo donde el escenario diga otra.
- **Direcciones de ejemplo** (cuerpos de `addMyAddress`; `updateMyAddress` usa los mismos sin `isDefault`):
  - `ADDR-CASA`:
    ```json
    { "label": "Casa", "type": "shipping", "isDefault": true,
      "postal": { "line1": "Calle 10 # 43-12", "line2": "Apto 301", "line3": null, "postalCode": "050021",
                  "countryCode": "CO", "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
                  "localityCode": "05001", "localityName": "Medellín" } }
    ```
  - `ADDR-OFI` (sin `isDefault`):
    ```json
    { "label": "Oficina", "type": "shipping",
      "postal": { "line1": "Carrera 7 # 72-41", "line2": null, "line3": null, "postalCode": "110231",
                  "countryCode": "CO", "adminAreaCode": "CO-DC", "adminAreaName": "Bogotá D.C.",
                  "localityCode": "11001", "localityName": "Bogotá" } }
    ```
  - `ADDR-FACT`:
    ```json
    { "label": null, "type": "billing", "isDefault": true,
      "postal": { "line1": "Calle de Alcalá 50", "line2": null, "line3": null, "postalCode": "28014",
                  "countryCode": "ES", "adminAreaCode": "ES-M", "adminAreaName": "Madrid",
                  "localityCode": null, "localityName": "Madrid" } }
    ```
  - `POSTAL(X)` es el objeto `postal` de la dirección `X`, campo a campo.
- **Contacto completo de ejemplo** (`CONTACT-ANA`, cuerpo de `updateMyContactDetails`):
  `{ "givenName": "Ana", "familyName": "Pérez", "displayName": "Ana Pérez", "contactPhone": "+573001234567" }`.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|-----------|--------|------------|
| provisionProfileFromIdentity | FL-PRV-001, FL-PRV-002, FL-PRV-003, FL-PRV-004, FL-PRV-005, FL-PRV-006, FL-PRV-007, FL-DEL-004 | interna (todas las de `/me` salvo el borrado) |
| getMyProfile | FL-PRV-001, FL-PRV-002, FL-PRV-003, FL-PRV-004, FL-PRV-006, FL-PRV-007, FL-ADR-010, FL-DEL-001, FL-DEL-004 | usuarios |
| updateMyContactDetails | FL-ME-001, FL-ME-002, FL-PRV-005, FL-ADR-009, FL-ADR-010, FL-OBX-001, FL-OBX-002 | usuarios |
| addMyAddress | FL-ADR-001, FL-ADR-002, FL-ADR-003, FL-ADR-004, FL-ADR-005, FL-ADR-009, FL-ADR-010 | usuarios |
| updateMyAddress | FL-ADR-006, FL-ADR-004, FL-ADR-010 | usuarios |
| removeMyAddress | FL-ADR-007, FL-ADR-005, FL-ADR-010 | usuarios |
| setMyDefaultAddress | FL-ADR-008, FL-ADR-009, FL-ADR-010 | usuarios |
| deleteMyProfile | FL-DEL-001, FL-DEL-002, FL-DEL-003, FL-DEL-005, FL-PRV-006 | usuarios |
| listProfiles | FL-BO-001, FL-BO-004 | usuarios (back-office) |
| getProfile | FL-BO-002, FL-DEL-001, FL-DEL-004, FL-BO-004 | usuarios (back-office) |
| deactivateProfile | FL-BO-003, FL-PRV-005, FL-ADR-005, FL-ADR-010, FL-BO-004 | usuarios (back-office) |
| reactivateProfile | FL-BO-003, FL-BO-004 | usuarios (back-office) |
| deleteProfile | FL-DEL-004, FL-DEL-001, FL-DEL-003, FL-DEL-005, FL-BO-004 | usuarios (back-office) |
| reinstateSubject | FL-DEL-004, FL-BO-004 | usuarios (back-office) |
| resolveContactForServices | FL-M2M-001, FL-M2M-005 | **servidores (M2M)** |
| resolveDeliveryProfileForServices | FL-M2M-002, FL-M2M-006 | **servidores (M2M)** |
| resolveContactsBatchForServices | FL-M2M-003 | **servidores (M2M)** |
| resolveDeliveryProfilesBatchForServices | FL-M2M-004 | **servidores (M2M)** |
| **outbox (profileEvents)** | FL-OBX-001, FL-OBX-002 | mecanismo |
| **CORS** | FL-CORS-001, FL-CORS-002 | mecanismo |

## Aprovisionamiento

### FL-PRV-001: el primer acceso crea el perfil en draft con los claims del token

**Given**: no existe perfil ni lápida para `sub-ana-001`. El canal `profileEvents` está vacío.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con el token de `ana`, sin cuerpo.

**Then**:
1. Status `200`.
2. El cuerpo es la proyección `UserProfile` con `id` (uuid, en adelante `p1`), `subject: "sub-ana-001"`,
   `contactEmail: "ana@example.com"`, `givenName: "Ana"`, `familyName: "Pérez"`,
   `displayName: "Ana Pérez"`, `contactPhone: null`, `status: "draft"`, `deactivatedBy: null`,
   `deactivatedAt: null`, `lockVersion: 0`, `createdAt` y `updatedAt` instantes válidos,
   `addresses: []`. No trae ningún campo adicional.
3. El canal `profileEvents` recibe exactamente un `ProfileProvisioned` con `data`
   `{ subject: "sub-ana-001", version: 0, status: "draft", contactEmail: "ana@example.com",
   givenName: "Ana", familyName: "Pérez", displayName: "Ana Pérez" }`.

**When**: se repite `GET /api/v1/me/profile` con el mismo token.

**Then**:
4. Status `200` con el mismo cuerpo: mismo `id` `p1`, `lockVersion: 0`, mismo `updatedAt`.
5. Ningún evento: no se publica un segundo `ProfileProvisioned`.
6. `getProfile` — `GET /api/v1/management/profiles/sub-ana-001` con el token de `admin` responde `200`
   con el mismo cuerpo.

**Notas de determinación**: el aprovisionamiento confirma en su propia transacción; la idempotencia
la garantiza la unicidad de `subject` (`payload-field` sobre `callerSubject`).

### FL-PRV-002: los claims ausentes o inválidos quedan a null y no hacen fallar la petición

**Given**: no existe perfil para `sub-beto-002` ni para `sub-cleo-003`.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con el token de `beto` con `email: "no-es-un-correo"`.

**Then**:
1. Status `200`; cuerpo `UserProfile` con `subject: "sub-beto-002"`, `contactEmail: null` (el claim no
   cumple `EmailAddress` y se trata como ausente), `givenName: null`, `familyName: null`,
   `displayName: "Beto"`, `contactPhone: null`, `status: "draft"`, `deactivatedBy: null`,
   `deactivatedAt: null`, `lockVersion: 0`, `addresses: []`.
2. `profileEvents` recibe un `ProfileProvisioned` con `data` `{ subject: "sub-beto-002", version: 0,
   status: "draft", contactEmail: null, givenName: null, familyName: null, displayName: "Beto" }`.

**When**: `GET /api/v1/me/profile` con el token de `cleo` con `name` de 101 caracteres.

**Then**:
3. Status `200`; `displayName: null` (el claim excede `PersonName` y se trata como ausente),
   `givenName: "Cleo"`, `familyName: "Ruiz"`, `contactEmail: "cleo@example.com"`.

**Casos borde**:
- Un token sin ningún claim salvo `sub` (`sub-zz-900`) → `200` con todos los campos de contacto a `null`.

### FL-PRV-003: dos primeros accesos a la vez crean un único perfil

**Given**: no existe perfil para `sub-cleo-003`. El canal `profileEvents` está vacío.

**When**: dos `GET /api/v1/me/profile` con el token de `cleo` se envían **a la vez**.

**Then**:
1. Las dos responden `200` con el mismo `id` y `lockVersion: 0`: el cliente nunca ve la colisión de
   unicidad.
2. `profileEvents` recibe **exactamente un** `ProfileProvisioned` con `subject: "sub-cleo-003"`.
3. `listProfiles` — `GET /api/v1/management/profiles?search=cleo` con el token de `admin` devuelve
   `totalElements: 1`.

### FL-PRV-004: el email se refresca desde el token y nada más

**Given**: `ana` ya hizo `GET /api/v1/me/profile` (perfil `p1`, `lockVersion: 0`, `status: "draft"`);
el canal `profileEvents` se purga después.

**When**: `GET /api/v1/me/profile` con el token de `ana` con `email: "ana.perez@example.org"` y
`given_name: "Anita"`.

**Then**:
1. Status `200`; `contactEmail: "ana.perez@example.org"`, `givenName: "Ana"` (los demás claims no
   refrescan nada), `status: "draft"`, `lockVersion: 1`, `updatedAt` avanza.
2. `profileEvents` recibe un `ProfileContactEmailRefreshed` con `data`
   `{ subject: "sub-ana-001", version: 1, status: "draft", contactEmail: "ana.perez@example.org" }`.

**When**: `GET /api/v1/me/profile` con el token de `ana` **sin** claim `email`.

**Then**:
3. Status `200`; `contactEmail: "ana.perez@example.org"` (sin claim no se toca), `lockVersion: 1`.
4. Ningún evento.

**When**: `GET /api/v1/me/profile` con el token de `ana` con `email: "ana.perez@example.org"` (el mismo).

**Then**:
5. Status `200`, `lockVersion: 1`; ningún evento.

**Casos borde**:
- Token con `email: "Ana.Perez@example.org"` (cambia solo la caja) → la comparación es exacta: refresca,
  `lockVersion: 2`, `ProfileContactEmailRefreshed` con ese valor tal cual.

### FL-PRV-005: el refresco recalcula el estado, también en un perfil desactivado

**Given**: no existe perfil para `sub-dani-006`.

**When**: `dani` (token sin `email`) ejecuta en orden: `GET /api/v1/me/profile`;
`updateMyContactDetails` — `PUT /api/v1/me/profile/contact-details` con
`{ "displayName": "Dani", "contactPhone": "+573005550000" }`; `addMyAddress` —
`POST /api/v1/me/profile/addresses` con `ADDR-CASA`.

**Then**:
1. Tras los tres pasos: `contactEmail: null`, `status: "draft"` (falta el email) y `lockVersion: 2`.

**When**: `GET /api/v1/me/profile` con el token de `dani` con `email: "dani@example.com"`.

**Then**:
2. Status `200`; `contactEmail: "dani@example.com"`, `status: "complete"` (draft → complete),
   `lockVersion: 3`.
3. `profileEvents` recibe `ProfileContactEmailRefreshed` con `data` `{ subject: "sub-dani-006",
   version: 3, status: "complete", contactEmail: "dani@example.com" }`.

**When**: `deactivateProfile` — `POST /api/v1/management/profiles/sub-dani-006/deactivate` con el token
de `admin`; después `GET /api/v1/me/profile` con el token de `dani` con `email: "dani@new.example.com"`.

**Then**:
4. La desactivación responde `200` con `status: "deactivated"`, `lockVersion: 4`.
5. El `GET` responde `200` con `contactEmail: "dani@new.example.com"`, `status: "deactivated"`
   (deactivated → deactivated: el refresco ocurre en cualquier estado) y `lockVersion: 5`.
6. `profileEvents` recibe `ProfileContactEmailRefreshed` con `data` `{ subject: "sub-dani-006",
   version: 5, status: "deactivated", contactEmail: "dani@new.example.com" }`.

### FL-PRV-006: un token de máquina no tiene perfil

**Given**: el canal `profileEvents` está vacío.

**When**: `getMyProfile` — `GET /api/v1/me/profile` con la credencial de máquina del cliente `order-service`.

**Then**:
1. Status `403` con `code: "SUBJECT_NOT_PROVISIONABLE"`.
2. Ningún evento; `GET /api/v1/management/profiles` con el token de `admin` devuelve `totalElements: 0`.

**When**: `updateMyContactDetails` — `PUT /api/v1/me/profile/contact-details` con la misma credencial y
`CONTACT-ANA`.

**Then**:
3. Status `403` con `code: "SUBJECT_NOT_PROVISIONABLE"`.

**When**: `deleteMyProfile` — `DELETE /api/v1/me/profile` con la misma credencial.

**Then**:
4. Status `404` con `code: "PROFILE_NOT_FOUND"`: un sujeto máquina nunca tiene perfil ni lápida.

### FL-PRV-007: un subject con lápida no se vuelve a aprovisionar

**Given**: `ana` hizo `GET /api/v1/me/profile` (perfil `p1`) y `addMyAddress` con `ADDR-CASA` (dirección
`a1`); después `deleteMyProfile` — `DELETE /api/v1/me/profile` → `204`. Canal purgado.

**When**: con el token de `ana`, en orden: `GET /api/v1/me/profile`;
`PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`; `POST /api/v1/me/profile/addresses` con
`ADDR-OFI`; `PUT /api/v1/me/profile/addresses/{a1}` con `ADDR-OFI`;
`DELETE /api/v1/me/profile/addresses/{a1}`; `PUT /api/v1/me/profile/addresses/{a1}/default`.

**Then**:
1. Las seis responden `410` con `code: "PROFILE_DELETED"`.
2. Ningún evento: no se crea perfil ni se publica `ProfileProvisioned`.

## Datos de contacto

### FL-ME-001: el bloque de contacto se reemplaza entero

**Given**: `ana` hizo `GET /api/v1/me/profile` (perfil `p1`, `lockVersion: 0`). Canal purgado.

**When**: `updateMyContactDetails` — `PUT /api/v1/me/profile/contact-details` con
```json
{ "givenName": "Ana María", "familyName": "Pérez Gómez", "displayName": "Ana P.", "contactPhone": "+573001234567" }
```

**Then**:
1. Status `200`; cuerpo `UserProfile` con `id: p1`, `subject: "sub-ana-001"`,
   `contactEmail: "ana@example.com"`, `givenName: "Ana María"`, `familyName: "Pérez Gómez"`,
   `displayName: "Ana P."`, `contactPhone: "+573001234567"`, `status: "draft"` (draft → draft: no hay
   dirección), `deactivatedBy: null`, `deactivatedAt: null`, `lockVersion: 1`, `updatedAt` avanza,
   `addresses: []`.
2. `profileEvents` recibe un `ProfileContactDetailsChanged` con `data` `{ subject: "sub-ana-001",
   version: 1, status: "draft", givenName: "Ana María", familyName: "Pérez Gómez",
   displayName: "Ana P.", contactPhone: "+573001234567" }`.

**When**: se repite el mismo `PUT` con el mismo cuerpo.

**Then**:
3. Status `200` con el mismo cuerpo, `lockVersion: 1` y el mismo `updatedAt`.
4. Ningún evento.

**When**: `PUT /api/v1/me/profile/contact-details` con `{ "displayName": "Ana" }`.

**Then**:
5. Status `200`; `givenName: null`, `familyName: null`, `displayName: "Ana"`, `contactPhone: null`,
   `contactEmail: "ana@example.com"`, `lockVersion: 2`.
6. `ProfileContactDetailsChanged` con `data` `{ subject: "sub-ana-001", version: 2, status: "draft",
   givenName: null, familyName: null, displayName: "Ana", contactPhone: null }`.

**When**: `PUT /api/v1/me/profile/contact-details` **sin cuerpo**.

**Then**:
7. Status `200`; los cuatro campos del bloque a `null`, `lockVersion: 3`, y un
   `ProfileContactDetailsChanged` con `version: 3` y los cuatro a `null`.

**Orden de evaluación**:
1. Forma del cuerpo → `400 VALIDATION_ERROR`.
2. Aprovisionamiento → `403 SUBJECT_NOT_PROVISIONABLE` o `410 PROFILE_DELETED` (FL-PRV-006, FL-PRV-007).
3. Perfil no desactivado → `PROFILE_DEACTIVATED` (`409`) (FL-ADR-010).
4. Escritura → `CONCURRENT_MODIFICATION` (`409`) (FL-ADR-009).

**Casos borde**:
- `contactPhone: "3001234567"` (sin `+`) → `400`. `displayName: ""` → `400`. `givenName` de 101
  caracteres → `400`. En los tres, `lockVersion` no cambia.
- Cuerpo con `contactEmail: "otro@example.com"` además de `CONTACT-ANA` → `200`; `contactEmail` sigue
  siendo `"ana@example.com"` (se ignora).

### FL-ME-002: la completitud se recalcula en cada cambio

**Given**: `ana` hizo `GET /api/v1/me/profile` (`lockVersion: 0`, `status: "draft"`).

**When**: `PUT /api/v1/me/profile/contact-details` con `{ "displayName": "Ana Pérez", "contactPhone": "+573001234567" }`.

**Then**:
1. `200`, `status: "draft"` (falta la dirección shipping predeterminada), `lockVersion: 1`.

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-CASA`.

**Then**:
2. `201`, `status: "complete"` (draft → complete), `lockVersion: 2`; `AddressAdded` con `status: "complete"`.

**When**: `PUT /api/v1/me/profile/contact-details` con `{ "displayName": "Ana Pérez", "contactPhone": "+573009999999" }`.

**Then**:
3. `200`, `status: "complete"` (complete → complete), `lockVersion: 3`; `ProfileContactDetailsChanged`
   con `data` `{ subject: "sub-ana-001", version: 3, status: "complete", givenName: null,
   familyName: null, displayName: "Ana Pérez", contactPhone: "+573009999999" }`.

**When**: `PUT /api/v1/me/profile/contact-details` con `{ "displayName": "Ana Pérez" }`.

**Then**:
4. `200`, `contactPhone: null`, `status: "draft"` (complete → draft), `lockVersion: 4`;
   `ProfileContactDetailsChanged` con `status: "draft"`.

**When**: `PUT /api/v1/me/profile/contact-details` con `{ "displayName": "Ana Pérez", "contactPhone": "+573001234567" }`.

**Then**:
5. `200`, `status: "complete"` (draft → complete), `lockVersion: 5`.

## Direcciones

### FL-ADR-001: añadir direcciones y la predeterminada por tipo

**Given**: `ana` hizo `GET /api/v1/me/profile` (perfil `p1`, `lockVersion: 0`). Canal purgado.

**When**: `addMyAddress` — `POST /api/v1/me/profile/addresses` con `ADDR-CASA`.

**Then**:
1. Status `201`. (La cabecera `Location` no se afirma: el alta no tiene recurso propio que leer, ver
   `decisions.yaml` → `CHK-API-CREATED-NO-READ`.)
2. Cuerpo `UserProfile` con `id: p1`, `contactEmail: "ana@example.com"`, `givenName: "Ana"`,
   `familyName: "Pérez"`, `displayName: "Ana Pérez"`, `contactPhone: null`, `status: "draft"` (falta el
   teléfono), `deactivatedBy: null`, `deactivatedAt: null`, `lockVersion: 1`, y `addresses` con un elemento `{ id: a1, userProfileId: p1,
   label: "Casa", type: "shipping", postal: POSTAL(ADDR-CASA), isDefault: true, createdAt, updatedAt }`.
3. `profileEvents` recibe un `AddressAdded` con `data` `{ subject: "sub-ana-001", version: 1,
   status: "draft", addressId: a1, label: "Casa", type: "shipping", isDefault: true,
   postal: POSTAL(ADDR-CASA), previousDefaultAddressId: null }`.

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-OFI` (sin `isDefault`).

**Then**:
4. `201`; `lockVersion: 2`; `addresses` = `[a1, a2]` en ese orden; `a2` con `isDefault: false` y
   `postal: POSTAL(ADDR-OFI)`; `a1` sigue con `isDefault: true`.
5. `AddressAdded` con `data` `{ subject: "sub-ana-001", version: 2, status: "draft", addressId: a2,
   label: "Oficina", type: "shipping", isDefault: false, postal: POSTAL(ADDR-OFI),
   previousDefaultAddressId: null }`. La primera dirección de un tipo no se marcó sola: `a1` lo es
   porque se pidió.

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-OFI` más `"isDefault": true` y `"label": "Oficina nueva"`.

**Then**:
6. `201`; `lockVersion: 3`; `addresses` = `[a1, a2, a3]`; `a3` con `isDefault: true`; `a1` con
   `isDefault: false` y su `updatedAt` avanza; `a2` sin cambios.
7. `profileEvents` recibe **solo** un `AddressAdded` con `data` `{ subject: "sub-ana-001", version: 3,
   status: "draft", addressId: a3, label: "Oficina nueva", type: "shipping", isDefault: true,
   postal: POSTAL(ADDR-OFI), previousDefaultAddressId: a1 }`; ningún `DefaultAddressChanged`.

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-FACT`.

**Then**:
8. `201`; `lockVersion: 4`; `addresses` = `[a1, a2, a3, a4]`; `a4` `{ label: null, type: "billing",
   isDefault: true }`; `a3` sigue predeterminada de shipping (una por tipo).
9. `AddressAdded` con `data` `{ subject: "sub-ana-001", version: 4, status: "draft", addressId: a4,
   label: null, type: "billing", isDefault: true, postal: POSTAL(ADDR-FACT),
   previousDefaultAddressId: null }`.

**Orden de evaluación** (`addMyAddress`, en el orden de `errors`):
1. Forma del cuerpo → `400 VALIDATION_ERROR`.
2. Clave en curso → `IDEMPOTENCY_KEY_IN_PROGRESS` (`409`).
3. Clave reutilizada con otro cuerpo → `IDEMPOTENCY_KEY_REUSED` (`409`).
4. Aprovisionamiento → `403 SUBJECT_NOT_PROVISIONABLE` / `410 PROFILE_DELETED`.
5. Perfil no desactivado → `PROFILE_DEACTIVATED` (`409`).
6. Menos de 20 direcciones → `ADDRESS_LIMIT_REACHED` (`409`).
7. Prefijo de `adminAreaCode` = `countryCode` → `SUBDIVISION_CODE_COUNTRY_MISMATCH` (`422`).
8. `adminAreaName` presente si hay `adminAreaCode` → `TERRITORY_NAME_MISSING` (`422`).
9. Escritura → `CONCURRENT_MODIFICATION` (`409`).

### FL-ADR-002: reintento con la misma Idempotency-Key

**Given**: `ana` y `beto` hicieron `GET /api/v1/me/profile`. Canal purgado.

**When**: `POST /api/v1/me/profile/addresses` con el token de `ana`, cabecera `Idempotency-Key: k-1` y
`ADDR-CASA`.

**Then**:
1. `201`; cuerpo `B1` con `lockVersion: 1` y `addresses` = `[a1]`; un `AddressAdded` con `addressId: a1`.

**When**: se repite la misma petición (misma clave, mismo cuerpo).

**Then**:
2. `201` con exactamente el cuerpo `B1` (mismo `a1`, `lockVersion: 1`).
3. Ningún evento; `GET /api/v1/me/profile` devuelve una sola dirección.

**When**: `Idempotency-Key: k-1` con `ADDR-OFI`.

**Then**:
4. `409` con `code: "IDEMPOTENCY_KEY_REUSED"`; ningún cambio ni evento.

**When**: `Idempotency-Key: k-2` con `ADDR-CASA` (otra clave, mismo cuerpo).

**Then**:
5. `201`; `lockVersion: 2`; `addresses` = `[a1, a2]` (los duplicados de contenido están permitidos);
   `a2` es predeterminada y `a1` deja de serlo.

**When**: sin cabecera, `ADDR-CASA`.

**Then**:
6. `201`; `lockVersion: 3`; tres direcciones: sin clave no se deduplica.

**When**: `POST /api/v1/me/profile/addresses` con el token de `beto`, `Idempotency-Key: k-1` y `ADDR-CASA`.

**Then**:
7. `201`; el cuerpo es el perfil de `beto` (`subject: "sub-beto-002"`, `lockVersion: 1`, una dirección):
   la clave está acotada por titular y no reproduce la respuesta de `ana`.

### FL-ADR-003: dos altas con la misma clave a la vez

**Given**: `ana` hizo `GET /api/v1/me/profile` (`lockVersion: 0`). Canal purgado.

**When**: dos `POST /api/v1/me/profile/addresses` con `Idempotency-Key: k-race` y `ADDR-CASA` se envían
**a la vez**.

**Then**:
1. O las dos responden `201` con el mismo cuerpo, o una responde `201` y la otra `409` con
   `code: "IDEMPOTENCY_KEY_IN_PROGRESS"`.
2. `GET /api/v1/me/profile` devuelve **exactamente una** dirección y `lockVersion: 1`, sea cual sea el
   desenlace.
3. `profileEvents` recibe exactamente un `AddressAdded`.

### FL-ADR-004: validación de la dirección postal

**Given**: `ana` hizo `GET /api/v1/me/profile` (`lockVersion: 0`) y `POST` de `ADDR-OFI` (dirección `a1`,
`lockVersion: 1`). Canal purgado.

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-OFI` cambiando `postal.adminAreaCode` a `"ES-M"`
y `postal.adminAreaName` a `"Madrid"` (con `countryCode: "CO"`).

**Then**:
1. `422` con `code: "SUBDIVISION_CODE_COUNTRY_MISMATCH"`.

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-OFI` y `postal.adminAreaName: null`
(`adminAreaCode: "CO-DC"`).

**Then**:
2. `422` con `code: "TERRITORY_NAME_MISSING"`.

**When**: `POST` con `adminAreaCode: "ES-M"`, `countryCode: "CO"` y `adminAreaName: null`.

**Then**:
3. `422` con `code: "SUBDIVISION_CODE_COUNTRY_MISMATCH"`: la guarda del prefijo precede a la del nombre.

**When**: `updateMyAddress` — `PUT /api/v1/me/profile/addresses/{a1}` con `ADDR-OFI` y
`adminAreaCode: "ES-M"`; y después otro `PUT` con `ADDR-OFI` tal cual salvo `adminAreaName: null`
(`adminAreaCode: "CO-DC"`).

**Then**:
4. `422 SUBDIVISION_CODE_COUNTRY_MISMATCH` y `422 TERRITORY_NAME_MISSING`, respectivamente.
5. Tras todos los pasos, `GET /api/v1/me/profile` devuelve `lockVersion: 1` y solo `a1`, sin cambios;
   ningún evento.

**Casos borde** (todos `400`, sin cambios):
- Sin `postal.line1`; `line1` de 141 caracteres; sin `postal.localityName`; sin `type`;
  `type: "home"`; `countryCode: "co"`; `adminAreaCode: "CO_ANT"`; `postalCode` de 17 caracteres;
  `localityCode` de 33 caracteres; `label` de 61 caracteres; `label: ""`.

**Casos aceptados** (cada uno sobre un perfil aparte, el de `beto` recién aprovisionado, y **no** sobre
el de `ana` de este flujo, cuyo estado afirma el Then 5):
- Una dirección **sin** `adminAreaCode` ni `adminAreaName` y sin `postalCode`, con `localityCode: null`
  → `201` (solo `line1`, `countryCode` y `localityName` son obligatorios).
- `adminAreaCode: "CO-ZZZ"` (forma válida, subdivisión inexistente) → `201`: solo se valida la forma.

### FL-ADR-005: el límite de 20 direcciones

**Given**: `ana` hizo `GET /api/v1/me/profile` y 20 `POST /api/v1/me/profile/addresses` con `ADDR-OFI`
(`lockVersion: 20`, 20 direcciones).

**When**: `POST /api/v1/me/profile/addresses` con `ADDR-FACT`.

**Then**:
1. `409` con `code: "ADDRESS_LIMIT_REACHED"`; `lockVersion` sigue en 20, ningún evento.

**When**: `POST` con `ADDR-FACT` y `adminAreaCode: "CO-ANT"` (prefijo distinto de `ES`).

**Then**:
2. `409 ADDRESS_LIMIT_REACHED`: el límite precede a las guardas `422`.

**When**: `deactivateProfile` con el token de `admin`; después `POST` con `ADDR-FACT` con el token de `ana`.

**Then**:
3. `409` con `code: "PROFILE_DEACTIVATED"`: el estado precede al límite.

**When**: `reactivateProfile` con el token de `admin`; `removeMyAddress` —
`DELETE /api/v1/me/profile/addresses/{la primera}`; `POST` con `ADDR-FACT`.

**Then**:
4. El `DELETE` responde `200` con 19 direcciones; el `POST`, `201` con 20 y la nueva al final.

### FL-ADR-006: modificar una dirección

**Given**: `ana`: `GET /api/v1/me/profile`; `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`
(`lockVersion: 1`); `POST /api/v1/me/profile/addresses` con `ADDR-CASA` (`a1`, `lockVersion: 2`,
`status: "complete"`); `beto`: `GET /api/v1/me/profile` y `POST` con `ADDR-OFI` (`b1`). Canal purgado.

**When**: `updateMyAddress` — `PUT /api/v1/me/profile/addresses/{a1}` con el token de `ana` y
```json
{ "label": "Casa principal", "type": "shipping",
  "postal": { "line1": "Calle 10 # 43-12", "line2": "Apto 302", "line3": null, "postalCode": "050021",
              "countryCode": "CO", "adminAreaCode": "CO-ANT", "adminAreaName": "Antioquia",
              "localityCode": "05001", "localityName": "Medellín" } }
```

**Then**:
1. `200`; `status: "complete"` (complete → complete), `lockVersion: 3`; `a1` con `label: "Casa principal"`,
   `postal.line2: "Apto 302"`, `isDefault: true` (no se toca), `updatedAt` avanza, `createdAt` igual.
2. `AddressChanged` con `data` `{ subject: "sub-ana-001", version: 3, status: "complete", addressId: a1,
   label: "Casa principal", type: "shipping", isDefault: true, postal: <el enviado> }`.

**When**: se repite el mismo `PUT`.

**Then**:
3. `200` con el mismo cuerpo y `lockVersion: 3`; ningún evento.

**When**: el mismo `PUT` sin `label`.

**Then**:
4. `200`, `a1.label: null`, `lockVersion: 4`, `AddressChanged` con `label: null`.

**When**: el mismo `PUT` sin `label` y con `"type": "billing"`.

**Then**:
5. `200`; `a1` con `type: "billing"` e `isDefault: false` (al cambiar de tipo pierde la marca);
   `status: "draft"` (complete → draft: ya no hay shipping predeterminada); `lockVersion: 5`.
6. `AddressChanged` con `data` `{ subject: "sub-ana-001", version: 5, status: "draft", addressId: a1,
   label: null, type: "billing", isDefault: false, postal: <el enviado> }`.

**When**: `PUT` de nuevo con `"type": "shipping"`.

**Then**:
7. `200`; `isDefault: false` (no la recupera), `status: "draft"` (draft → draft), `lockVersion: 6`.

**When**: `PUT /api/v1/me/profile/addresses/{b1}` con el token de `ana` (dirección de `beto`); y
`PUT /api/v1/me/profile/addresses/{uuid inexistente}`.

**Then**:
8. Las dos responden `404` con `code: "ADDRESS_NOT_FOUND"`; la dirección de `beto` no cambia.

**Orden de evaluación** (`updateMyAddress`): forma (`400`) → aprovisionamiento (`403`/`410`) →
`PROFILE_DEACTIVATED` (`409`) → `ADDRESS_NOT_FOUND` (`404`) → `SUBDIVISION_CODE_COUNTRY_MISMATCH` (`422`)
→ `TERRITORY_NAME_MISSING` (`422`) → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**:
- `PUT /{uuid inexistente}` con `adminAreaCode: "ES-M"` y `countryCode: "CO"` → `404 ADDRESS_NOT_FOUND`
  (precede a la `422`).

### FL-ADR-007: quitar una dirección no promociona otra

**Given**: `ana`: `GET`; `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` (`lockVersion: 1`);
`POST` `ADDR-CASA` (`a1`, predeterminada, `lockVersion: 2`, `complete`); `POST` `ADDR-OFI` (`a2`,
`lockVersion: 3`); `beto`: `GET` y `POST` `ADDR-OFI` (`b1`). Canal purgado.

**When**: `removeMyAddress` — `DELETE /api/v1/me/profile/addresses/{a2}` con el token de `ana`.

**Then**:
1. `200`; cuerpo `UserProfile` con `addresses` = `[a1]`, `status: "complete"` (complete → complete),
   `lockVersion: 4`.
2. `AddressRemoved` con `data` `{ subject: "sub-ana-001", version: 4, status: "complete", addressId: a2,
   type: "shipping", wasDefault: false }`.

**When**: `POST` `ADDR-OFI` (`a3`, `lockVersion: 5`); `DELETE /api/v1/me/profile/addresses/{a1}`.

**Then**:
3. `200`; `addresses` = `[a3]` con `a3.isDefault: false` (no se promociona); `status: "draft"`
   (complete → draft); `lockVersion: 6`.
4. `AddressRemoved` con `data` `{ subject: "sub-ana-001", version: 6, status: "draft", addressId: a1,
   type: "shipping", wasDefault: true }`.

**When**: `DELETE /api/v1/me/profile/addresses/{a3}`.

**Then**:
5. `200`, `addresses: []`, `status: "draft"` (draft → draft), `lockVersion: 7`, `AddressRemoved` con
   `wasDefault: false`.

**When**: `DELETE /api/v1/me/profile/addresses/{a1}` otra vez; y `DELETE /api/v1/me/profile/addresses/{b1}`
con el token de `ana`.

**Then**:
6. Las dos responden `404` con `code: "ADDRESS_NOT_FOUND"`; ningún evento; `lockVersion` sigue en 7 y
   `b1` sigue en el perfil de `beto`.

### FL-ADR-008: cambiar la dirección predeterminada

**Given**: `ana`: `GET`; `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` (`lockVersion: 1`);
`POST` `ADDR-OFI` (`a1`, `lockVersion: 2`, `draft`). `beto`: `GET` (sin teléfono). Canal purgado.

**When**: `setMyDefaultAddress` — `PUT /api/v1/me/profile/addresses/{a1}/default` con el token de `ana`, sin cuerpo.

**Then**:
1. `200`; `a1.isDefault: true`; `status: "complete"` (draft → complete); `lockVersion: 3`.
2. `DefaultAddressChanged` con `data` `{ subject: "sub-ana-001", version: 3, status: "complete",
   addressId: a1, type: "shipping", previousAddressId: null }`.

**When**: `POST` `ADDR-OFI` (`a2`, `lockVersion: 4`); `PUT /api/v1/me/profile/addresses/{a2}/default`.

**Then**:
3. `200`; `a2.isDefault: true`, `a1.isDefault: false`; `status: "complete"` (complete → complete);
   `lockVersion: 5`.
4. `DefaultAddressChanged` con `data` `{ subject: "sub-ana-001", version: 5, status: "complete",
   addressId: a2, type: "shipping", previousAddressId: a1 }`.

**When**: se repite `PUT /api/v1/me/profile/addresses/{a2}/default`.

**Then**:
5. `200` con el mismo cuerpo, `lockVersion: 5`; ningún evento.

**When**: `PUT /api/v1/me/profile/addresses/{uuid inexistente}/default`.

**Then**:
6. `404` con `code: "ADDRESS_NOT_FOUND"`.

**When**: `beto` hace `POST` con `ADDR-FACT` sin `isDefault` (`b1`, `lockVersion: 1`) y
`PUT /api/v1/me/profile/addresses/{b1}/default`.

**Then**:
7. `200`; `b1.isDefault: true`; `status: "draft"` (draft → draft: sin teléfono ni shipping);
   `lockVersion: 2`; `DefaultAddressChanged` con `data` `{ subject: "sub-beto-002", version: 2,
   status: "draft", addressId: b1, type: "billing", previousAddressId: null }`.

### FL-ADR-009: dos mutaciones a la vez sobre el mismo perfil

**Given**: `ana`: `GET`; `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`; `POST` `ADDR-CASA`
(`a1`); `POST` `ADDR-OFI` (`a2`). `lockVersion: 3`. Canal purgado.

**When**: se envían **a la vez** `PUT /api/v1/me/profile/contact-details` con
`{ "displayName": "Ana P.", "contactPhone": "+573001234567" }` y `POST /api/v1/me/profile/addresses` con `ADDR-FACT`.

**Then**:
1. O las dos responden con éxito (`200` y `201`), o una responde con éxito y la otra `409` con
   `code: "CONCURRENT_MODIFICATION"`.
2. `GET /api/v1/me/profile` devuelve `lockVersion` igual a 3 más el número de respuestas con éxito, y
   el canal recibe exactamente un evento por respuesta con éxito, con `version` consecutivas.

**When**: se envían **a la vez** `PUT /api/v1/me/profile/addresses/{a1}/default` y
`PUT /api/v1/me/profile/addresses/{a2}/default`.

**Then**:
3. Cada una responde `200` o `409 CONCURRENT_MODIFICATION`.
4. `GET /api/v1/me/profile` devuelve **exactamente una** dirección shipping con `isDefault: true`.

### FL-ADR-010: un perfil desactivado no admite mutaciones por la API

**Given**: `ana`: `GET`; `POST` `ADDR-CASA` (`a1`, `lockVersion: 1`). `admin` ejecuta
`POST /api/v1/management/profiles/sub-ana-001/deactivate` (`lockVersion: 2`). Canal purgado.

**When**: con el token de `ana`, en orden: `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`;
`POST /api/v1/me/profile/addresses` con `ADDR-OFI`; `PUT /api/v1/me/profile/addresses/{a1}` con `ADDR-OFI`;
`DELETE /api/v1/me/profile/addresses/{a1}`; `PUT /api/v1/me/profile/addresses/{a1}/default`.

**Then**:
1. Las cinco responden `409` con `code: "PROFILE_DEACTIVATED"`.
2. Ningún evento; `GET /api/v1/me/profile` responde `200` con `status: "deactivated"`, `lockVersion: 2`
   y `a1` intacta (un perfil desactivado conserva sus datos y se puede leer).

**Casos borde**:
- `PUT /api/v1/me/profile/addresses/{uuid inexistente}` → `409 PROFILE_DEACTIVATED` (el estado precede
  a `ADDRESS_NOT_FOUND`).
- `PUT /api/v1/me/profile/contact-details` con `contactPhone: "123"` → `400` (la forma precede al estado).

## Borrado y lápida

### FL-DEL-001: el titular borra su perfil

**Given**: `ana`: `GET` (`p1`); `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`; `POST`
`ADDR-CASA` (`lockVersion: 2`). Canal purgado.

**When**: `deleteMyProfile` — `DELETE /api/v1/me/profile` con el token de `ana`, sin cuerpo.

**Then**:
1. Status `204`, sin cuerpo.
2. `profileEvents` recibe un `ProfileDeleted` con `data` `{ subject: "sub-ana-001", version: 3,
   deletedAt: <instante>, reason: "self-service" }`, sin ningún dato personal.
3. `GET /api/v1/me/profile` con el token de `ana` → `410 PROFILE_DELETED`.
4. `getProfile` — `GET /api/v1/management/profiles/sub-ana-001` con el token de `admin` →
   `410 PROFILE_DELETED`.
5. `listProfiles` con el token de `admin` no incluye a `sub-ana-001`.

**When**: se repite `DELETE /api/v1/me/profile` con el token de `ana`.

**Then**:
6. `204`; ningún evento.

**When**: `deleteProfile` — `DELETE /api/v1/management/profiles/sub-ana-001` con el token de `admin`.

**Then**:
7. `200` con la lápida original `{ subject: "sub-ana-001", deletedAt: <el del evento>,
   deletedBy: "sub-ana-001", reason: "self-service" }`; no se reescribe y no hay evento.

### FL-DEL-002: un perfil desactivado también se puede borrar

**Given**: `ana`: `GET` (`lockVersion: 0`); `admin` desactiva el perfil (`lockVersion: 1`). Canal purgado.

**When**: `DELETE /api/v1/me/profile` con el token de `ana`.

**Then**:
1. `204`; `ProfileDeleted` con `data` `{ subject: "sub-ana-001", version: 2, deletedAt: <instante>,
   reason: "self-service" }`.

### FL-DEL-003: borrar sin perfil ni lápida

**Given**: no existe perfil ni lápida para `sub-eva-005` (`eva` nunca llamó al servicio).

**When**: `DELETE /api/v1/me/profile` con el token de `eva`.

**Then**:
1. `404` con `code: "PROFILE_NOT_FOUND"`; no se crea perfil ni lápida.

**When**: `DELETE /api/v1/management/profiles/sub-eva-005` con el token de `admin`.

**Then**:
2. `404` con `code: "PROFILE_NOT_FOUND"`.
3. `GET /api/v1/me/profile` con el token de `eva` → `200` con un perfil nuevo en `draft` (no había lápida).

### FL-DEL-004: borrado desde back-office y levantamiento de la lápida

**Given**: `ana`: `GET` (`p1`); `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`
(`lockVersion: 1`). Canal purgado.

**When**: `deleteProfile` — `DELETE /api/v1/management/profiles/sub-ana-001` con el token de `admin`.

**Then**:
1. `200` con `{ subject: "sub-ana-001", deletedAt: <instante>, deletedBy: "sub-op-admin",
   reason: "back-office" }` y nada más.
2. `ProfileDeleted` con `data` `{ subject: "sub-ana-001", version: 2, deletedAt: <el de la lápida>,
   reason: "back-office" }`.

**When**: se repite el mismo `DELETE` con el token de `admin`; y `DELETE /api/v1/me/profile` con el de `ana`.

**Then**:
3. El primero, `200` con la misma lápida (mismo `deletedAt`); el segundo, `204`. Ningún evento.

**When**: `reinstateSubject` — `POST /api/v1/management/deleted-subjects/sub-ana-001/reinstate` con el
token de `admin`, sin cuerpo.

**Then**:
4. `204`, sin cuerpo; ningún evento.
5. `GET /api/v1/management/profiles/sub-ana-001` → `404 PROFILE_NOT_FOUND` (no se restaura nada).

**When**: `GET /api/v1/me/profile` con el token de `ana`.

**Then**:
6. `200` con un perfil **nuevo**: `id` distinto de `p1`, `lockVersion: 0`, `status: "draft"`,
   `givenName: "Ana"`, `contactPhone: null`, `addresses: []`.
7. `ProfileProvisioned` con `version: 0`: es una encarnación nueva (`ProfileDeleted` cerró la anterior).

**When**: `POST /api/v1/management/deleted-subjects/sub-ana-001/reinstate` otra vez.

**Then**:
8. `404` con `code: "SUBJECT_NOT_DELETED"`.

### FL-DEL-005: dos borrados del mismo perfil a la vez

**Given**: `ana`: `GET` (`lockVersion: 0`). Canal purgado.

**When**: se envían **a la vez** `DELETE /api/v1/me/profile` (token de `ana`) y
`DELETE /api/v1/management/profiles/sub-ana-001` (token de `admin`).

**Then**:
1. El de `ana` responde `204`; el de `admin`, `200` con la lápida que quedó guardada. Ninguno responde
   `409`: el perdedor relee y responde como una repetición.
2. `profileEvents` recibe **exactamente un** `ProfileDeleted`, y su `reason` coincide con la lápida
   devuelta (`self-service` con `deletedBy: "sub-ana-001"`, o `back-office` con `deletedBy: "sub-op-admin"`).

**Notas de determinación**: `CONCURRENT_MODIFICATION` en un borrado solo aparece si la otra escritura
concurrente no es un borrado (p. ej. un `POST` de dirección): `DELETE /api/v1/me/profile` a la vez que
`POST /api/v1/me/profile/addresses` → el borrado responde `204` o `409 CONCURRENT_MODIFICATION`, y si
responde `204` el perfil termina borrado.

## Back-office

### FL-BO-001: listado paginado, filtrado y buscado

**Given**: en este orden, `GET /api/v1/me/profile` con los tokens de `ana`, `beto`, `cleo` y `dora`,
dejando pasar al menos 1 ms entre uno y otro, de modo que sus `createdAt` son estrictamente crecientes en
ese orden. `admin` desactiva `sub-ana-001`.

**When**: `listProfiles` — `GET /api/v1/management/profiles` con el token de `admin`.

**Then**:
1. `200`; `items` en orden `dora`, `cleo`, `beto`, `ana` (createdAt desc), cada uno con la proyección
   de `listProfiles` y **sin** `addresses`; `page: 0`, `size: 25`, `totalElements: 4`, `totalPages: 1`.
2. `ana` aparece con `status: "deactivated"`; los demás con `"draft"`.

**When**: `?size=2`; `?size=2&page=1`; `?size=2&page=5`; `?size=500`.

**Then**:
3. `?size=2` → `[dora, cleo]`, `totalElements: 4`, `totalPages: 2`.
4. `?size=2&page=1` → `[beto, ana]`.
5. `?size=2&page=5` → `items: []`, `page: 5`, `size: 2`, `totalElements: 4`, `totalPages: 2`.
6. `?size=500` → `size: 100` y los 4 perfiles.

**When**: búsquedas y filtros con el token de `admin`.

**Then**:
7. `?search=ANGEL` → `[dora]` (prefijo de `displayName` "Ángela Dora", sin mayúsculas ni acentos).
8. `?search=dora@` → `[dora]` (prefijo de `contactEmail`).
9. `?search=ana` → `[ana]`; `?search=pérez` → página vacía canónica (`items: []`, `totalElements: 0`,
   `totalPages: 0`): es subcadena, no prefijo.
10. `?status=deactivated` → `[ana]`; `?status=complete` → página vacía canónica.
11. `?status=draft&search=ana` → página vacía (AND); `?status=deactivated&search=ana` → `[ana]`.

**When**: `admin` ejecuta `DELETE /api/v1/management/profiles/sub-beto-002`; después `GET /api/v1/management/profiles`.

**Then**:
12. `items` = `[dora, cleo, ana]`, `totalElements: 3`: los borrados no se listan.

**Casos borde**:
- `?status=foo` → `400`. `?search=` (vacío) → `400`.

### FL-BO-002: ficha de un perfil

**Given**: `ana`: `GET` (`p1`); `POST` `ADDR-CASA` (`a1`); `POST` `ADDR-FACT` (`a2`).

**When**: `getProfile` — `GET /api/v1/management/profiles/sub-ana-001` con el token de `support`.

**Then**:
1. `200`; cuerpo `UserProfile` de `p1` con `lockVersion: 2` y `addresses` = `[a1, a2]` en orden de alta,
   cada una con su proyección `Address` completa (`userProfileId: p1`).
2. Es idéntico a lo que devuelve `GET /api/v1/me/profile` a `ana` en ese momento.

**When**: `GET /api/v1/management/profiles/sub-nadie-999`.

**Then**:
3. `404` con `code: "PROFILE_NOT_FOUND"`.

### FL-BO-003: desactivar y reactivar

**Given**: `ana`: `GET`; `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`; `POST` `ADDR-CASA`
(`status: "complete"`, `lockVersion: 2`). `beto`: `GET` (`draft`, `lockVersion: 0`). Canal purgado.

**When**: `deactivateProfile` — `POST /api/v1/management/profiles/sub-ana-001/deactivate` con el token
de `support`, sin cuerpo.

**Then**:
1. `200`; cuerpo `UserProfile` con `status: "deactivated"`, `deactivatedBy: "sub-op-support"`,
   `deactivatedAt` instante válido, `lockVersion: 3` y los datos intactos.
2. `ProfileDeactivated` con `data` `{ subject: "sub-ana-001", version: 3, status: "deactivated",
   deactivatedBy: "sub-op-support", deactivatedAt: <el del perfil> }`.

**When**: `GET /api/v1/me/profile` con el token de `ana` con `email: "ana.perez@example.org"`; después
se repite la desactivación.

**Then**:
3. El `GET` responde `200` con `contactEmail: "ana.perez@example.org"`, `status: "deactivated"`,
   `lockVersion: 4`, y `deactivatedBy: "sub-op-support"` y `deactivatedAt` sin cambios: el refresco del
   aprovisionamiento no pisa quién desactivó.
4. La desactivación repetida responde `409` con `code: "PROFILE_ALREADY_DEACTIVATED"` y no publica evento.

**When**: `reactivateProfile` — `POST /api/v1/management/profiles/sub-ana-001/reactivate` con el token de `support`.

**Then**:
5. `200`; `status: "complete"` (deactivated → complete: se recalcula), `deactivatedBy: null`,
   `deactivatedAt: null`, `lockVersion: 5`.
6. `ProfileReactivated` con `data` `{ subject: "sub-ana-001", version: 5, status: "complete" }`.

**When**: se repite la reactivación.

**Then**:
7. `409` con `code: "PROFILE_NOT_DEACTIVATED"`.

**When**: desactivar y reactivar `sub-beto-002`.

**Then**:
8. La desactivación: `200`, `status: "deactivated"` (draft → deactivated), `deactivatedBy: "sub-op-support"`, `lockVersion: 1`. La
   reactivación: `200`, `status: "draft"` (deactivated → draft), `lockVersion: 2`, `ProfileReactivated`
   con `status: "draft"`.

**When**: `admin` borra `sub-beto-002`; después `deactivate` y `reactivate` de `sub-beto-002`; y
`deactivate`/`reactivate` de `sub-nadie-999`.

**Then**:
9. Sobre `sub-beto-002`, las dos responden `410 PROFILE_DELETED`; sobre `sub-nadie-999`,
   `404 PROFILE_NOT_FOUND`.

**Notas de determinación**: un operador puede actuar sobre su propio subject: `admin` desactivando
`sub-op-admin` (tras su propio `GET /api/v1/me/profile`) responde `200`.

### FL-BO-004: autenticación y permisos de back-office y de /me

**Given**: `ana`: `GET /api/v1/me/profile`.

**When**: cada operación expuesta se llama **sin** cabecera `Authorization`.

**Then**:
1. `getMyProfile`, `updateMyContactDetails`, `addMyAddress`, `updateMyAddress`, `removeMyAddress`,
   `setMyDefaultAddress`, `deleteMyProfile`, `listProfiles`, `getProfile`, `deactivateProfile`,
   `reactivateProfile`, `deleteProfile`, `reinstateSubject`, `resolveContactForServices`,
   `resolveDeliveryProfileForServices`, `resolveContactsBatchForServices` y
   `resolveDeliveryProfilesBatchForServices` responden `401`.

**When**: con el token de `ana` (sin roles): `GET /api/v1/management/profiles`,
`GET /api/v1/management/profiles/sub-ana-001`, `POST …/sub-ana-001/deactivate`,
`POST …/sub-ana-001/reactivate`, `DELETE /api/v1/management/profiles/sub-ana-001` y
`POST /api/v1/management/deleted-subjects/sub-ana-001/reinstate`.

**Then**:
2. Las seis responden `403`; el perfil de `ana` no cambia.

**When**: con el token de `support` (rol `profile-support`): `DELETE /api/v1/management/profiles/sub-ana-001`
y `POST /api/v1/management/deleted-subjects/sub-ana-001/reinstate`.

**Then**:
3. Las dos responden `403` (`profile:delete` es solo del rol `profile-admin`); `GET
   /api/v1/management/profiles` con el mismo token responde `200`.

**When**: `GET /api/v1/management/profiles` con la credencial de máquina del cliente `order-service`.

**Then**:
4. `403`.

## Superficie servidor-a-servidor

### FL-M2M-001: resolver el contacto de un subject

**Given**: `ana`: `GET`; `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` (`lockVersion: 1`);
`POST` `ADDR-CASA` (`lockVersion: 2`, `complete`).

**When**: `resolveContactForServices` — `GET /api/v1/services/profiles/sub-ana-001/contact` con la
credencial de máquina del cliente `notification-service`.

**Then**:
1. `200` con exactamente `{ subject: "sub-ana-001", displayName: "Ana Pérez", givenName: "Ana",
   familyName: "Pérez", contactEmail: "ana@example.com", contactPhone: "+573001234567",
   status: "complete", version: 2, updatedAt: <el del perfil> }`; ninguna dirección ni campo adicional.

**When**: la misma llamada con la credencial de máquina del cliente `order-service`; con la del cliente
`notification-service` emitida para otra audiencia; con el token de `ana`.

**Then**:
2. Las tres responden `403`.

**When**: `GET /api/v1/services/profiles/sub-nadie-999/contact` con la credencial de `notification-service`.

**Then**:
3. `404` con `code: "PROFILE_NOT_FOUND"`. No se aprovisiona nada.

**When**: `admin` desactiva `sub-ana-001`; misma lectura de contacto.

**Then**:
4. `200` con `status: "deactivated"` y `version: 3`: se resuelve en cualquier estado.

**When**: `admin` borra `sub-ana-001`; misma lectura.

**Then**:
5. `410` con `code: "PROFILE_DELETED"`.

### FL-M2M-002: resolver la entrega de un subject

**Given**: `ana`: `GET` (con teléfono a `null`, sin direcciones); `beto`: `GET`; `dani`: `GET` y
`PUT /api/v1/me/profile/contact-details` con `{ "contactPhone": "+573005550000" }`.

**When**: `resolveDeliveryProfileForServices` — `GET /api/v1/services/profiles/sub-ana-001/delivery` con
la credencial de máquina del cliente `order-service`.

**Then**:
1. `409` con `code: "PROFILE_INCOMPLETE"` y `details` = `["contactPhone", "defaultShippingAddress"]`.

**When**: la misma lectura para `sub-dani-006`.

**Then**:
2. `409 PROFILE_INCOMPLETE` con `details` = `["contactEmail", "displayName", "defaultShippingAddress"]`
   (`dani` no tiene email y el bloque de contacto se reemplazó sin `displayName`).

**When**: `ana` hace `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` y `POST` con `ADDR-CASA`
(`a1`, `lockVersion: 2`); misma lectura para `sub-ana-001`.

**Then**:
3. `200` con exactamente `{ subject: "sub-ana-001", displayName: "Ana Pérez", givenName: "Ana",
   familyName: "Pérez", contactEmail: "ana@example.com", contactPhone: "+573001234567",
   status: "complete", version: 2, updatedAt: <el del perfil>, shippingAddress: POSTAL(ADDR-CASA) }`;
   sin `id`, `label` ni `isDefault` de la dirección.

**When**: la misma lectura con la credencial de máquina del cliente `notification-service`.

**Then**:
4. `403`: ese cliente no puede obtener una dirección ni pidiéndola.

**When**: `admin` desactiva `sub-ana-001`; misma lectura con `order-service`.

**Then**:
5. `200` con `status: "deactivated"`, `version: 3` y la misma `shippingAddress`: la entregabilidad se
   evalúa sobre los datos.

**When**: lectura de `sub-nadie-999`; `admin` borra `sub-beto-002` y se lee `sub-beto-002`.

**Then**:
6. `404 PROFILE_NOT_FOUND` y `410 PROFILE_DELETED`, respectivamente.

### FL-M2M-003: lote de contacto

**Given**: `ana` y `beto`: `GET /api/v1/me/profile`. `cleo`: `GET` y `admin` borra `sub-cleo-003`.
`admin` desactiva `sub-beto-002`. No existe `sub-nadie-999`.

**When**: `resolveContactsBatchForServices` — `POST /api/v1/services/profiles/contact/batch` con la
credencial de máquina del cliente `notification-service` y
```json
{ "subjects": ["sub-beto-002", "sub-nadie-999", "sub-ana-001", "sub-beto-002", "sub-cleo-003"] }
```

**Then**:
1. `200` con exactamente `{ profiles, unresolved }`.
2. `profiles` = `[beto, ana]` en ese orden (primera aparición), cada uno con la proyección de contacto
   completa; `beto` con `status: "deactivated"`. `beto` aparece una sola vez.
3. `unresolved` = `[{ subject: "sub-nadie-999", reason: "not-found" },
   { subject: "sub-cleo-003", reason: "deleted" }]`.

**When**: con 101 subjects distintos; con 101 veces `"sub-ana-001"`.

**Then**:
4. Las dos responden `422` con `code: "TOO_MANY_SUBJECTS"` (el tope se cuenta antes de deduplicar).

**When**: con `{ "subjects": [] }`; sin `subjects`; con `["sub-ana-001", ""]`; con un subject de 256 caracteres.

**Then**:
5. Las cuatro responden `400`: el lote entero se rechaza.

**When**: la llamada válida del paso 1 con la credencial de máquina del cliente `order-service`.

**Then**:
6. `403`.

**Casos borde**:
- 100 subjects (todos inexistentes) → `200` con `profiles: []` y 100 entradas `not-found` en orden.

### FL-M2M-004: lote de entrega

**Given**: `ana`: `GET`, `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`, `POST` `ADDR-CASA`
(`complete`). `beto`: `GET` (incompleto). `cleo`: `GET` y `admin` la borra.

**When**: `resolveDeliveryProfilesBatchForServices` — `POST /api/v1/services/profiles/delivery/batch`
con la credencial de máquina del cliente `order-service` y
`{ "subjects": ["sub-cleo-003", "sub-beto-002", "sub-ana-001", "sub-nadie-999"] }`.

**Then**:
1. `200`; `profiles` = `[ana]` con la proyección de entrega completa (`shippingAddress: POSTAL(ADDR-CASA)`).
2. `unresolved` = `[{ subject: "sub-cleo-003", reason: "deleted" },
   { subject: "sub-beto-002", reason: "incomplete" }, { subject: "sub-nadie-999", reason: "not-found" }]`.

**When**: 101 subjects; la llamada del paso 1 con la credencial de `notification-service`.

**Then**:
3. `422 TOO_MANY_SUBJECTS` y `403`, respectivamente.

### FL-M2M-005: la caché de contacto se invalida con cada uno de los diez eventos

**Given**: no existe perfil para `sub-ana-001`. Todas las lecturas de este flujo son
`GET /api/v1/services/profiles/sub-ana-001/contact` con la credencial de `notification-service`, y
todas ocurren dentro de los 60 s del TTL.

**When / Then** (cada paso: mutación y lectura inmediata):
1. Lectura → `404 PROFILE_NOT_FOUND`.
2. `ana` hace `GET /api/v1/me/profile` (`ProfileProvisioned`) → lectura `200`, `version: 0`, `status: "draft"`.
3. `GET /api/v1/me/profile` con `email: "ana.perez@example.org"` (`ProfileContactEmailRefreshed`) →
   lectura con `contactEmail: "ana.perez@example.org"`, `version: 1`. **Desde aquí, toda petición de
   `ana` de este flujo lleva `email: "ana.perez@example.org"`**, para no provocar otro refresco.
4. `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` (`ProfileContactDetailsChanged`) →
   `contactPhone: "+573001234567"`, `version: 2`.
5. `POST` `ADDR-OFI` (`AddressAdded`, `a1`) → `version: 3`, `status: "draft"`.
6. `PUT /api/v1/me/profile/addresses/{a1}` con `ADDR-CASA` sin `isDefault` (`AddressChanged`) → `version: 4`.
7. `PUT /api/v1/me/profile/addresses/{a1}/default` (`DefaultAddressChanged`) → `version: 5`,
   `status: "complete"`.
8. `POST` `ADDR-FACT` (`a2`, `version: 6`) y `DELETE /api/v1/me/profile/addresses/{a2}`
   (`AddressRemoved`) → `version: 7`.
9. `admin` desactiva (`ProfileDeactivated`) → `version: 8`, `status: "deactivated"`.
10. `admin` reactiva (`ProfileReactivated`) → `version: 9`, `status: "complete"`.
11. `ana` hace `DELETE /api/v1/me/profile` (`ProfileDeleted`) → `410 PROFILE_DELETED`.

### FL-M2M-006: la caché de entrega se invalida con cada uno de los diez eventos

**Given**: no existe perfil para `sub-ana-001`. Todas las lecturas son
`GET /api/v1/services/profiles/sub-ana-001/delivery` con la credencial de `order-service`, dentro del TTL.

**When / Then**:
1. Lectura → `404 PROFILE_NOT_FOUND`.
2. `ana` hace `GET /api/v1/me/profile` (`ProfileProvisioned`) → `409 PROFILE_INCOMPLETE` con `details`
   `["contactPhone", "defaultShippingAddress"]`.
3. `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` (`ProfileContactDetailsChanged`) →
   `409` con `details` `["defaultShippingAddress"]`.
4. `POST` `ADDR-CASA` (`AddressAdded`, `a1`) → `200`, `version: 2`, `shippingAddress: POSTAL(ADDR-CASA)`.
5. `GET /api/v1/me/profile` con `email: "ana.perez@example.org"` (`ProfileContactEmailRefreshed`) →
   `200`, `contactEmail: "ana.perez@example.org"`, `version: 3`. **Desde aquí, toda petición de `ana`
   de este flujo lleva `email: "ana.perez@example.org"`**, para no provocar otro refresco.
6. `PUT /api/v1/me/profile/addresses/{a1}` con `ADDR-OFI` (`AddressChanged`) → `shippingAddress:
   POSTAL(ADDR-OFI)`, `version: 4`.
7. `POST` `ADDR-CASA` sin `isDefault` (`a2`, `version: 5`); `PUT /api/v1/me/profile/addresses/{a2}/default`
   (`DefaultAddressChanged`) → `shippingAddress: POSTAL(ADDR-CASA)`, `version: 6`.
8. `DELETE /api/v1/me/profile/addresses/{a2}` (`AddressRemoved`) → `409` con `details`
   `["defaultShippingAddress"]`.
9. `PUT /api/v1/me/profile/addresses/{a1}/default` (`version: 8`); `admin` desactiva
   (`ProfileDeactivated`) → `200`, `status: "deactivated"`, `version: 9`.
10. `admin` reactiva (`ProfileReactivated`) → `200`, `status: "complete"`, `version: 10`.
11. `DELETE /api/v1/me/profile` (`ProfileDeleted`) → `410 PROFILE_DELETED`.

## Mecanismos

### FL-OBX-001: el evento sobrevive a un canal indisponible

**Given**: `ana` hizo `GET /api/v1/me/profile` y su `ProfileProvisioned` ya se recibió; el canal
`profileEvents` está vacío y el canal de eventos **indisponible**.

**When**: `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA`.

**Then**:
1. `200` con el cuerpo completo (`lockVersion: 1`): la indisponibilidad del canal no llega al cliente.
2. `GET /api/v1/me/profile` devuelve `contactPhone: "+573001234567"` y `lockVersion: 1`.
3. El canal `profileEvents` no ha recibido ningún mensaje todavía.

**When**: el canal vuelve a estar disponible.

**Then**:
4. En ≤ 10 s el canal recibe **exactamente un** `ProfileContactDetailsChanged` con `data`
   `{ subject: "sub-ana-001", version: 1, status: "draft", givenName: "Ana", familyName: "Pérez",
   displayName: "Ana Pérez", contactPhone: "+573001234567" }` y el `correlationId` de la petición.
5. El servidor no informa de ningún evento abandonado.

### FL-OBX-002: el evento que el relay abandona no se pierde en silencio

**Given**: `ana` hizo `GET /api/v1/me/profile` y su `ProfileProvisioned` ya se recibió; el canal está
indisponible y `PUT /api/v1/me/profile/contact-details` con `CONTACT-ANA` respondió `200`, con su evento
pendiente de salir.

**When**: se agota el presupuesto de reintentos de ese evento.

**Then**:
1. El servidor informa de **un** evento abandonado.
2. Restablecido el canal, ese `ProfileContactDetailsChanged` **no** se publica.
3. El canal no recibe ninguna otra cosa.

### FL-CORS-001: preflight desde el navegador

**When**: `OPTIONS /api/v1/me/profile/addresses` **sin credencial**, con `Origin` de un origen web
permitido, `Access-Control-Request-Method: POST` y
`Access-Control-Request-Headers: Authorization, Content-Type, Idempotency-Key`.

**Then**:
1. Respuesta `2xx`.
2. `Access-Control-Allow-Methods` incluye `POST`; `Access-Control-Allow-Headers` incluye
   `Authorization`, `Content-Type` e `Idempotency-Key`.
3. `Access-Control-Max-Age: 3600`, y **no** hay `Access-Control-Allow-Credentials: true`.

### FL-CORS-002: una petición normal cross-origin

**Given**: `ana` hizo `GET /api/v1/me/profile`.

**When**: `GET /api/v1/me/profile` con el token de `ana` y `Origin` del mismo origen web permitido.

**Then**:
1. `200` con el cuerpo del perfil.
2. `Access-Control-Allow-Origin` es ese origen y `Access-Control-Expose-Headers` incluye `X-Correlation-Id`.
3. La respuesta trae la cabecera `X-Correlation-Id`.

## Lo que no tiene escenario

- **Retención de la caché M2M** (`resolveContactForServices`, `resolveDeliveryProfileForServices`):
  `invalidatedBy` incluye los diez eventos, que son **todas** las vías de mutación del perfil, así que no
  existe un cambio por una vía fuera de `invalidatedBy` contra el que afirmar que se sirve el valor viejo.
  La única escritura sin evento (`reinstateSubject`) cambia un error (`410` → `404`), y si un error se
  cachea es decisión del generador. Una caché ausente pasaría estos escenarios: se verifica de forma
  estática en la generación.
