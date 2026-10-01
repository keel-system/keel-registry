# user-profile — Documento de diseño

> specs/user-profile v0.1.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`user-profile` guarda el **perfil de negocio** de cada persona autenticada: datos de contacto y direcciones
postales. Lo indexa por el `sub` del token y no duplica nada de lo que pertenece al servidor de identidad.
La tesis del diseño es que **ningún dato tiene dos dueños**:

- La identidad (credenciales, MFA, sesiones, roles) es del servidor OIDC.
- El perfil es de este servicio.
- Lo único que comparten es el `sub` (`SubjectIdentifier`), un identificador opaco, inmutable y clave de
  negocio. El email nunca es clave, porque puede cambiar de dueño.

El servicio nunca llama al servidor de identidad (valida la firma del token en local) y no se suscribe a
sus eventos.

Sirve a tres públicos, con operaciones separadas:

- **El titular**, por `/me`: ve su perfil y edita su contacto y sus direcciones.
- **El back-office**, por `/management`, indexado por subject: lista, consulta, desactiva, borra y levanta
  lápidas.
- **Otros servidores**, por `/services`: resuelven el contacto (`notification-service`) o los datos de
  entrega (`order-service`) de un subject o de un lote.

**Fuera de alcance:**

- El alta administrativa de identidades. Sería otra variante de la familia.
- La validación de existencia de códigos territoriales. Le corresponde al llamante.
- Cualquier retención o purga automática.

El perfil se crea por **aprovisionamiento just-in-time**: lo crea la primera petición autenticada de una
persona a `/me`. No hay alta manual.

## 2. Modelo de dominio

**Value types:**

| Tipo | Significado |
|---|---|
| `SubjectIdentifier` | `sub` del token: opaco, de 1 a 255 caracteres, se compara byte a byte |
| `EmailAddress` | Forma simple `algo@algo.algo`, como máximo 254 |
| `PhoneNumber` | E.164 (`+` y de 2 a 15 dígitos) |
| `PersonName` | Nombre de persona, de 1 a 100 |
| `CountryCode` | ISO 3166-1 alpha-2 |
| `PostalCode` | De 1 a 16, **sin patrón**: cada país tiene su formato y algunos no tienen código postal |
| `SubdivisionCode` | ISO 3166-2 (`CO-ANT`, `ES-M`); el prefijo es el país |
| `TerritoryCode` | Segundo nivel territorial, opaco, de 1 a 32; se interpreta según el país |
| `TerritoryName` / `AddressLine` | Nombre territorial (hasta 120) y línea de dirección (hasta 140) |

**Enums:**

- `ProfileStatus` (`draft`, `complete`, `deactivated`): `draft` es un estado normal, no un error.
- `AddressType` (`shipping`, `billing`).
- `DeletionReason` (`self-service`, `back-office`): cerrado a propósito, para que la lápida no acumule PII
  en texto libre.
- `UnresolvedReason` (`not-found`, `deleted`, `incomplete`).

**Value objects:**

- `PostalAddress`, la unidad que viaja en respuestas, eventos y M2M:
  - `line1` obligatoria, `line2` y `line3` opcionales.
  - `postalCode` opcional y `countryCode` obligatorio.
  - Dos niveles territoriales con **nombres genéricos**: `adminAreaCode`/`adminAreaName` y
    `localityCode`/`localityName`. Solo `localityName` es obligatorio.
  - El nombre se **denormaliza** al guardar y no se vuelve a resolver nunca.
- Las proyecciones M2M `ContactDetails`, `DeliveryProfile` y `UnresolvedSubject`.

**Entidades y agregados:**

- **`UserProfile`** (raíz):
  - Campos: `id` (generated), `subject` (único), `contactEmail`, `givenName`, `familyName`,
    `displayName`, `contactPhone`, `status`, `deactivatedBy`, `deactivatedAt`, `lockVersion` (generated)
    y `createdAt`/`updatedAt` (generated).
  - `contactEmail` es una réplica de solo lectura del claim `email`: ninguna operación de la API lo cambia.
  - Contiene `addresses` (como máximo 20, en orden de alta).
- **`Address`** (dentro del agregado `UserProfile`): `id`, `label` (hasta 60, opcional), `type`, `postal`,
  `isDefault` y timestamps.
- **`DeletedSubject`** (agregado propio): la **lápida**.
  - Campos: `subject` (id), `deletedAt`, `deletedBy` y `reason`. No guarda ningún dato personal.
  - Es permanente hasta que un acto administrativo la levanta.

**Ciclo de vida de `UserProfile.status`:**

```
draft ⇄ complete          (se recalcula en cada mutación)
draft / complete → deactivated     (deactivateProfile)
deactivated → draft / complete     (reactivateProfile, recalculando)
draft→draft, complete→complete, deactivated→deactivated   (mutación que deja el estado igual)
```

El perfil está `complete` si y solo si tiene `contactEmail`, `contactPhone`, `displayName` y una dirección
`shipping` con `isDefault`. Si falta cualquiera de las cuatro, está `draft`. `deactivated` es un estado
de back-office que se superpone a la completitud.

## 3. Invariantes y reglas clave

- **Versión:** `lockVersion` empieza en 0 y sube exactamente 1 con cada mutación confirmada del agregado,
  también cuando la mutación es de una dirección. Es el `version` de todos los eventos. Una mutación cuyo
  resultado es igual a lo guardado **no es mutación**: no sube versión, no toca `updatedAt` y no publica
  evento.
- **Direcciones:**
  - Como mucho una dirección predeterminada **por tipo**; marcar una desmarca la anterior.
  - Al cambiar de tipo, una dirección pierde `isDefault`.
  - Al quitar la predeterminada no se promociona otra.
  - La primera dirección de un tipo no se marca predeterminada sola.
  - Si viene `adminAreaCode`, viene `adminAreaName` y su prefijo coincide con `countryCode`. Solo se valida
    la forma, nunca la existencia.
- **Perfil desactivado:** conserva todos sus datos y no admite mutaciones por la API. El refresco del
  email por aprovisionamiento sí ocurre, y no pisa `deactivatedBy`/`deactivatedAt`.
- **Perfil y lápida:** **nunca coexisten** para el mismo subject.
  - La creación del perfil y la escritura de la lápida se serializan.
  - Mientras haya lápida, el aprovisionamiento responde `410 PROFILE_DELETED`.
  - Un borrado repetido no reescribe la lápida.
- **Aprovisionamiento:**
  - Crea el perfil en `draft` con los claims que traiga el token. Un claim con forma inválida se trata como
    ausente.
  - Después, solo refresca `contactEmail` cuando el claim cambia (comparación exacta). Ningún otro claim
    refresca nada.
  - Confirma en **su propia transacción**, antes de la operación que lo dispara.
  - Un token de máquina (flujo client-credentials) no se aprovisiona nunca.
- **Errores:** se evalúan en el orden declarado. La forma (`400`) va antes que todo, también antes que la
  idempotencia.

## 4. Qué hace

**Transversal:** `provisionProfileFromIdentity` (interna). La ejecutan todas las operaciones de `/me` salvo
`deleteMyProfile`. Su idempotencia es `payload-field` sobre `callerSubject`: la unicidad de `subject` es
la guarda, y si dos primeros accesos chocan, el perdedor relee y devuelve el ganador.

**Titular (`/me`)**. El subject sale siempre del token (`callerIdentity`) y nunca del cuerpo. Toda
mutación devuelve el perfil completo, porque el status recalculado no es predecible.

| Operación | Endpoint | Notas |
|---|---|---|
| `getMyProfile` | `GET /me/profile` | Un usuario nuevo recibe su perfil en draft, nunca un 404 |
| `updateMyContactDetails` | `PUT /me/profile/contact-details` | Reemplaza el bloque completo; el email queda fuera |
| `addMyAddress` | `POST /me/profile/addresses` → 201 | `Idempotency-Key` opcional, 24 h, acotada al titular; la nueva es la última de `addresses` |
| `updateMyAddress` | `PUT /me/profile/addresses/{addressId}` | Reemplazo completo; no toca `isDefault` salvo al cambiar de tipo |
| `removeMyAddress` | `DELETE /me/profile/addresses/{addressId}` → 200 | Devuelve el perfil (el status puede volver a draft) |
| `setMyDefaultAddress` | `PUT /me/profile/addresses/{addressId}/default` | Idempotente por naturaleza |
| `deleteMyProfile` | `DELETE /me/profile` → 204 | Borra y deja la lápida self-service; repetir da el mismo 204; sin perfil ni lápida, 404 |

**Back-office (`/management`):**

| Operación | Endpoint | Notas |
|---|---|---|
| `listProfiles` | `GET /management/profiles` | `status` AND `search` (prefijo sin mayúsculas ni acentos en email o nombre visible); `createdAt` desc; sin direcciones; paginado 25/100 |
| `getProfile` | `GET /management/profiles/{subject}` | Con direcciones |
| `deactivateProfile` | `POST …/{subject}/deactivate` | Estampa `deactivatedBy`/`deactivatedAt`; irrepetible (`PROFILE_ALREADY_DEACTIVATED`) |
| `reactivateProfile` | `POST …/{subject}/reactivate` | Recalcula draft o complete; irrepetible (`PROFILE_NOT_DEACTIVATED`) |
| `deleteProfile` | `DELETE /management/profiles/{subject}` → 200 | Devuelve la lápida; un reintento devuelve la original |
| `reinstateSubject` | `POST /management/deleted-subjects/{subject}/reinstate` → 204 | Levanta la lápida sin restaurar nada ni publicar evento |

### Superficie servidor-a-servidor

Hay dos familias, separadas por contenido y por scope. **El scope autoriza sobre cualquier subject**.

| Operación | Endpoint | Scope | Notas |
|---|---|---|---|
| `resolveContactForServices` | `GET /services/profiles/{subject}/contact` | `profile-contact:read` | Caché 60 s por subject, invalidada por los diez eventos |
| `resolveDeliveryProfileForServices` | `GET /services/profiles/{subject}/delivery` | `profile-delivery:read` | Igual; `PROFILE_INCOMPLETE` (409) enumera en `details` lo que falta |
| `resolveContactsBatchForServices` | `POST /services/profiles/contact/batch` | `profile-contact:read` | 1..100; nunca falla entero; deduplica; conserva el orden; sin caché |
| `resolveDeliveryProfilesBatchForServices` | `POST /services/profiles/delivery/batch` | `profile-delivery:read` | Igual, con reason `incomplete` |

Las cuatro resuelven un perfil en cualquier estado, y el status viaja en la respuesta. La entregabilidad
se evalúa sobre los datos, no sobre el status: un perfil `deactivated` con datos completos se resuelve.

## 5. Fronteras e integraciones

- **Dependencias:** ninguna. El servicio no llama a nadie: no tiene `dependencies`, `http-clients`,
  `storage` ni `mail`.
- **Mensajería:** solo publica, en el canal `profileEvents`, con **outbox**. Diez eventos con **estado**,
  todos con `version` = `lockVersion`:
  - `ProfileProvisioned`, `ProfileContactEmailRefreshed` y `ProfileContactDetailsChanged`. Este último
    lleva el status, porque la completitud no tiene evento propio.
  - `AddressAdded`, que lleva `previousDefaultAddressId` cuando la nueva desbanca a otra.
  - `AddressChanged`, `AddressRemoved` (con `wasDefault`) y `DefaultAddressChanged` (con
    `previousAddressId`).
  - `ProfileDeactivated` (con `deactivatedBy` y `deactivatedAt`) y `ProfileReactivated`.
  - `ProfileDeleted`, sin PII, con `deletedAt` y `reason` iguales a los de la lápida.

  Cada transacción publica como mucho un evento. **`ProfileDeleted` cierra la encarnación** del subject: el
  consumidor olvida todo lo suyo, también la última versión vista, y un `ProfileProvisioned` posterior a
  `reinstateSubject` vuelve a empezar en 0. Hoy no hay consumidores declarados.
- **Persistencia:**
  - Modelo relacional, con `subject` como clave natural.
  - Índices de `UserProfile`: `[status, createdAt]`, `[createdAt]`, `[contactEmail]` y `[displayName]`.
  - Índice único **condicionado** sobre `[userProfile, type]` cuando `isDefault = true`.
  - Frontera transaccional `per-operation`, bloqueo optimista `declared` (solo `UserProfile`),
    `timestamps: declared` y `authorship: all`.
  - Sin retención.
- **Seguridad:**
  - OIDC con el token en la cabecera; `callerSubject` sale del claim `sub`.
  - Clientes máquina por client-credentials, con la audiencia `user-profile` validada.
  - Roles: `profile-support` (`profile:read`, `profile:deactivate`) y `profile-admin` (además,
    `profile:delete`, que cubre borrar y levantar lápidas).
  - Clientes: `order-service` solo tiene el scope de entrega y `notification-service` solo el de contacto.
  - CORS para el navegador, con `Authorization`, `Content-Type` e `Idempotency-Key`, exponiendo
    `X-Correlation-Id`, sin credentials y `maxAge` 3600.

## 6. Decisiones de diseño (qué / por qué)

**Frontera y modelo:**

- **Un dato, un dueño.** El perfil no guarda credenciales, MFA, sesiones ni roles, y no se suscribe al
  servidor de identidad. `contactEmail` es una réplica de solo lectura que se refresca desde el token en
  cada petición. El email nunca es clave, porque puede cambiar de dueño; el `sub` sí lo es.
- **Agregados.** `UserProfile` contiene `Address` porque la completitud y la predeterminada por tipo son
  invariantes que cruzan los dos. La lápida es su propio agregado.
- **Nombres territoriales genéricos y denormalizados.** El diseño sirve a cualquier país; que el código y
  el nombre diverjan con los años es historia, no inconsistencia. Solo se valida la forma: consultar un
  catálogo territorial acoplaría el servicio a un proveedor que no le corresponde.
- **`DeletionReason` cerrado.** Así la lápida no puede acumular PII en texto libre.

**Aprovisionamiento y borrado:**

- **Aprovisionamiento solo en `/me`.** En back-office y M2M no se crea perfil: un operador no recibe un
  perfil de cliente como efecto secundario, y su propia lápida no le bloquea el trabajo. Se descartó
  aprovisionar en cualquier endpoint.
- **Aprovisionamiento en transacción propia.** Mantiene la regla de un evento por transacción y deja las
  versiones consecutivas. El perfil queda creado aunque falle la operación que lo disparó. Se descartó la
  misma transacción, que publicaría dos eventos.
- **Lápida en lugar de borrado a secas.** Sin lápida, la siguiente petición del titular recrearía el
  perfil. Que sea permanente y solo se levante por un acto administrativo es una decisión explícita.
- **El 403 de token de máquina vive solo en la operación interna.** `SUBJECT_NOT_PROVISIONABLE` es una
  discriminación por tipo de token, no por recurso. Declararlo en las operaciones de `/me` levantaba
  `OBL-RESOURCE-SCOPE`, que no admite aceptación. Las reglas de `/me` nombran su propagación y los
  escenarios lo cubren. Coste aceptado: el error no está en los `errors` de esas operaciones; el OpenAPI lo documenta en `/me` como propagado del aprovisionamiento.

**Contrato con los clientes:**

- **Mutaciones que devuelven el perfil completo.** El status recalculado no es predecible para el cliente.
  Por eso `removeMyAddress` responde 200 con cuerpo y no 204.
- **La dirección nueva es la última de `addresses`.** Es la forma de identificar el `addressId` creado
  sin otro endpoint. Las altas sobre un mismo perfil las serializa el bloqueo optimista. El 201 sale sin
  `Location`, porque no hay recurso propio que leer.
- **Superficie M2M en dos familias y lotes por POST.** Contacto y entrega se separan por contenido y por
  scope, para que `notification-service` no pueda obtener una dirección. Los lotes van por POST porque los
  subjects son opacos y no caben con garantías en una query. El scope autoriza sobre cualquier subject:
  acotarlo acoplaría este servicio a sus consumidores.
- **Entregabilidad sobre los datos.** Un perfil desactivado con datos completos se resuelve con su status
  y el consumidor decide. Se descartaron `PROFILE_DEACTIVATED` y tratarlo como incompleto.

**Decisiones estructurales** (registro completo en `specs/user-profile/decisions.yaml`):

| § | Decisión | Elegido | Descartado | Por qué |
|---|---|---|---|---|
| 3.1 | Fiabilidad de publicación | `outbox` | best-effort | Los eventos alimentan réplicas versionadas: un hueco las deja rancias sin que nadie lo vea, y un `ProfileDeleted` perdido es PII que nadie borra |
| 3.2 | Idempotencia del aprovisionamiento | `payload-field callerSubject`, permanente | client-key | El front abre varias peticiones a la vez en el primer acceso; la unicidad de subject es la guarda |
| 3.2 | Idempotencia de `addMyAddress` | `client-key`, 24 h, por titular, no exigida | global; obligatoria | Un doble clic crearía dos direcciones y gastaría cupo; acotada por titular para que dos personas no se vean; no se exige para no romper clientes |
| 3.2 | Resto de comandos | Sin idempotency (naturales, irrepetibles o con lápida) | client-key | Reemplazos y fijaciones son no-op al repetirse; de-/reactivar son irrepetibles por lifecycle; el borrado lo guarda la lápida |
| 3.3 | Caché M2M individual | 60 s por subject, invalidada por los 10 eventos | sin caché | Lecturas calientes por pedido o aviso; el status viaja, así que cualquier evento la cambia; el TTL es respaldo |
| 3.3 | Resto de queries | Sin caché | TTL | El back-office tiene que ver el estado real; los lotes reconcilian réplicas |
| 3.4 | Superficie M2M | Operaciones propias `audience: services` | `both`; no exponer | Contratos estables y mínimos, que evolucionan aparte del de usuarios |
| 3.7 | Frontera transaccional | `per-operation` | per-aggregate | Borrar el perfil y escribir la lápida tocan dos agregados y deben confirmar juntos |
| 3.8 | Paginación | offset, 25/100 | sin paginar | `listProfiles` crece con cada persona que se autentica |
| 3.9 | Concurrencia | `declared` (`lockVersion` en `UserProfile`), 409 `CONCURRENT_MODIFICATION` | none | El status depende de todo el agregado y `lockVersion` es la versión de los eventos |
| 3.9b | Timestamps | `declared` | all | `createdAt` ordena el listado y `updatedAt` viaja en M2M |
| 3.9b | Autoría | `all` (última escritura, fuera de contrato) | declared; none | Lo que sí es contrato se declara en el dominio: `deactivatedBy`/`deactivatedAt` y `deletedBy` |

**Decisiones del análisis de huecos** (`specs/user-profile/gaps.yaml`):

- **Multidispositivo:** entre dos dispositivos del mismo titular gana la última escritura. El único que
  escribe es el titular y el bloque se reemplaza entero; solo lo estrictamente simultáneo choca.
- **Back-office sobre su propio subject:** se permite.
- **Lotes M2M mal formados:** un lote vacío o con un subject inválido es un `400` del lote entero. «Nunca
  falla entero» se refiere a la resolución.
- **PII en el broker:** coste aceptado. La retención del canal es de despliegue, y `ProfileDeleted` es la
  señal para que cada consumidor borre lo suyo.
- **Subjects en la ruta:** viajan codificados en `%`. Se acepta el riesgo de proxies que rechacen `%2F`.

## 7. Ficha de reutilización: adoptar, derivar o evolucionar

### Contrato estable vs adaptable

- **Estable** (cambiarlo es *major*):
  - Los `code` de error: `SUBJECT_NOT_PROVISIONABLE`, `PROFILE_DELETED`, `PROFILE_DEACTIVATED`,
    `PROFILE_NOT_FOUND`, `ADDRESS_NOT_FOUND`, `ADDRESS_LIMIT_REACHED`,
    `SUBDIVISION_CODE_COUNTRY_MISMATCH`, `TERRITORY_NAME_MISSING`, `PROFILE_ALREADY_DEACTIVATED`,
    `PROFILE_NOT_DEACTIVATED`, `SUBJECT_NOT_DELETED`, `PROFILE_INCOMPLETE` y `TOO_MANY_SUBJECTS`.
  - Los diez eventos y sus payloads.
  - Las rutas bajo `/api/v1`, que dentro de v1 solo admiten cambios aditivos.
  - Los roles, permisos y scopes.
  - La semántica de `version` y de la encarnación.
- **Adaptable sin romper a nadie:**
  - El TTL de la caché M2M, el límite de 20 direcciones y el tope de 100 por lote.
  - El TTL de la `Idempotency-Key` y las cotas de longitud de los value types (al alza).
  - El tamaño de página.
- **Versionado:** según `docs/methodology.md` (patch, minor o major según el impacto en el contrato).

### Puntos de extensión típicos

- **Enums ampliables:** `AddressType` (por ejemplo `pickup`) y `UnresolvedReason`. Ampliar `DeletionReason`
  exige mantenerlo cerrado y sin texto libre.
- **Proyecciones M2M nuevas:** una familia más con su propio scope, por ejemplo facturación con la billing
  predeterminada, sin tocar las existentes.
- **Capas opcionales ausentes:** un derivado puede añadir `storage` (avatar), `dependencies` (validar
  territorios contra un catálogo) o el alta administrativa, que es la variante de familia prevista.
- **Piezas reutilizables:**
  - El patrón de aprovisionamiento just-in-time con lápida.
  - El value object `PostalAddress` internacional.
  - Las proyecciones M2M separadas por scope con lotes que nunca fallan enteros.

### Supuestos y limitaciones

- **Un solo servidor de identidad y un solo tenant.** Todos los tokens los emite el mismo IdP, y un `sub`
  identifica a una persona en todo el servicio. El diseño no es multi-tenant: para eso habría que acotar
  el `sub` por emisor o por tenant, y eso lo haría un derivado.
- **El IdP es la fuente del email.** Se asume que el IdP verifica el email de contacto. El servicio no
  verifica direcciones ni teléfonos (no hay OTP) y el teléfono es un dato de negocio declarado por el
  titular, distinto del de MFA.
- **Sin historial ni retención.** No se guarda el historial de cambios: la autoría solo registra la última
  escritura, más `deactivatedBy` y `deletedBy` como datos de dominio. Tampoco hay purga. Un perfil vive
  hasta que se borra, y del borrado solo queda la lápida, sin PII.
- **Volumen moderado y lectura M2M dominante.** El listado de back-office es la única colección. Se
  espera que la carga la dominen las lecturas M2M individuales, y por eso solo ellas tienen caché. Los
  índices de búsqueda declaran la intención (prefijo sin mayúsculas ni acentos), y su materialización es
  del motor.
- **Limitaciones deliberadas:**
  - No hay alta administrativa de identidades ni validación de existencia de territorios.
  - No hay suscripción a eventos del IdP: un cambio de email solo se refleja en la siguiente petición del
    titular.
  - Los scopes M2M no se acotan por subject.

### Cómo reutilizarlo

- **Antes de decidir:** `keel describe user-profile` da el resumen mecánico.
- **Adoptarlo:** si tu servicio de perfiles tiene un servidor OIDC dueño de la identidad, consumidores
  M2M de contacto y de entrega, y aprovisionamiento por primer acceso, el diseño sirve **tal cual**.
  `keel registry get user-profile` lo trae con sus derivados al día y se va directo a generar.
- **Derivarlo:** si cambia algo del contrato (otros consumidores, alta administrativa, otra regla de
  completitud), `keel new <nuevo> --from registry:user-profile` clona solo el spec con linaje `basedOn`, y
  `/keel-design` entrevista solo sobre lo que cambia.
- **Qué se espera:** para el caso general, **adopción**. La derivación queda para las variantes de familia.
