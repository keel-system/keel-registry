# user-profile — Documento de diseño

> specs/user-profile v0.1.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`user-profile` posee el **perfil de negocio** de cada persona —contacto y direcciones— y adopta, por aprovisionamiento just-in-time, la identidad que crea un servidor de identidad. Nunca es dueño de credenciales.

La tesis que hace correcto o incorrecto este diseño es que **hay un solo dueño de la identidad y un solo dueño del perfil, nunca los dos sobre el mismo dato**. El servidor de identidad posee lo que sirve para **entrar** (credenciales, MFA, sesiones, roles, el identificador); este servicio posee lo que sirve para **operar** (contacto, direcciones, estado de negocio). La costura entre ambos es un único campo: el `subject` del token, que es la clave de negocio del agregado.

De ahí salen dos propiedades que conviene tener presentes antes de leer nada más:

- **Este servicio nunca llama al servidor de identidad**, ni siquiera para validar el token: comprueba la firma en local contra las claves públicas del emisor. Y el servidor de identidad nunca llama a este servicio en el camino caliente. El emisor concreto es una URL de configuración, no una línea de código, y por eso el mismo diseño vale sin cambios para cualquier proveedor.
- **El usuario nace en el servidor de identidad y este servicio lo adopta.** No hay alta: hay adopción, en la primera petición autenticada que llegue, sea al endpoint que sea.

**Qué queda fuera, a propósito**: el alta administrativa de la identidad (crear la persona desde aquí — es otra variante de la familia), las preferencias de localización y notificación, los consentimientos RGPD versionados, el avatar y cualquier catálogo territorial. Este último no es una omisión: ver § 3.

## 2. Modelo de dominio

### Value types

Nueve tipos con significado de negocio, en lugar de constraints repetidas inline:

| Tipo | Qué es |
|---|---|
| `SubjectIdentifier` | Identificador opaco e inmutable del sujeto tal como lo emite el servidor de identidad. No se interpreta, no se parsea y no se deriva de él ningún otro dato. |
| `EmailAddress`, `PhoneNumber` | Correo de contacto; teléfono en E.164. |
| `CountryCode` | ISO 3166-1 alpha-2. |
| `PostalCode` | **Sin `pattern`**, a propósito (ver § 3). |
| `SubdivisionCode` | Código de división territorial de primer nivel, ISO 3166-2 (`CO-ANT`). |
| `TerritoryCode` | Código de segundo nivel en la codificación nacional. Opaco, sin patrón universal. |
| `PostalAddress` | Value object compuesto: la dirección entera, con el nombre de cada territorio al lado de su código. Es la unidad que viaja y que un consumidor congela. |
| `ContactDetails`, `DeliveryProfile`, `UnresolvedSubject` | Las tres proyecciones de la superficie servidor-a-servidor. |
| `ProfileStatus`, `AddressType`, `DeletionReason`, `UnresolvedReason` | Enums nominales. |

### Entidades y agregados

**`UserProfile`** (raíz) — `id`, `subject` (único), `contactEmail`, `givenName`, `familyName`, `displayName`, `contactPhone`, `status`. Campos `generated`: `id`, `lockVersion`, `createdAt`, `updatedAt`. Ningún campo `sensitive` ni `computed`.

**`Address`** (entidad hija) — una ranura con `label`, `type` (`shipping`/`billing`), `isDefault` y un `postal` (`PostalAddress`). Es entidad y no value object en lista porque tiene identidad propia: se referencia por su id, se consulta aparte y se modifica sola.

**`DeletedSubject`** (raíz de su propio agregado) — la lápida: `subject` (id), `deletedAt`, `deletedBy`, `reason`.

Dos agregados: `UserProfile` + `Address`, y `DeletedSubject` solo. La lápida está **fuera** del agregado del perfil porque tiene que sobrevivir precisamente a su borrado.

### Ciclo de vida de `UserProfile`

```
draft ⇄ complete          draft/complete → deactivated → draft/complete
```

`draft` es el estado inicial (`default`) y es **válido, no un error**: el perfil nace incompleto y se completa por perfilado progresivo. Un perfil está `complete` si y solo si tiene `contactEmail`, `contactPhone` y `displayName` con valor **y** una dirección `shipping` marcada por defecto. La degradación es simétrica: quitar la dirección por defecto devuelve el perfil a `draft`. Reactivar recalcula la completitud con los datos del momento; no se guarda ningún estado anterior.

## 3. Invariantes y reglas clave

Las tres que un adoptante no puede ignorar:

**a) El nombre del territorio viaja junto al código, denormalizado y copiado al guardar, no resuelto al leer.** Una dirección es un hecho histórico: los territorios se crean, se fusionan y se renombran, así que una dirección que solo guardara el código pasaría a significar otra cosa —o nada— el día que su territorio cambie, y las direcciones viejas dejarían de poder imprimirse como se emitieron. Que código y nombre diverjan con los años no es una inconsistencia que arreglar: es la historia.

**b) Este servicio NO valida la división territorial contra ningún catálogo.** Valida la **forma** —que el código de primer nivel, cuando venga, sea un ISO 3166-2 del país declarado; que no llegue un código sin su nombre— y nada más. Que el territorio exista, que pertenezca a esa división y que el nombre sea el oficial es responsabilidad del **llamante**, que lo resuelve en el borde contra su propio catálogo. **El servidor acepta `localityName: "Medallo"` sin rechistar**, y eso es contrato: quien se integre tiene que saberlo. Está escrito también en `INTEGRATION.md`, y los escenarios FL-TER-001 y FL-TER-002 lo hacen verificable en las dos direcciones.

**c) `postalCode` es opcional y sin `pattern`.** Hay países donde existe pero nadie lo escribe y países donde no existe. Un patrón único sería una garantía falsa que rechazaría direcciones válidas.

Y las tres del patrón de adopción:

- **Los tokens de máquina no aprovisionan nada.** Un token de client-credentials trae subject pero no hay persona detrás; sin la guarda se crearía un perfil fantasma por cada servicio que nos llame (`SUBJECT_NOT_PROVISIONABLE`, 403).
- **Un subject borrado no se vuelve a aprovisionar solo.** Como no llamamos al servidor de identidad, la identidad sobrevive al borrado y puede presentar un token válido al día siguiente. La lápida lo impide (`PROFILE_DELETED`, 410).
- **El perfil incompleto falla con un error de dominio propio, nunca con un 500.** `PROFILE_INCOMPLETE` (409) dice qué falta, para que el llamante sepa qué modal abrir.

Estructurales: como mucho una dirección por defecto **por tipo**; `contactEmail` solo se escribe en el aprovisionamiento; un perfil `deactivated` no admite mutaciones **por la API** (el refresco del correo desde el claim sí, en cualquier estado); un perfil tiene como mucho veinte direcciones; **no hay retención automática** — un perfil vive mientras nadie lo borre.

## 4. Qué hace

**Transversal.** `provisionProfileFromIdentity` (`internal: true`) es el corazón del patrón y su único punto de entrada: la invoca la frontera de identidad en la primera petición autenticada, busca por subject y, si no está, inserta con los claims del token. Es deliberadamente aburrida. Solo el `subject` es obligatorio: si el token no trae email ni nombres, el perfil nace igual en `draft`. Idempotente por `payload-field` sobre el subject, que es la clave natural del agregado.

**El usuario sobre sí mismo** (`/api/v1/me/...`, siete operaciones): `getMyProfile`, `updateMyContactDetails` (sin el correo — es del servidor de identidad), `addMyAddress` (idempotente por `Idempotency-Key`, 24 h), `updateMyAddress`, `removeMyAddress`, `setMyDefaultAddress` y `deleteMyProfile`. Las cinco mutaciones devuelven `200` con el perfil entero, porque todas recalculan el `status` de una forma que el cliente no puede predecir. **Ninguna ruta lleva el subject**: lo estampa el servidor desde el claim `sub`.

**Back-office y soporte** (`/api/v1/management/...`, seis operaciones): `listProfiles` (paginada 25/100, orden por fecha de alta descendente, búsqueda por prefijo sobre correo y nombre), `getProfile`, `deactivateProfile`, `reactivateProfile`, `deleteProfile` y `reinstateSubject`. Todas claveadas por `subject`, no por el id técnico.

### Superficie servidor-a-servidor

Cuatro operaciones propias, con contrato propio, **ninguna compartida** con la superficie de usuarios y ninguna `audience: both`:

| Operación | Método y ruta | Devuelve | Scope |
|---|---|---|---|
| `resolveContactForServices` | `GET /api/v1/services/profiles/{subject}/contact` | `ContactDetails` | `profile-contact:read` |
| `resolveContactsBatchForServices` | `POST /api/v1/services/profiles/contact/batch` | `ContactDetails[]` + `UnresolvedSubject[]` | `profile-contact:read` |
| `resolveDeliveryProfileForServices` | `GET /api/v1/services/profiles/{subject}/delivery` | `DeliveryProfile` | `profile-delivery:read` |
| `resolveDeliveryProfilesBatchForServices` | `POST /api/v1/services/profiles/delivery/batch` | `DeliveryProfile[]` + `UnresolvedSubject[]` | `profile-delivery:read` |

Las dos familias están separadas **en la ruta y en el scope**: `notification-service` tiene solo el scope de contacto, así que no hay ninguna operación que le devuelva una dirección postal. El mínimo privilegio a nivel de campo solo se expresa como operación aparte.

Los lotes topan en 100 subjects (`TOO_MANY_SUBJECTS`, 422) y son también el **arranque y la reconciliación** de una réplica: los eventos solo dan el delta desde que alguien se suscribió. Las individuales llevan caché de 60 s con `invalidatedBy` completo; los lotes no.

Un subject que no resuelve sale en `unresolved` con su `reason` (`not-found`, `deleted`, `incomplete`) en lugar de hacer fallar el lote entero: **distinguir «nunca existió» de «se borró» es contrato**, porque es lo que permite a un consumidor purgar sus propias copias de quien se borró sin purgar las de quien todavía no ha entrado.

## 5. Fronteras e integraciones

**No hay capa `dependencies`, ni `http-clients`, ni suscripciones.** Este servicio no lee ningún dato ajeno ni le encarga trabajo a nadie. Es deliberado y es lo que lo hace **arrancable en solitario**.

**Eventos** (`messaging`, canal lógico `profileEvents`, `reliability: outbox`). Diez eventos que cubren **toda** la superficie de mutación, recorrida command a command: `ProfileProvisioned`, `ProfileContactEmailRefreshed`, `ProfileContactDetailsChanged`, `AddressAdded`, `AddressChanged`, `AddressRemoved`, `DefaultAddressChanged`, `ProfileDeactivated`, `ProfileReactivated` y `ProfileDeleted`. Llevan **estado** y llevan `version`. Ninguna operación publica dos a la vez.

Dos de la lista se escapan si no se buscan a propósito: la **reactivación** (sin su evento, un consumidor deja al usuario desactivado para siempre) y el **refresco del `contactEmail`** dentro del aprovisionamiento, que cambia el dato sin pasar por ningún command — el fallo más silencioso del diseño, porque no hay error ni traza y solo se descubre cuando un correo no llega.

**No nos suscribimos a los eventos del servidor de identidad**, y es decisión, no olvido: el JIT ya cubre todas las vías de alta sin enumerarlas, y un usuario deshabilitado allí deja de recibir tokens, así que no llega hasta aquí. La baja la inicia un administrador contra nuestra API y por esa vía `ProfileDeleted` se emite. **Consecuencia aceptada**: si alguien borra la identidad directamente en la consola del servidor de identidad, este servicio no se entera y quedan datos personales de alguien que ya no existe. El procedimiento correcto es el contrario —se borra aquí primero, y deshabilitar la identidad es un acto aparte— y quien quiera cerrar ese hueco monta reconciliación periódica, que es despliegue y no diseño.

**Almacenamiento** (`persistence`, modelo `relational`): `transactionalBoundary: per-operation`, `optimisticLocking: declared` sobre `UserProfile`, `audit.timestamps: declared` y `audit.authorship: all`. Unicidad condicionada `[profileId, type] unique when isDefault = true` para la dirección por defecto.

> **Una consecuencia del patrón que hay que tener delante al generar**: el aprovisionamiento corre en toda petición autenticada, así que **la primera petición de un usuario nuevo escribe aunque sea un `GET`** — un INSERT más su fila de outbox, en la misma transacción. Ningún endpoint de este servicio puede tratarse como de solo lectura: ni enrutarse a una réplica de lectura, ni ejecutarse en una transacción de solo lectura.

**Acceso** (`security`, protocolo `oidc`): `callerIdentity` estampa el `sub` del token en el campo `callerSubject` de las operaciones de `/me`, que deja de viajar en el cuerpo. Dos roles —`profile-support` (leer, desactivar, reactivar) y `profile-admin` (además borrar y levantar lápidas)—, cinco permisos, `serviceAuth: client-credentials` con `validateAudience`, y `cors` sin credenciales con `Idempotency-Key` entre las cabeceras permitidas.

### Las tres copias: qué puede y qué no puede hacer un consumidor

El contrato distingue tres cosas que se parecen y no lo son:

| | Qué es | ¿Legítima? |
|---|---|---|
| **Snapshot inmutable** | La dirección congelada en el momento del pedido. No es una copia del perfil: es un dato del pedido. | **Sí.** Es lo que necesita `order-service`, y por eso el `DeliveryProfile` lleva la dirección entera, códigos incluidos. |
| **Réplica de solo lectura** | Alimentada por nuestros eventos, con este servicio como **único escritor**. | **Sí**, y solo porque la lista de eventos cubre toda la superficie de mutación. Debe aplicar `ProfileDeleted` purgando, y usar `version` para descartar lo que llegue tarde. |
| **Réplica mutable** | Una copia que el consumidor edita por su cuenta. | **No.** Con dos escritores no hay fuente de verdad. |

## 6. Decisiones de diseño (qué / por qué)

| Decisión | Por qué, y qué se descartó |
|---|---|
| **El `subject` es la clave de negocio**, no el email | El email cambia de dueño; el subject no. Peor aún, un email reutilizado reasignaría el perfil a otra persona. La unicidad no es cosmética: al arrancar, un frontend lanza tres o cuatro peticiones en paralelo, las tres ven que no existe y las tres insertan. Con la restricción, dos fallan y el caso de uso relee. |
| **`Address` es entidad hija, no value object en lista** | Tiene identidad propia: se referencia por id, cambia sola y se consulta aparte. |
| **`PostalAddress` es un value object extraído** | Lo forzó el lote de la familia de entrega —un value object no puede contener una colección, así que las líneas van numeradas— pero es mejor modelado: lo que `order-service` congela **es exactamente ese objeto**, y el snapshot deja de ser un concepto de la prosa. |
| **`status` no es `computed`** | `draft ⇄ complete` sí se deriva de los datos, pero `deactivated` lo provoca un command y no se deriva de ningún campo. Un `computed` con una arista que no computa nada es mentira en el contrato. Que el cliente no lo envíe lo garantiza que no aparece en ningún input. |
| **Existe la arista `complete → draft`** | Si alguien elimina su única dirección de envío por defecto, el perfil deja de cumplir la condición. Sin esa arista, un perfil `complete` sin dirección es un estado que el diseño no admite pero que las operaciones producen. |
| **Reactivar recalcula** | Descartado: guardar `statusBeforeDeactivation`, un campo que existiría solo para eso. Descartado: `deactivated` terminal, que dejaría una desactivación por error sin arreglo. |
| **Lápida permanente con la PII purgada** | Es la decisión con más carga legal del diseño. Se borra el agregado entero y queda `DeletedSubject` con solo el identificador pseudónimo y el rastro de quién y cuándo; el JIT la consulta y nunca re-aprovisiona. Descartados: retención con caducidad (el borrado dejaría de ser permanente en una fecha que nadie recuerda), soft-delete del agregado (conserva la PII de quien pidió el borrado) y purga sin lápida (el JIT deshace el borrado solo, sin error y sin traza). |
| **`reason` de la lápida es un enum, no texto libre** | Un campo de observaciones en una lápida es la puerta de atrás por la que vuelve a entrar la PII que acabamos de purgar. |
| **Solo el `subject` es obligatorio para aprovisionar** | Descartado: exigir el claim de email. El día que alguien toque los scopes del cliente, todo usuario nuevo vería un error en cualquier endpoint, incluido un `GET`. Es la diferencia entre un usuario que entra a onboarding y uno que ve un 500. |
| **`localityName` obligatorio, `adminAreaName` opcional** | Descartado: ambos obligatorios. En Mónaco, Singapur o Malta el llamante rellenaría `adminAreaName` repitiendo el nombre del país, y una garantía que se satisface con relleno no es una garantía. Descartado: ambos opcionales, que aceptaría una dirección imposible de repartir. |
| **Nombres territoriales genéricos** (`adminArea`, `locality`) | Llamarlos `department`/`municipality` convertiría un agregado universal en uno colombiano. El mismo tipo tiene que valer para un departamento, una provincia y un state, y para países que no tienen ninguno de los dos niveles. |
| **El país es el esquema implícito del código de nivel 2** | Descartado: un `localityCodeScheme` explícito — un tercer campo territorial constante que alguien acabará rellenando mal sin detección. Consecuencia aceptada y escrita como invariante: en un país con más de una codificación en circulación el código queda huérfano, y la dirección sigue siendo imprimible porque el nombre viaja al lado. |
| **No hay entidad `Municipality` ni catálogo territorial** | Una relación viva convertiría una fusión de municipios en una corrupción silenciosa de direcciones históricas, y metería mil filas dentro de un agregado de perfil. Y un `need` contra un servicio de territorios volvería este diseño **inarrancable** hasta montar ese otro servicio, además de traer la pregunta de qué hacer cuando el catálogo está caído: fallar deja al usuario encerrado, e ignorar es exactamente esta decisión con tres capas de maquinaria encima. |
| **Dos familias M2M, dos scopes** | Descartado: forma única para los dos consumidores — `notification-service` recibiría y muy probablemente registraría en sus trazas domicilios que no necesita. Descartado: forma mínima sin dirección, que obligaría a `order-service` a montar una réplica antes de poder arrancar. |
| **La dirección M2M viaja entera, códigos incluidos** | El consumidor la congela y no puede volver a pedirla; un snapshot sin códigos no sirve después para liquidar impuestos ni enrutar reparto, y el consumidor acabaría resolviéndolos contra otro catálogo y guardando unos que no son los nuestros. |
| **El scope M2M alcanza a cualquier subject** | Acotarlo exigiría que este servicio supiera qué pedidos tiene `order-service`, y eso es dato ajeno: lo convertiría en dependiente de sus propios consumidores. Queda declarado para que quien conceda el scope sepa que concede acceso al padrón entero. |
| **`reliability: outbox`** | Lo fija `ProfileDeleted`: es la señal con la que el resto del sistema purga sus copias de datos personales, y no hay nada más abajo que la recupere. Descartado: `best-effort` — una supresión ejercida que `order-service` nunca aplica, sin error ni traza, descubierta cuando esa persona vuelva a preguntar. |
| **Los eventos llevan estado** | Descartados: solo el subject (cada evento generaría una llamada síncrona de vuelta y la replicación asíncrona perdería su sentido) y solo el delta (la copia diverge para siempre en cuanto se pierda un evento). **Coste aceptado por escrito**: el broker contiene datos personales y su retención es un sitio más que inventariar y justificar. |
| **Los eventos llevan `version`** | Dos eventos del mismo perfil pueden llegar desordenados; sin versión el consumidor aplica el viejo sobre el nuevo y su copia queda mal para siempre, sin síntoma. `ProfileDeleted` la lleva también, y no por simetría: sin ella un evento de contacto que llegue tarde **resucitaría el perfil en la réplica**. Descartado: ordenar por `occurredAt`, que es reloj y no contador. |
| **Un solo evento por transacción** | `addMyAddress` con `isDefault` no emite además `DefaultAddressChanged`: serían dos eventos con la misma `version` y la regla «descarta el más viejo» dejaría de desempatarlos. Lo que la anterior por defecto deja de serlo lo deriva el consumidor de la invariante publicada. |
| **`transactionalBoundary: per-operation`** | El borrado destruye un agregado y crea otro, y los dos tienen que confirmar juntos. Descartado: `per-aggregate` — en las dos direcciones y las dos malas: perfil borrado sin lápida (el JIT lo re-aprovisiona y el borrado se deshace solo) o lápida sin borrado (PII que ninguna operación puede ya alcanzar, porque un borrado repetido responde éxito al ver la lápida). |
| **`optimisticLocking: declared`, no `all`** | Con `declared`, la versión es un campo del dominio y puede viajar en los eventos. Con `all` existiría pero sería invisible al contrato. Descartado: `none`, que perdería la edición ajena en silencio. |
| **`audit.timestamps: declared`, `authorship: all`** | Soporte necesita responder desde cuándo existe un perfil y el listado ordena por fecha de alta. La autoría se registra pero fuera del contrato: quién tocó un perfil no es algo que `order-service` deba recibir. Quién **borró** no depende de esta política: `deletedBy` es campo propio de la lápida, porque tiene que sobrevivir a la fila cuyo rastro de autoría desaparece con ella. |
| **Idempotencia: tres respuestas distintas** | `provisionProfileFromIdentity` por `payload-field` sobre la clave natural, así que la restricción de unicidad **es** la guarda y no hay registro de claves. `addMyAddress` por `client-key` (24 h), porque es la única alta real de la superficie de usuario. El **borrado** no declara ninguna: la lápida es la guarda, y como es permanente la ventana de deduplicación también — un operador que reintenta tras un timeout recibe el mismo éxito y la lápida original, que es cómo distingue su propio reintento de un borrado ajeno. |
| **`deleteProfile` devuelve la lápida (200), `deleteMyProfile` no (204)** | La operación administrativa cambia más de lo que borra: crea un registro permanente, y ese registro es lo que el operador necesita ver. Al usuario no se le da un recibo de su propia baja. |
| **Las cinco mutaciones de `/me` devuelven `200` con el perfil** | Todas recalculan el `status` de una forma que el cliente no puede predecir. Descartado: REST canónico (`201` + la dirección, `204` al borrar), que obligaría al front de onboarding a un `GET` detrás de cada alta y de cada borrado. |
| **Búsqueda por prefijo, no por subcadena** | Es lo único que un índice ordinario sostiene, así que el listado sigue respondiendo cuando el padrón crezca. Es **cota, no rendimiento**. |
| **Dos roles y no uno** | El corte está donde está la irreversibilidad: desactivar se deshace con reactivar, borrar no se deshace nunca. Descartado: un solo rol, que dejaría que cualquiera que pueda buscar un perfil lo borre. |
| **Todo campo sin valor viaja como nulo, nunca omitido** | En un evento con estado, omitir un campo que el usuario borró y omitir uno que no cambió se escribirían igual, y el consumidor no podría saber si vaciar su copia. |
| **El `code` es contrato, el texto del mensaje no** | Con dos stacks implementando el mismo diseño, reproducir un texto al carácter es la afirmación más frágil que se puede escribir. |

## 7. Ficha de reutilización: adoptar, derivar o evolucionar

### Contrato estable vs adaptable

**Estable** — cambiarlo rompe a alguien y exige versión mayor:

- Los **16 códigos de error** y su status.
- Los **10 nombres de evento** y sus payloads; la lista es **completa sobre la superficie de mutación** y recortarla pudre cualquier réplica ajena.
- Las **rutas publicadas** bajo `/api/v1`, y en especial las cuatro de `/services`.
- Los **roles** (`profile-support`, `profile-admin`), los **permisos** y los dos `serviceClients`.
- La regla territorial: **se valida forma, no existencia**. Un adoptante que añada validación de catálogo cambia el contrato.
- La semántica de `404` vs `410` vs `409 PROFILE_INCOMPLETE` en la superficie M2M.

**Adaptable** sin romper a nadie: los `ttlSeconds` de la caché y de la idempotencia, las cotas (20 direcciones, lote de 100, paginación 25/100), el orden por defecto del listado, la semántica del `search`, los índices, y cualquier regla de `use-cases` que no cambie un `code`.

Versionado del spec según `docs/methodology.md`: **patch** para prosa y rationale; **minor** para añadir campos opcionales, operaciones, errores o eventos nuevos; **major** para quitar o volver obligatorio cualquiera de ellos. El contrato HTTP sigue la misma regla: dentro de `/api/v1` solo caben cambios aditivos.

### Puntos de extensión típicos

- **Capas ausentes que un derivado puede añadir**: `storage` (avatar), `mail` (avisos propios), `dependencies` + `http-clients` (si el contexto sí tiene un servicio de territorios y quiere validar contra él).
- **Enums ampliables**: `AddressType` (añadir `work`, `pickup`), `DeletionReason`, `UnresolvedReason`.
- **`lifecycle`**: hay sitio para un estado `archived` entre `deactivated` y el borrado, si el contexto necesita retención en lugar de purga.
- **`provisionProfileFromIdentity` es sustituible**: es `internal: true`, y una variante con alta administrativa la reemplaza sin tocar el resto del diseño. Esa es, de hecho, la otra variante de la familia.
- **Piezas reutilizables en otro servicio**: el value type `PostalAddress` con su regla de denormalización territorial, el patrón lápida + JIT completo, y la separación de familias M2M por scope.

### Supuestos y limitaciones

- **Un solo servidor de identidad.** El `subject` es único por sí solo y basta como clave de negocio. **Quien necesite federar dos emisores tiene que derivar el diseño** y convertir la clave en el par `(issuer, subject)` —en el agregado, en la lápida, en los diez eventos y en las cuatro operaciones M2M—, porque dos emisores pueden emitir el mismo `sub` para personas distintas.
- **Escala asumida: cientos de miles a millones de perfiles.** Es lo que hace defendibles la paginación obligatoria, la búsqueda por prefijo indexado y el techo del lote. El agregado se mantiene pequeño (20 direcciones como máximo) y cabe entero en una respuesta.
- **Un solo inquilino.** No hay `authentication.scoping` ni ninguna noción de organización: el back-office ve el padrón entero.
- **El token es un JWT verificable en local** contra las claves públicas del emisor. Si el emisor solo ofrece introspección remota, este diseño deja de cumplir su propia premisa de no llamar nunca al servidor de identidad.
- **La frontera de identidad resuelve si el token es de persona o de máquina.** El diseño lo declara como entrada (`actorIsPerson`) pero no dice cómo: depende del proveedor.
- **No cubre**: alta administrativa de identidad, preferencias de localización y notificación, consentimientos RGPD versionados, avatar, catálogo territorial, retención o archivado automático, ni federación de emisores.
- **Limitación conocida y aceptada**: un borrado hecho directamente en la consola del servidor de identidad no llega aquí. Cerrar ese hueco es reconciliación periódica, que es despliegue y no diseño.
- **Limitación de verificación**: ningún escenario distingue una caché que funciona de la ausencia de caché, porque `invalidatedBy` cubre las diez vías de mutación y no queda ninguna vía fuera con la que probar la retención. El `ttlSeconds: 60` se verifica contra el artefacto, no por ejecución. Está escrito en `validation-scenarios.md § Lo que no tiene escenario`.

### Cómo reutilizarlo

Empieza por el resumen mecánico: `keel describe user-profile`.

**Si te sirve tal cual** —y esa es la expectativa para la mayoría de los adoptantes, porque lo estable de este diseño es justamente lo que lo hace útil—, **adóptalo**:

```bash
keel registry get user-profile
```

Llega con sus derivados al día (`DESIGN.md`, contratos formales, panel, `INTEGRATION.md`) y se va **directo a generar**, sin fase de diseño. Lo adaptable —TTLs, cotas, orden del listado— se ajusta después sin volver a entrevistar nada.

**Si tienes que cambiarlo** —federar emisores, añadir alta administrativa, meter avatar o consentimientos, validar contra un catálogo territorial propio—, **derívalo**:

```bash
keel new <tu-servicio> --from registry:user-profile
```

Clona solo el spec con linaje `basedOn` y hace que `/keel-design` arranque en modo derivación, entrevistando solo sobre lo que cambia. Los derivados de `docs/` no se heredan: describen a este servicio y se regeneran al cerrar el tuyo.

**Derivar para acabar usándolo sin cambios es el peor de los dos caminos**: obliga a regenerar a mano todo lo que ya estaba hecho.
