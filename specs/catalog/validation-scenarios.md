# catalog — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/catalog v0.1.0. Contrato de validación para la fase de generación.

## Convenciones de determinación

Valen para **todo** el servicio y ningún escenario las repite.

- **Instantes**: UTC, ISO-8601 con sufijo `Z` (`createdAt`, `updatedAt` y el `occurredAt` de la envoltura de eventos). El servicio no maneja fechas de negocio locales. Se verifican **por forma o por rango** («posterior o igual al instante del `When`»), nunca por valor exacto.
- **Ausencia vs nulo** (`conventions.nulls: include` en el manifiesto): todo campo que un output o un payload de evento declara **está presente**; el que no tiene valor viaja como `null`, nunca omitido. «`description` es nulo» exige la clave presente con valor `null`.
- **Identificadores**: `id` de productos, marcas, categorías e imágenes se verifican por forma (uuid) y por reutilización simbólica dentro del flujo (`b1`, `c1`, `p1`, `i1`…), jamás por valor literal.
- **Autoría** (`createdBy`, `updatedBy`): es el identificador del principal del token con el que se hizo la petición. Qué claim lo da depende del proveedor de identidad, así que se verifica **por reutilización**: dos peticiones con el mismo token producen el mismo valor, y dos tokens de usuarios distintos, valores distintos. Nunca por valor literal.
- **Dinero**: `price.amount` viaja como **número JSON** y se compara numéricamente con escala 2 (`49.90` y `49.9` son el mismo valor). Un importe de **entrada** con más de 2 decimales se **rechaza** con `400` (`scalePolicy: reject`), nunca se redondea. `price.currency` es código ISO 4217.
- **Moneda del catálogo**: es un parámetro de despliegue; el entorno de prueba lo configura a **`EUR`**. Todo `price` válido de este documento lleva `currency: "EUR"`.
- **SKU**: se acepta en mayúsculas o minúsculas y se devuelve **siempre en mayúsculas**; la unicidad es exacta sobre el valor normalizado (`tshirt-01` colisiona con `TSHIRT-01`).
- **Mayúsculas y acentos** (`compare`/`match` en el YAML): la unicidad de `Brand.name` y `Category.name` es **insensible a mayúsculas y acentos** (`Acmé` colisiona con `ACME`). Los filtros `name` de los listados coinciden por **contenido** e ignoran mayúsculas y acentos (`cami` encuentra `Camiseta Básica` y `CAMISÉTA`).
- **Forma del cuerpo de error**: la impone el generador (en keel-spring, `{timestamp, status, error, code, message, details}` más `correlationId`). Los escenarios solo fijan el **`code`** y el **status**; el texto del mensaje **no es contrato**. Un error de forma de la petición (`400` por constraint o campo requerido) no lleva `code` de negocio.
- **Sobre de paginación**: el canónico del DSL, `{ items, page, size, totalElements, totalPages }`, con query params `page` (base 0) y `size`. `defaultSize` 24, `maxSize` 100; un `size` mayor se **recorta** a 100, no da error.
- **Orden de las colecciones**: el `sort` declarado de cada operación, con **desempate por `id` ascendente** siempre. `images` viaja siempre ordenada por `position` ascendente.
- **Archivos**: `productImages` es `public`, así que en las **respuestas HTTP** `images[].file` es una **URL absoluta** que responde `200` a un `GET` anónimo con el binario subido y su content-type. En los **eventos** `images[].file` es la **clave** del objeto (una cadena no vacía que no es una URL), no la URL.
- **Subida de imágenes**: `addProductImage` recibe `multipart/form-data` con una parte `file` (el binario) y los campos `altText` y `main` como partes de texto.
- **Cabecera de idempotencia**: `Idempotency-Key`, **obligatoria** en las 13 operaciones que declaran `idempotency`; sin ella, `400 IDEMPOTENCY_KEY_REQUIRED`. La misma clave con el mismo cuerpo reproduce la respuesta original (mismo status, mismo cuerpo, sin segundo efecto ni segundo evento); con otro cuerpo, `409 IDEMPOTENCY_KEY_REUSED`; dos a la vez, una puede recibir `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
- **Cabecera `Location`**: la llevan las tres creaciones con `201` (`createProduct`, `createBrand`, `createCategory`) con la URI de la petición más el `id` devuelto. En un reintento idempotente de una creación, la respuesta reproducida lleva **también** la misma `Location`. `addProductImage` responde `200` con la ficha del producto (no crea un recurso direccionable propio), así que no hay `Location` que afirmar.
- **Concurrencia**: `optimisticLocking: all`. Dos escrituras concurrentes sobre la misma raíz producen conflicto `409 CONCURRENT_MODIFICATION`, nunca un último-gana silencioso. El cliente no envía versión: el conflicto solo es observable bajo una carrera real.
- **Transición inválida**: una operación con `transitions` aplicada desde un estado que no está en su `from` responde `409 INVALID_STATE_TRANSITION` (código canónico) y no publica nada. Se evalúa **justo después** de `PRODUCT_NOT_FOUND` y antes de cualquier precondición.
- **Sin cambio, sin evento**: un PATCH, `activate*` o `deactivate*` que no cambia ningún valor responde `200` con el recurso, **no** publica evento y **no** toca `updatedAt`/`updatedBy`.
- **Eventos**: todo evento viaja en la envoltura Keel `{ metadata, data }`; los `Then` afirman `metadata.eventType`, el canal y el `data` completo. El `correlationId` de la envoltura es el de la petición que lo provocó.
- **Precedencia de la seguridad**: la autenticación (`401`) y la autorización (`403`) se evalúan **antes** que cualquier validación de la petición, incluida la cabecera `Idempotency-Key` y la forma del cuerpo. Los órdenes de evaluación de cada flujo empiezan después de ellas.
- **Token de otra audiencia**: el arnés obtiene del mismo proveedor de identidad un token de máquina válido emitido para una audiencia **distinta** de `catalog`. Con `validateAudience: true` se rechaza; el status lo fija el generador, no el diseño, y en keel-spring es **`403`** (el token es legítimo pero no está emitido para este servicio).
- **Outbox**: la semántica del mecanismo es la que fija el método (`docs/validation-scenarios.md`, § Reglas de cobertura) y materializa el generador: el relay entrega en **≤ 10 s** tras restablecerse el canal, tiene un **presupuesto de reintentos** propio del generador, y un evento que lo agota se da por **abandonado**: el servidor lo informa por su superficie de operación (en keel-spring, el actuator) y no vuelve a publicarlo. Los escenarios nombran el mecanismo, nunca la métrica ni el número.
- **Coste**: las afirmaciones de coste («no cuesta más consultas al almacén») las exige el método para toda colección que resuelve referencias; afirman la **forma** y su medición es del generador (keel-spring instrumenta las consultas en su arnés).
- **Identidades**: tokens de usuario con rol `catalog-manager` (usuario `u-manager`), con rol `catalog-admin` (usuario `u-admin`) y un token de usuario autenticado **sin ningún rol del catálogo** (`u-none`). Credenciales de máquina: credencial de máquina del cliente `orders` y credencial de máquina del cliente `cart`. Además, el token de otra audiencia descrito arriba. Ninguna otra identidad aparece en este documento.

## Proyecciones

Derivadas del YAML; todo `Then` que enumere un cuerpo se refiere a ellas y **no trae ningún campo adicional**.

- **Producto (gestión)** — `createProduct`, `updateProduct`, `publishProduct`, `unpublishProduct`, `retireProduct`, `getProduct`, `listProducts`, `addProductImage`, `updateProductImage`, `removeProductImage`:
  `id`, `sku`, `name`, `description`, `price { amount, currency }`, `status`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `brand { id, name, description, active }`, `category { id, name, description, active }`, `images [ { id, file, altText, position, main } ]`. **No** trae `brandId` ni `categoryId` (van embebidos) ni ningún campo de versión.
- **Producto (público y M2M)** — `listPublicProducts`, `getPublicProduct`, `getProductForServices`, `listProductsBatchForServices`: la de gestión **sin** `createdAt`, `updatedAt`, `createdBy`, `updatedBy`. `brand` y `category` embebidos llevan también `active` (el DSL no recorta dentro de un agregado embebido).
- **Marca / categoría (gestión)**: `id`, `name`, `description`, `active`.
- **Marca / categoría (público)** — `listPublicBrands`, `listPublicCategories`: `id`, `name`, `description` (sin `active`).
- **Payload `ProductCreated` / `ProductUpdated`** (canal `productEvents`): `productId`, `sku`, `name`, `description`, `price { amount, currency }`, `status`, `brandId`, `brandName`, `categoryId`, `categoryName`, `images [ { imageId, file, altText, position, main } ]` (con `file` como clave).
- **Payload `ProductStatusChanged`** (canal `productEvents`): `productId`, `sku`, `previousStatus`, `status`, `reason`.
- **Payload `BrandCreated` / `BrandUpdated`** (canal `taxonomyEvents`): `brandId`, `name`, `description`, `active`. **`BrandDeleted`**: `brandId`, `name`.
- **Payload `CategoryCreated` / `CategoryUpdated`** (canal `taxonomyEvents`): `categoryId`, `name`, `description`, `active`. **`CategoryDeleted`**: `categoryId`, `name`.

## Aislamiento y orden de ejecución

Cada flujo `FL-*` arranca con el servicio **reseteado** (sin marcas, categorías, productos, imágenes ni eventos en los canales) y construye su `Given` **por la API** dentro del propio flujo. Los escenarios de un flujo se ejecutan en orden y encadenan estado; ningún flujo depende de otro. Salvo que se diga otra cosa, las operaciones de gestión se llaman con el token de `u-manager` y cada mutación con idempotencia lleva una `Idempotency-Key` nueva.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|---|---|---|
| `createBrand` | FL-BRD-001, FL-IDM-001 | gestión |
| `updateBrand` | FL-BRD-010, FL-IDM-001 | gestión |
| `activateBrand` | FL-BRD-020, FL-IDM-001 | gestión |
| `deactivateBrand` | FL-BRD-020, FL-IDM-001 | gestión |
| `deleteBrand` | FL-BRD-030, FL-BRD-031, FL-SEC-001 | gestión |
| `getBrand` | FL-BRD-001, FL-BRD-030 | gestión |
| `listBrands` | FL-BRD-040 | gestión |
| `createCategory` | FL-CAT-001, FL-IDM-001 | gestión |
| `updateCategory` | FL-CAT-010, FL-IDM-001 | gestión |
| `activateCategory` | FL-CAT-020, FL-IDM-001 | gestión |
| `deactivateCategory` | FL-CAT-020, FL-IDM-001 | gestión |
| `deleteCategory` | FL-CAT-030, FL-CAT-031, FL-SEC-001 | gestión |
| `getCategory` | FL-CAT-001, FL-CAT-030 | gestión |
| `listCategories` | FL-CAT-040 | gestión |
| `createProduct` | FL-PRD-001, FL-PRD-002, FL-BRD-020, FL-CAT-020, FL-BRD-031, FL-CAT-031, FL-IDM-001, FL-OBX-001 | gestión |
| `updateProduct` | FL-PRD-010, FL-PRD-011, FL-IDM-001 | gestión |
| `publishProduct` | FL-LCY-001, FL-LCY-002 | gestión |
| `unpublishProduct` | FL-LCY-001 | gestión |
| `retireProduct` | FL-LCY-001, FL-LCY-003, FL-SEC-001 | gestión |
| `getProduct` | FL-PRD-001, FL-PRD-010, FL-IMG-001 | gestión |
| `listProducts` | FL-PRD-040 | gestión |
| `addProductImage` | FL-IMG-001, FL-IMG-002, FL-IDM-001 | gestión |
| `updateProductImage` | FL-IMG-003, FL-IDM-001 | gestión |
| `removeProductImage` | FL-IMG-004, FL-LCY-002, FL-IDM-001 | gestión |
| `listPublicProducts` | FL-PUB-001, FL-PUB-002, FL-BRD-020 | pública |
| `getPublicProduct` | FL-PUB-003, FL-LCY-001 | pública |
| `listPublicCategories` | FL-PUB-004, FL-CAT-020 | pública |
| `listPublicBrands` | FL-PUB-004, FL-BRD-020 | pública |
| `getProductForServices` | FL-M2M-001, FL-M2M-003 | **servidores (M2M)** |
| `listProductsBatchForServices` | FL-M2M-002, FL-M2M-003 | **servidores (M2M)** |

**Cobertura de errores** — los 25 códigos del diseño y los 4 canónicos que el diseño acepta:

| `code` | Status | Flujo que lo ejercita |
|---|---|---|
| `IDEMPOTENCY_KEY_REQUIRED` | 400 | FL-BRD-001, FL-PRD-001, FL-IDM-001 |
| `PRODUCT_SKU_ALREADY_EXISTS` | 409 | FL-PRD-001 |
| `CURRENCY_NOT_SUPPORTED` | 422 | FL-PRD-001, FL-PRD-010 |
| `BRAND_REFERENCE_NOT_FOUND` | 422 | FL-PRD-001, FL-PRD-010, FL-BRD-031 |
| `CATEGORY_REFERENCE_NOT_FOUND` | 422 | FL-PRD-001, FL-PRD-010, FL-CAT-031 |
| `BRAND_INACTIVE` | 422 | FL-BRD-020, FL-PRD-010 |
| `CATEGORY_INACTIVE` | 422 | FL-CAT-020, FL-PRD-010 |
| `PRODUCT_NOT_FOUND` | 404 | FL-PRD-010, FL-LCY-001, FL-IMG-002, FL-PUB-003, FL-M2M-001, FL-PRD-001 |
| `PRODUCT_RETIRED` | 409 | FL-LCY-003 |
| `PRICE_NOT_POSITIVE` | 422 | FL-PRD-010 |
| `PRODUCT_HAS_NO_IMAGES` | 409 | FL-LCY-002 |
| `PRODUCT_PRICE_NOT_POSITIVE` | 409 | FL-LCY-002 |
| `INVALID_PRICE_RANGE` | 400 | FL-PRD-040, FL-PUB-001 |
| `IMAGE_LIMIT_REACHED` | 409 | FL-IMG-002 |
| `FILE_TOO_LARGE` | 413 | FL-IMG-002 |
| `UNSUPPORTED_CONTENT_TYPE` | 415 | FL-IMG-002 |
| `IMAGE_NOT_FOUND` | 404 | FL-IMG-003, FL-IMG-004 |
| `MAIN_IMAGE_REQUIRED` | 409 | FL-IMG-003 |
| `LAST_IMAGE_OF_ACTIVE_PRODUCT` | 409 | FL-LCY-002 |
| `BRAND_NAME_ALREADY_EXISTS` | 409 | FL-BRD-001, FL-BRD-010 |
| `BRAND_NOT_FOUND` | 404 | FL-BRD-010, FL-BRD-020, FL-BRD-030 |
| `BRAND_HAS_PRODUCTS` | 409 | FL-BRD-030, FL-BRD-031 |
| `CATEGORY_NAME_ALREADY_EXISTS` | 409 | FL-CAT-001, FL-CAT-010 |
| `CATEGORY_NOT_FOUND` | 404 | FL-CAT-010, FL-CAT-020, FL-CAT-030 |
| `CATEGORY_HAS_PRODUCTS` | 409 | FL-CAT-030, FL-CAT-031 |
| `INVALID_STATE_TRANSITION` *(canónico)* | 409 | FL-LCY-001, FL-LCY-003 |
| `CONCURRENT_MODIFICATION` *(canónico)* | 409 | FL-PRD-011 |
| `IDEMPOTENCY_KEY_IN_PROGRESS` *(canónico)* | 409 | FL-PRD-002, FL-IDM-001 |
| `IDEMPOTENCY_KEY_REUSED` *(canónico)* | 409 | FL-BRD-001, FL-PRD-001 |

**Estados del lifecycle de `Product`**: `draft` (FL-PRD-001), `active` (FL-LCY-001), `retired` (FL-LCY-001, FL-LCY-003).

---

## Marcas

### FL-BRD-001: alta de marca, consulta y unicidad sin acentos

**Given**: no existe ninguna marca.

**When**: `createBrand` — `POST /api/v1/brands` con `Idempotency-Key: k-brd-1`
```json
{ "name": "Acmé", "description": "Ropa técnica de montaña." }
```

**Then**:
1. Status `201`, cabecera `Location` = `/api/v1/brands/{id}` con el `id` devuelto (en adelante `b1`).
2. Cuerpo: `id` (uuid), `name: "Acmé"`, `description: "Ropa técnica de montaña."`, `active: true`. Ningún campo más.
3. `getBrand` — `GET /api/v1/brands/b1` responde `200` con el mismo cuerpo.
4. El canal `taxonomyEvents` recibe exactamente un `BrandCreated` con `data`: `brandId: b1`, `name: "Acmé"`, `description: "Ropa técnica de montaña."`, `active: true`.
5. Reintento secuencial: repetir la misma petición con `Idempotency-Key: k-brd-1` responde `201`, la misma `Location` y el mismo cuerpo (`id: b1`); `listBrands` sigue devolviendo `totalElements: 1` y `taxonomyEvents` no recibe un segundo `BrandCreated`.

**Orden de evaluación**:
1. Cabecera `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Nombre libre (sin mayúsculas ni acentos) → `BRAND_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- Misma petición **sin** `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`; no se crea nada.
- `Idempotency-Key: k-brd-1` con cuerpo distinto (`"name": "Otra"`) → `409 IDEMPOTENCY_KEY_REUSED`; no se crea nada.
- `name: "ACME"` con clave nueva → `409 BRAND_NAME_ALREADY_EXISTS` (colisiona con `Acmé`).
- `name` con 81 caracteres → `400`. `name` ausente → `400`.
- `getBrand` sobre un uuid inexistente → `404 BRAND_NOT_FOUND`.

### FL-BRD-010: edición de marca

**Given**: marca `b1` (`name: "Acme"`, `description: "Ropa técnica."`) y marca `b2` (`name: "Zeta"`), creadas en el flujo; canal `taxonomyEvents` purgado tras las altas.

**When**: `updateBrand` — `PATCH /api/v1/brands/b1`
```json
{ "name": "Acme Outdoor" }
```

**Then**:
1. Status `200`; cuerpo `id: b1`, `name: "Acme Outdoor"`, `description: "Ropa técnica."` (conservada: ausente en la petición), `active: true`.
2. `taxonomyEvents` recibe exactamente un `BrandUpdated` con `data`: `brandId: b1`, `name: "Acme Outdoor"`, `description: "Ropa técnica."`, `active: true`.
3. `PATCH` con `{ "description": null }` → `200` con `description: null`, y un `BrandUpdated` con `description: null`.
4. `PATCH` con `{ "name": "Acme Outdoor" }` (sin cambio) → `200` con el mismo cuerpo y **ningún** evento nuevo en `taxonomyEvents`.

**Orden de evaluación**:
1. `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. La marca existe → `BRAND_NOT_FOUND` (`404`).
3. Nombre libre → `BRAND_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `PATCH /api/v1/brands/b1` con `{ "name": "zéta" }` → `409 BRAND_NAME_ALREADY_EXISTS`.
- `PATCH` sobre uuid inexistente con `{ "name": "Zeta" }` → `404 BRAND_NOT_FOUND` (precedencia: la guarda 2 va antes que la 3, aunque el nombre colisione).
- `{ "name": null }` → `400`.

### FL-BRD-020: desactivar y reactivar una marca

**Given**: marca `b1` (`Acme`, creada **sin** `description`), categoría `c1` (`Camisetas`) y producto `p1` (`sku: "TS-01"`, marca `b1`, categoría `c1`) publicado con una imagen, creados en el flujo.

**When**: `deactivateBrand` — `POST /api/v1/brands/b1/deactivate`

**Then**:
1. Status `200`; cuerpo `id: b1`, `name: "Acme"`, `description: null`, `active: false`.
2. `taxonomyEvents` recibe un `BrandUpdated` con `active: false`.
3. `listPublicBrands` — `GET /api/v1/public/brands` ya no contiene `b1`.
4. `listPublicProducts` — `GET /api/v1/public/products?brandId=b1` sigue devolviendo `p1` (desactivar no oculta productos), con `brand.active: false`.
5. `createProduct` con `brandId: b1` → `422 BRAND_INACTIVE`.
6. Repetir `deactivateBrand` (clave nueva) → `200` con `active: false` y **ningún** `BrandUpdated` nuevo.
7. `activateBrand` — `POST /api/v1/brands/b1/activate` → `200` con `active: true`, un `BrandUpdated` con `active: true`, y `b1` vuelve a `listPublicBrands`.

**Casos borde**:
- `deactivateBrand` y `activateBrand` sobre uuid inexistente → `404 BRAND_NOT_FOUND`.
- Sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`.

### FL-BRD-030: borrado de marca

**Given**: marcas `b1` (`Acme`, sin productos) y `b2` (`Zeta`), categoría `c1` y producto `p1` con marca `b2` en `draft`.

**When**: `deleteBrand` — `DELETE /api/v1/brands/b1` con el token de `u-admin`

**Then**:
1. Status `204`, sin cuerpo.
2. `taxonomyEvents` recibe un `BrandDeleted` con `data`: `brandId: b1`, `name: "Acme"`.
3. `getBrand` sobre `b1` → `404 BRAND_NOT_FOUND`.
4. Repetir el `DELETE` sobre `b1` → `404 BRAND_NOT_FOUND` y ningún `BrandDeleted` nuevo.

**Orden de evaluación**:
1. La marca existe → `BRAND_NOT_FOUND` (`404`).
2. No tiene productos → `BRAND_HAS_PRODUCTS` (`409`).

**Casos borde**:
- `DELETE /api/v1/brands/b2` (tiene `p1`, en `draft`) → `409 BRAND_HAS_PRODUCTS`; `b2` sigue existiendo y no se publica nada.
- `DELETE /api/v1/brands/b2` con el token de `u-manager` → `403` (ver FL-SEC-001).

### FL-BRD-031: borrar una marca a la vez que se le asigna un producto

**Given**: marca `b1` sin productos, categoría `c1`.

**When**: a la vez, `deleteBrand` sobre `b1` (token de `u-admin`) y `createProduct` con `brandId: b1`, `categoryId: c1`, `sku: "RACE-01"`.

**Then**:
1. Exactamente uno de los dos desenlaces: (a) el borrado responde `204` y el alta `422 BRAND_REFERENCE_NOT_FOUND`; o (b) el alta responde `201` y el borrado `409 BRAND_HAS_PRODUCTS`.
2. Ninguna de las dos responde `5xx`.
3. Independiente del ganador: `listProducts` no devuelve ningún producto cuya `brand` no se pueda resolver con `getBrand` (`200`); si `listProducts` devuelve `RACE-01`, `getBrand b1` responde `200`.

### FL-BRD-040: listado de marcas para la gestión

**Given**: marcas creadas en este orden: `Zeta`, `Acme`, `Arcadia`, `Beta`; `Beta` desactivada.

**When**: `listBrands` — `GET /api/v1/brands`

**Then**:
1. Status `200`, sobre de paginación con `page: 0`, `size: 24`, `totalElements: 4`, `totalPages: 1`.
2. `items` en orden de `name` ascendente: `Acme`, `Arcadia`, `Beta`, `Zeta`, cada uno con la proyección de gestión.
3. `?active=false` → solo `Beta`. `?active=true` → `Acme`, `Arcadia`, `Zeta`.
4. `?name=ÁRC` → `Arcadia` (contenido, sin mayúsculas ni acentos). `?name=me` → `Acme`.
5. `?size=2` → `Acme`, `Arcadia`, `totalPages: 2`; `?size=2&page=1` → `Beta`, `Zeta`; `?size=2&page=2` → `items: []`, `totalElements: 4`.
6. `?size=500` → `size: 100`.

## Categorías

Espejo del bloque de marcas sobre `Category`, con su canal `taxonomyEvents` y sus eventos propios.

### FL-CAT-001: alta de categoría, consulta y unicidad sin acentos

**Given**: no existe ninguna categoría.

**When**: `createCategory` — `POST /api/v1/categories` con `Idempotency-Key: k-cat-1`
```json
{ "name": "Camisetas", "description": "Manga corta y larga." }
```

**Then**:
1. Status `201`, `Location` = `/api/v1/categories/{id}` (en adelante `c1`).
2. Cuerpo: `id`, `name: "Camisetas"`, `description: "Manga corta y larga."`, `active: true`. Ningún campo más.
3. `getCategory` — `GET /api/v1/categories/c1` → `200` con el mismo cuerpo.
4. `taxonomyEvents` recibe exactamente un `CategoryCreated` con `data`: `categoryId: c1`, `name: "Camisetas"`, `description: "Manga corta y larga."`, `active: true`.
5. Reintento con `k-cat-1` y el mismo cuerpo → `201`, misma `Location`, mismo cuerpo; `listCategories` con `totalElements: 1`; ningún segundo `CategoryCreated`.

**Orden de evaluación**:
1. `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Nombre libre → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `name: "CAMISÉTAS"` con clave nueva → `409 CATEGORY_NAME_ALREADY_EXISTS`.
- Sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`.
- `getCategory` sobre uuid inexistente → `404 CATEGORY_NOT_FOUND`.

### FL-CAT-010: edición de categoría

**Given**: categorías `c1` (`Camisetas`, `description: "Manga corta."`) y `c2` (`Pantalones`); canal purgado.

**When**: `updateCategory` — `PATCH /api/v1/categories/c1` con `{ "name": "Camisetas y polos" }`

**Then**:
1. `200`; cuerpo `id: c1`, `name: "Camisetas y polos"`, `description: "Manga corta."`, `active: true`.
2. Un `CategoryUpdated` con `data`: `categoryId: c1`, `name: "Camisetas y polos"`, `description: "Manga corta."`, `active: true`.
3. `{ "description": null }` → `200` con `description: null` y su `CategoryUpdated`.
4. Repetir `{ "name": "Camisetas y polos" }` → `200`, mismo cuerpo, **ningún** evento.

**Orden de evaluación**:
1. `Idempotency-Key` → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Existe → `CATEGORY_NOT_FOUND` (`404`).
3. Nombre libre → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `{ "name": "pantalónes" }` → `409 CATEGORY_NAME_ALREADY_EXISTS`.
- uuid inexistente con `{ "name": "Pantalones" }` → `404 CATEGORY_NOT_FOUND` (precedencia sobre la colisión).

### FL-CAT-020: desactivar y reactivar una categoría

**Given**: marca `b1`, categoría `c1` y producto `p1` publicado en `c1`.

**When**: `deactivateCategory` — `POST /api/v1/categories/c1/deactivate`

**Then**:
1. `200`; cuerpo `id: c1`, `name`, `description`, `active: false`.
2. Un `CategoryUpdated` con `active: false`.
3. `listPublicCategories` ya no contiene `c1`.
4. `listPublicProducts?categoryId=c1` sigue devolviendo `p1`, con `category.active: false`.
5. `createProduct` con `categoryId: c1` → `422 CATEGORY_INACTIVE`.
6. Repetir `deactivateCategory` → `200`, ningún evento nuevo.
7. `activateCategory` — `POST /api/v1/categories/c1/activate` → `200`, `active: true`, un `CategoryUpdated`, y `c1` vuelve a `listPublicCategories`.

**Casos borde**:
- `activateCategory`/`deactivateCategory` sobre uuid inexistente → `404 CATEGORY_NOT_FOUND`.

### FL-CAT-030: borrado de categoría

**Given**: categorías `c1` (sin productos) y `c2` con el producto `p1` en `draft`; marca `b1`.

**When**: `deleteCategory` — `DELETE /api/v1/categories/c1` con el token de `u-admin`

**Then**:
1. `204` sin cuerpo.
2. Un `CategoryDeleted` con `data`: `categoryId: c1`, `name` de `c1`.
3. `getCategory c1` → `404 CATEGORY_NOT_FOUND`.

**Orden de evaluación**:
1. Existe → `CATEGORY_NOT_FOUND` (`404`).
2. Sin productos → `CATEGORY_HAS_PRODUCTS` (`409`).

**Casos borde**:
- `DELETE /api/v1/categories/c2` → `409 CATEGORY_HAS_PRODUCTS`; sin evento.
- `DELETE` otra vez sobre `c1` → `404 CATEGORY_NOT_FOUND`.

### FL-CAT-031: borrar una categoría a la vez que se le asigna un producto

**Given**: categoría `c1` sin productos, marca `b1`.

**When**: a la vez, `deleteCategory c1` (token de `u-admin`) y `createProduct` con `categoryId: c1`, `sku: "RACE-02"`.

**Then**:
1. Exactamente uno: (a) `204` y `422 CATEGORY_REFERENCE_NOT_FOUND`; o (b) `201` y `409 CATEGORY_HAS_PRODUCTS`.
2. Ninguna responde `5xx`.
3. Si `RACE-02` existe, `getCategory c1` responde `200`.

### FL-CAT-040: listado de categorías para la gestión

**Given**: categorías `Zapatos`, `Camisetas`, `Calcetines`, `Abrigos`; `Abrigos` desactivada.

**When**: `listCategories` — `GET /api/v1/categories`

**Then**:
1. `200`; `totalElements: 4`; `items` por `name` ascendente: `Abrigos`, `Calcetines`, `Camisetas`, `Zapatos`.
2. `?active=true` → sin `Abrigos`. `?name=CA` → `Calcetines`, `Camisetas`.
3. `?size=3` → 3 items y `totalPages: 2`; `?size=3&page=1` → `Zapatos`; `?size=3&page=2` → `items: []`; `?size=1000` → `size: 100`.

## Productos — alta y edición

### FL-PRD-001: alta de producto

**Given**: marca `b1` (`Acme`) y categoría `c1` (`Camisetas`), activas y creadas **sin** `description`. Ningún producto.

**When**: `createProduct` — `POST /api/v1/products` con `Idempotency-Key: k-prd-1` y el token de `u-manager`
```json
{
  "sku": "ts-red-m",
  "name": "Camiseta Básica Roja",
  "description": "Algodón orgánico.",
  "price": { "amount": 19.90, "currency": "EUR" },
  "brandId": "b1",
  "categoryId": "c1"
}
```

**Then**:
1. Status `201`, `Location` = `/api/v1/products/{id}` (en adelante `p1`).
2. Cuerpo (proyección de gestión): `id: p1`, `sku: "TS-RED-M"` (normalizado), `name: "Camiseta Básica Roja"`, `description: "Algodón orgánico."`, `price: { amount: 19.90, currency: "EUR" }`, `status: "draft"`, `createdAt` y `updatedAt` (instantes), `createdBy` y `updatedBy` iguales entre sí (el principal de `u-manager`), `brand: { id: b1, name: "Acme", description: null, active: true }`, `category: { id: c1, name: "Camisetas", description: null, active: true }`, `images: []`. Sin `brandId` ni `categoryId`.
3. `getProduct` — `GET /api/v1/products/p1` → `200` con el mismo cuerpo.
4. `productEvents` recibe exactamente un `ProductCreated` con `data`: `productId: p1`, `sku: "TS-RED-M"`, `name: "Camiseta Básica Roja"`, `description: "Algodón orgánico."`, `price: { amount: 19.90, currency: "EUR" }`, `status: "draft"`, `brandId: b1`, `brandName: "Acme"`, `categoryId: c1`, `categoryName: "Camisetas"`, `images: []`.
5. `getPublicProduct p1` → `404 PRODUCT_NOT_FOUND` (un `draft` no existe para el escaparate).
6. Reintento con `k-prd-1` y el mismo cuerpo → `201`, misma `Location`, mismo cuerpo; `listProducts` con `totalElements: 1`; ningún segundo `ProductCreated`.

**Orden de evaluación**:
1. `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. SKU libre → `PRODUCT_SKU_ALREADY_EXISTS` (`409`).
3. Moneda del catálogo → `CURRENCY_NOT_SUPPORTED` (`422`).
4. La marca existe → `BRAND_REFERENCE_NOT_FOUND` (`422`).
5. La categoría existe → `CATEGORY_REFERENCE_NOT_FOUND` (`422`).
6. La marca está activa → `BRAND_INACTIVE` (`422`).
7. La categoría está activa → `CATEGORY_INACTIVE` (`422`).

**Casos borde** (cada uno con clave nueva y con un `sku` libre propio —`TS-B1`, `TS-B2`…— salvo que se diga; el resto del cuerpo como en el `When`):
- Sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`.
- `k-prd-1` con `name` distinto → `409 IDEMPOTENCY_KEY_REUSED`.
- `sku: "TS-RED-M"` (el del `When`) → `409 PRODUCT_SKU_ALREADY_EXISTS`.
- `currency: "USD"` → `422 CURRENCY_NOT_SUPPORTED`.
- `brandId` inexistente → `422 BRAND_REFERENCE_NOT_FOUND`; `categoryId` inexistente → `422 CATEGORY_REFERENCE_NOT_FOUND`.
- **Precedencia**: `sku: "TS-RED-M"` + `currency: "USD"` + `brandId` inexistente → `409 PRODUCT_SKU_ALREADY_EXISTS`. Con un `sku` libre, `currency: "USD"` + `brandId` inexistente → `422 CURRENCY_NOT_SUPPORTED`. Con un `sku` libre, `brandId` y `categoryId` inexistentes → `422 BRAND_REFERENCE_NOT_FOUND`.
- `amount: 19.999` → `400` (escala). `amount: -1` → `400`. `sku: "x"` (1 carácter) o `"TS RED"` (espacio) → `400`. `name` de 161 caracteres → `400`. `price` ausente → `400`.
- En ningún caso borde se publica evento ni aparece producto nuevo en `listProducts`.
- `getProduct` sobre uuid inexistente → `404 PRODUCT_NOT_FOUND`.

**Notas de determinación**: `amount` se compara numéricamente con escala 2.

### FL-PRD-002: dos altas con la misma clave a la vez

**Given**: marca `b1` y categoría `c1` activas.

**When**: dos `createProduct` idénticos (`sku: "TS-RACE"`, `name: "Camiseta Carrera"`) con la misma `Idempotency-Key: k-race` enviados a la vez.

**Then**:
1. Desenlace admisible: o las dos responden `201` con el mismo cuerpo (mismo `id`), o una responde `201` y la otra `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
2. Ninguna responde `PRODUCT_SKU_ALREADY_EXISTS` ni `5xx`.
3. Independiente del ganador: `listProducts?name=Carrera` devuelve **exactamente un** producto (`sku: "TS-RACE"`), y `productEvents` recibe **exactamente un** `ProductCreated` para él.

### FL-PRD-010: edición de producto

**Given**: marcas `b1` (`Acme`) y `b2` (`Zeta`), categorías `c1` y `c2`, todas activas; marca `b3` y categoría `c3` desactivadas; producto `p1` (`TS-01`, `19.90 EUR`, `b1`, `c1`, `description: "Algodón."`) creado con el token de `u-manager`; canal `productEvents` purgado.

**When**: `updateProduct` — `PATCH /api/v1/products/p1` con el token de `u-admin`
```json
{ "price": { "amount": 24.50, "currency": "EUR" }, "brandId": "b2" }
```

**Then**:
1. `200`; proyección de gestión con `price.amount: 24.50`, `brand.id: b2`, `brand.name: "Zeta"`; `name`, `description`, `category` y `sku: "TS-01"` sin cambio; `status: "draft"`.
2. `updatedBy` distinto de `createdBy` (principal de `u-admin` frente al de `u-manager`); `createdBy` sin cambio; `updatedAt` posterior o igual a `createdAt`.
3. `productEvents` recibe exactamente un `ProductUpdated` con la ficha completa tras el cambio (`price.amount: 24.50`, `brandId: b2`, `brandName: "Zeta"`, resto como en el `Given`).
4. `{ "description": null }` → `200` con `description: null` y un `ProductUpdated` con `description: null`.
5. `{ "price": { "amount": 24.50, "currency": "EUR" } }` (sin cambio) → `200`, mismo cuerpo, `updatedAt` y `updatedBy` sin cambio respecto al punto 4 y **ningún** `ProductUpdated`.
6. `getProduct p1` devuelve el estado del punto 4.

**Ramas condicionales**:
- La marca asignada se desactiva después (`deactivateBrand b2`): un `PATCH` con `{ "brandId": "b2", "name": "Camiseta Zeta" }` responde `200` (no cambia la marca, así que no se comprueba su actividad). Un `PATCH` que cambia a la marca inactiva `b3` responde `422 BRAND_INACTIVE`; a la categoría inactiva `c3`, `422 CATEGORY_INACTIVE`.
- Con `p1` publicado (`active`), `{ "price": { "amount": 0, "currency": "EUR" } }` → `422 PRICE_NOT_POSITIVE`. Con `p1` en `draft`, el mismo cuerpo responde `200` con `amount: 0`.

**Orden de evaluación**:
1. `Idempotency-Key` → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Existe → `PRODUCT_NOT_FOUND` (`404`).
3. No retirado → `PRODUCT_RETIRED` (`409`).
4. Marca existe → `BRAND_REFERENCE_NOT_FOUND` (`422`).
5. Categoría existe → `CATEGORY_REFERENCE_NOT_FOUND` (`422`).
6. Marca nueva activa → `BRAND_INACTIVE` (`422`).
7. Categoría nueva activa → `CATEGORY_INACTIVE` (`422`).
8. Precio positivo si `active` → `PRICE_NOT_POSITIVE` (`422`).
9. Moneda del catálogo → `CURRENCY_NOT_SUPPORTED` (`422`).

**Casos borde**:
- uuid inexistente → `404 PRODUCT_NOT_FOUND`; con además `brandId` inexistente → sigue `404` (precedencia 2 sobre 4).
- `brandId` inexistente → `422 BRAND_REFERENCE_NOT_FOUND`; `categoryId` inexistente → `422 CATEGORY_REFERENCE_NOT_FOUND`.
- `currency: "USD"` → `422 CURRENCY_NOT_SUPPORTED`.
- `{ "name": null }`, `{ "price": null }`, `{ "brandId": null }` → `400`.

### FL-PRD-011: dos ediciones a la vez sobre el mismo producto

**Given**: `p1` (`TS-01`, `name: "Original"`), canal `productEvents` purgado.

**When**: a la vez, `updateProduct p1` con `{ "name": "Versión A" }` y `updateProduct p1` con `{ "name": "Versión B" }`, cada una con su propia clave.

**Then**:
1. Al menos una responde `200`. La otra responde `200` o `409 CONCURRENT_MODIFICATION`; nunca `5xx`.
2. `getProduct p1` devuelve `name: "Versión A"` o `"Versión B"`; si solo una respondió `200`, es el `name` de esa.
3. Independiente del ganador: `productEvents` recibe exactamente tantos `ProductUpdated` para `p1` como respuestas `200` hubo.

### FL-PRD-040: listado de productos para la gestión

**Given**: marcas `b1` (`Acme`) y `b2` (`Zeta`), categorías `c1`, `c2`. Productos creados y modificados en este orden (la última escritura de cada uno es la indicada):
1. `p1` `Camiseta Roja`, `10.00`, `b1`, `c1` — publicado.
2. `p2` `Pantalón Azul`, `40.00`, `b2`, `c2` — `draft`.
3. `p3` `Camiseta Verde`, `25.00`, `b2`, `c1` — retirado.
4. Por último, `updateProduct p2` con `{ "description": "Nuevo." }`.

**When**: `listProducts` — `GET /api/v1/products`

**Then**:
1. `200`, `totalElements: 3`, proyección de gestión.
2. Orden `updatedAt` descendente: `p2` (última escritura), `p3`, `p1`.
3. `?status=draft` → `p2`. `?status=retired` → `p3`.
4. `?name=camiseta` → `p3`, `p1`. `?brandId=b2` → `p2`, `p3`. `?categoryId=c1&brandId=b2` → `p3`.
5. `?minPrice=10.00&maxPrice=25.00` → `p3`, `p1` (extremos incluidos).
6. `?categoryId=` con un uuid inexistente → `items: []`, `totalElements: 0`.
7. `?size=1` → `p2`, `totalPages: 3`; `?size=1&page=3` → `items: []`; `?size=999` → `size: 100`.
8. **Coste**: una página de 3 productos con marca, categoría e imágenes embebidas no cuesta más consultas al almacén que una de 1.

**Casos borde**:
- `?minPrice=30&maxPrice=20` → `400 INVALID_PRICE_RANGE`.
- `?minPrice=10.555` → `400`.

**Notas de determinación**: el orden de `p3` frente a `p1` lo fija su última escritura (retirar `p3` fue posterior a publicar `p1`).

## Imágenes

### FL-IMG-001: añadir imágenes y leerlas

**Given**: `p1` en `draft` sin imágenes (marca `b1`, categoría `c1`); canal purgado.

**When**: `addProductImage` — `POST /api/v1/products/p1/images` (multipart) con `Idempotency-Key: k-img-1`, `file` = `front.jpg` (JPEG de 200 KB), `altText: "Frontal"`, `main: false`

**Then**:
1. `200`.
2. Cuerpo: proyección de gestión de `p1` con `images: [ { id: <uuid, en adelante i1>, file: <URL absoluta>, altText: "Frontal", position: 0, main: true } ]` — la primera imagen es principal aunque se pidiera `main: false`.
3. `GET` anónimo sobre `images[0].file` → `200` con content-type `image/jpeg` y los mismos bytes subidos.
4. `productEvents` recibe un `ProductUpdated` de `p1` con `images: [ { imageId: i1, file: <clave, no URL>, altText: "Frontal", position: 0, main: true } ]`.
5. Segunda imagen `back.png` (PNG) con `main: true` → `200`; `images` = `[ {i1, position 0, main false}, {i2, position 1, main true} ]`, ordenadas por `position`.
6. Reintento de la segunda petición con la misma clave → `200` y el mismo cuerpo; `getProduct p1` sigue con **dos** imágenes y no hay un tercer `ProductUpdated`.

### FL-IMG-002: límites de subida

**Given**: `p1` en `draft` con 9 imágenes (posiciones 0–8).

**When**: `addProductImage` con `side.webp` (WebP de 300 KB)

**Then**:
1. `200`; `images` con 10 elementos, la nueva en `position: 9`, `main: false`.

**Orden de evaluación**:
1. `Idempotency-Key` → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Producto existe → `PRODUCT_NOT_FOUND` (`404`).
3. No retirado → `PRODUCT_RETIRED` (`409`).
4. Menos de 10 imágenes → `IMAGE_LIMIT_REACHED` (`409`).
5. Tamaño ≤ 5 MB → `FILE_TOO_LARGE` (`413`).
6. Formato JPEG, PNG o WebP → `UNSUPPORTED_CONTENT_TYPE` (`415`).

**Casos borde**:
- Una undécima imagen → `409 IMAGE_LIMIT_REACHED`; siguen 10 y no hay evento.
- **Precedencia**: undécima imagen de 6 MB → `409 IMAGE_LIMIT_REACHED`.
- Sobre otro producto `p2` con 0 imágenes: un JPEG de 6 MB → `413 FILE_TOO_LARGE`; un GIF de 10 KB → `415 UNSUPPORTED_CONTENT_TYPE`; un PDF de 6 MB → `413 FILE_TOO_LARGE` (precedencia 5 sobre 6). En los tres, `p2` sigue sin imágenes.
- `productId` inexistente → `404 PRODUCT_NOT_FOUND`.
- Sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`.

### FL-IMG-003: reordenar y cambiar la principal

**Given**: `p1` con imágenes `i1` (pos 0, main), `i2` (pos 1), `i3` (pos 2).

**When**: `updateProductImage` — `PATCH /api/v1/products/p1/images/i3` con `{ "position": 0, "altText": "Detalle" }`

**Then**:
1. `200`; `images` = `i3` (pos 0, `altText: "Detalle"`, main false), `i1` (pos 1, main true), `i2` (pos 2): las demás se desplazan sin huecos.
2. Un `ProductUpdated` con ese orden.
3. `{ "main": true }` sobre `i2` → `i2` principal, `i1` deja de serlo; exactamente una `main: true`.
4. `{ "position": 9 }` sobre `i3` → `i3` pasa al final (pos 2), posiciones 0–2 sin huecos.
5. `{ "altText": null }` sobre `i1` → `altText: null`.
6. `{ "position": 2 }` sobre `i3` (ya en 2) → `200` sin cambio y sin evento.

**Orden de evaluación**:
1. `Idempotency-Key` → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Producto existe → `PRODUCT_NOT_FOUND` (`404`).
3. No retirado → `PRODUCT_RETIRED` (`409`).
4. La imagen es de ese producto → `IMAGE_NOT_FOUND` (`404`).
5. No se deja sin principal → `MAIN_IMAGE_REQUIRED` (`409`).

**Casos borde**:
- `{ "main": false }` sobre la principal → `409 MAIN_IMAGE_REQUIRED`.
- `imageId` de otro producto `p2` → `404 IMAGE_NOT_FOUND`.
- `{ "position": 10 }` → `400`.

### FL-IMG-004: eliminar imágenes

**Given**: `p1` en `draft` con `i1` (pos 0, main), `i2` (pos 1), `i3` (pos 2).

**When**: `removeProductImage` — `DELETE /api/v1/products/p1/images/i1`

**Then**:
1. `200` con la ficha: `images` = `i2` (pos 0, **main true**), `i3` (pos 1).
2. Un `ProductUpdated` con esas dos imágenes.
3. `getProduct p1` ya no referencia `i1`, y ninguna imagen restante tiene su `file`.
4. Repetir el `DELETE` sobre `i1` con clave nueva → `404 IMAGE_NOT_FOUND`.
5. Borrar `i2` y `i3` → `p1` queda con `images: []` (en `draft` se permite).

**Orden de evaluación**:
1. `Idempotency-Key` → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Producto existe → `PRODUCT_NOT_FOUND` (`404`).
3. No retirado → `PRODUCT_RETIRED` (`409`).
4. Imagen del producto → `IMAGE_NOT_FOUND` (`404`).
5. No es la última de un `active` → `LAST_IMAGE_OF_ACTIVE_PRODUCT` (`409`) — ver FL-LCY-002.

## Ciclo de vida

### FL-LCY-001: publicar, despublicar y retirar

**Given**: `p1` en `draft` con una imagen `i1` y `19.90 EUR`; canal purgado.

**When**: `publishProduct` — `POST /api/v1/products/p1/publish`

**Then**:
1. `200`; proyección de gestión con `status: "active"`.
2. `productEvents` recibe un `ProductStatusChanged` con `data`: `productId: p1`, `sku`, `previousStatus: "draft"`, `status: "active"`, `reason: null`.
3. `getPublicProduct` — `GET /api/v1/public/products/p1` (anónimo) → `200` con la proyección pública.
4. `publishProduct` otra vez → `409 INVALID_STATE_TRANSITION`, sin evento.
5. `unpublishProduct` — `POST /api/v1/products/p1/unpublish` → `200`, `status: "draft"`, `ProductStatusChanged` con `productId: p1`, `sku` de `p1`, `previousStatus: "active"`, `status: "draft"`, `reason: null`; `getPublicProduct p1` → `404 PRODUCT_NOT_FOUND`.
6. `unpublishProduct` desde `draft` → `409 INVALID_STATE_TRANSITION`.
7. Publicar de nuevo y `retireProduct` — `POST /api/v1/products/p1/retire` con token de `u-admin` y `{ "reason": "Fin de temporada" }` → `200`, `status: "retired"`, `ProductStatusChanged` con `productId: p1`, `sku` de `p1`, `previousStatus: "active"`, `status: "retired"`, `reason: "Fin de temporada"`.
8. `retireProduct` sobre `p1` otra vez → `409 INVALID_STATE_TRANSITION`; `publishProduct` y `unpublishProduct` sobre `p1` → `409 INVALID_STATE_TRANSITION`.
9. `getProduct p1` → `200` con `status: "retired"`, su imagen y todos sus datos; `getPublicProduct p1` → `404`.

**Casos borde**:
- `publishProduct`, `unpublishProduct` y `retireProduct` sobre uuid inexistente → `404 PRODUCT_NOT_FOUND`.

### FL-LCY-002: precondiciones de publicación

**Given**: `p1` en `draft` sin imágenes con `19.90 EUR`; `p2` en `draft` con una imagen y `0.00 EUR`.

**When**: `publishProduct p1`

**Then**:
1. `409 PRODUCT_HAS_NO_IMAGES`; `p1` sigue en `draft` y no hay evento.
2. `publishProduct p2` → `409 PRODUCT_PRICE_NOT_POSITIVE`.
3. Añadir una imagen a `p1` y publicarlo → `200 active`.
4. `removeProductImage` sobre la única imagen de `p1` (`active`) → `409 LAST_IMAGE_OF_ACTIVE_PRODUCT`; la imagen sigue ahí.

**Orden de evaluación** (`publishProduct`):
1. Existe → `PRODUCT_NOT_FOUND` (`404`).
2. Está en `draft` → `INVALID_STATE_TRANSITION` (`409`).
3. Tiene imágenes → `PRODUCT_HAS_NO_IMAGES` (`409`).
4. Precio > 0 → `PRODUCT_PRICE_NOT_POSITIVE` (`409`).

**Casos borde**:
- **Precedencia**: `p3` en `draft`, sin imágenes y a `0.00` → `409 PRODUCT_HAS_NO_IMAGES`.
- **Precedencia**: `p4` retirado desde `draft` sin imágenes → `publishProduct` responde `409 INVALID_STATE_TRANSITION`, no `PRODUCT_HAS_NO_IMAGES`.

### FL-LCY-003: un producto retirado ya no admite cambios

**Given**: `p1` retirado desde `draft` (`retireProduct` sin `reason`) con una imagen `i1`.

**When**: `updateProduct p1` con `{ "name": "Otro" }`

**Then**:
1. `409 PRODUCT_RETIRED`; sin evento.
2. `addProductImage`, `updateProductImage i1` y `removeProductImage i1` sobre `p1` → `409 PRODUCT_RETIRED`.
3. El `ProductStatusChanged` de la retirada llevó `productId: p1`, `sku` de `p1`, `previousStatus: "draft"`, `status: "retired"`, `reason: null`.
4. `getProductForServices p1` con la credencial de máquina del cliente `orders` → `200` con `status: "retired"`.

**Casos borde**:
- **Precedencia**: `updateProduct p1` con `brandId` inexistente → `409 PRODUCT_RETIRED` (guarda 3 antes que la 4).

## Escaparate público

### FL-PUB-001: listado con filtros

**Given**: marcas `b1` (`Acme`) y `b2` (`Zeta`); categorías `c1` (`Camisetas`) y `c2` (`Pantalones`); productos:
- `p1` `Camiseta Básica`, `10.00`, `b1`, `c1`, **publicado**.
- `p2` `Camiseta Técnica`, `25.00`, `b2`, `c1`, **publicado**.
- `p3` `Pantalón Chino`, `40.00`, `b1`, `c2`, **publicado**.
- `p4` `Camiseta Borrador`, `15.00`, `b1`, `c1`, en `draft`.
- `p5` `Camiseta Antigua`, `12.00`, `b1`, `c1`, **retirado**.

**When**: `listPublicProducts` — `GET /api/v1/public/products` sin credencial

**Then**:
1. `200`; `totalElements: 3`; `items` por `name` ascendente: `p1` (`Camiseta Básica`), `p2` (`Camiseta Técnica`), `p3` (`Pantalón Chino`).
2. Cada elemento con la proyección pública: sin `createdAt`, `updatedAt`, `createdBy` ni `updatedBy`; `brand` y `category` como objetos `{ id, name, description, active }`; `images` con URL absoluta.
3. Ni `p4` (`draft`) ni `p5` (`retired`) aparecen con ningún filtro.
4. `?categoryId=c1` → `p1`, `p2`. `?brandId=b1` → `p1`, `p3`. `?categoryId=c1&brandId=b1` → `p1`.
5. `?name=CAMISÉTA` → `p1`, `p2`; `?name=tecnica` → `p2` (contenido, sin mayúsculas ni acentos).
6. `?minPrice=10.00&maxPrice=25.00` → `p1`, `p2` (extremos incluidos). `?minPrice=25.01` → `p3`.
7. `?categoryId=` con un uuid inexistente → `200`, `items: []`, `totalElements: 0`.

**Casos borde**:
- `?minPrice=30&maxPrice=10` → `400 INVALID_PRICE_RANGE`.
- `?maxPrice=9.999` → `400`.

### FL-PUB-002: paginación y coste del escaparate

**Given**: marca `b1`, categoría `c1` y 30 productos publicados, `name` de `Producto 01` a `Producto 30`, cada uno con una imagen.

**When**: `listPublicProducts` sin parámetros

**Then**:
1. `page: 0`, `size: 24`, `totalElements: 30`, `totalPages: 2`; `Producto 01`…`Producto 24`.
2. `?page=1` → `Producto 25`…`Producto 30`. `?page=2` → `items: []`, `totalElements: 30`.
3. `?size=500` → `size: 100` y los 30 productos.
4. **Coste**: servir una página de 24 productos (con su marca, categoría e imágenes embebidas) no cuesta más consultas al almacén que servir una de 2 (`?size=2`): el trabajo no crece con el tamaño de la página.

### FL-PUB-003: ficha pública

**Given**: `p1` publicado, `p2` en `draft`.

**When**: `getPublicProduct` — `GET /api/v1/public/products/p1` sin credencial

**Then**:
1. `200` con la proyección pública de `p1`.
2. `p2` → `404 PRODUCT_NOT_FOUND`; un uuid inexistente → `404 PRODUCT_NOT_FOUND` (indistinguibles).

### FL-PUB-004: menús de filtro

**Given**: categorías `Zapatos`, `Abrigos` (inactiva), `Camisetas`; marcas `Zeta`, `Acme`, `Beta` (inactiva).

**When**: `listPublicCategories` — `GET /api/v1/public/categories` sin credencial

**Then**:
1. `200` con una **lista** (sin sobre de paginación): `Camisetas`, `Zapatos`, cada una `{ id, name, description }` sin `active`.
2. `listPublicBrands` — `GET /api/v1/public/brands` → lista `Acme`, `Zeta`, con la misma forma.

## Superficie servidor-a-servidor

### FL-M2M-001: resolver un producto por id

**Given**: `p1` publicado (`TS-01`, `19.90 EUR`, marca `b1`, categoría `c1`, imagen `i1`) y `p2` retirado.

**When**: `getProductForServices` — `GET /api/v1/internal/products/p1` con la credencial de máquina del cliente `orders`

**Then**:
1. `200` con la proyección M2M completa: `id`, `sku: "TS-01"`, `name`, `description`, `price: { amount: 19.90, currency: "EUR" }`, `status: "active"`, `brand { id: b1, name, description, active }`, `category { id: c1, … }`, `images: [ { id: i1, file: <URL absoluta>, altText, position: 0, main: true } ]`. Sin campos de auditoría.
2. `p2` → `200` con `status: "retired"`.
3. uuid inexistente → `404 PRODUCT_NOT_FOUND`.
4. La misma llamada con la credencial de máquina del cliente `cart` → `200` con el mismo cuerpo.

### FL-M2M-002: resolver productos por lote

**Given**: `p1` publicado (`Zapatilla`), `p2` en `draft` (`Abrigo`), `p3` retirado (`Mochila`).

**When**: `listProductsBatchForServices` — `POST /api/v1/internal/products/batch` con la credencial de máquina del cliente `cart`
```json
{ "ids": ["p1", "p2", "p3", "p1", "00000000-0000-0000-0000-000000000000"] }
```

**Then**:
1. `200` con una **lista** (sin sobre de paginación) de 3 elementos en orden de `name` ascendente: `p2` (`Abrigo`), `p3` (`Mochila`), `p1` (`Zapatilla`), cada uno con la proyección M2M.
2. El id repetido aparece una sola vez; el inexistente se omite sin error.
3. `{ "ids": [] }` → `400`. 101 ids → `400`. `{ "ids": ["no-es-uuid"] }` → `400`.
4. **Coste**: resolver 100 ids no cuesta más consultas al almacén que resolver 2.

### FL-M2M-003: autenticación de la superficie M2M

**Given**: `p1` publicado.

**When**: `getProductForServices p1` sin credencial

**Then**:
1. `401`.
2. Con un token de máquina válido emitido para **otra audiencia** → `403`.
3. `listProductsBatchForServices` sin credencial → `401`; con token de otra audiencia → `403`.

## Idempotencia simultánea

### FL-IDM-001: la misma clave a la vez, operación por operación

**Given**, **por fila y de forma independiente** (cada fila se ejecuta tras el reset, como un flujo propio): el estado mínimo que la operación necesita — marca `b1` activa (o inactiva, donde se dice), categoría `c1` activa (o inactiva), y producto `p1` en `draft` con exactamente dos imágenes, `i1` (pos 0, main) e `i2` (pos 1). **Tras construir el estado, los canales `productEvents` y `taxonomyEvents` se purgan**: los conteos solo miden lo que producen las dos peticiones del `When`.

**When**: dos peticiones idénticas con la misma `Idempotency-Key` enviadas a la vez, para cada operación:

| Operación | Petición | Conteo independiente del ganador |
|---|---|---|
| `createProduct` | alta de `TS-IDM` | exactamente un producto `TS-IDM` y un `ProductCreated` |
| `updateProduct` | `p1` a `name: "Doble"` | exactamente un `ProductUpdated` de `p1` |
| `addProductImage` | `extra.jpg` a `p1` | `p1` con exactamente 3 imágenes y un `ProductUpdated` |
| `updateProductImage` | `i2` a `position: 0` | exactamente un `ProductUpdated`; `i2` en pos 0 |
| `removeProductImage` | borrar `i2` | `p1` con exactamente 1 imagen y un `ProductUpdated` |
| `createBrand` | alta de `Doble` | exactamente una marca `Doble` y un `BrandCreated` |
| `updateBrand` | `b1` a `name: "B-Doble"` | exactamente un `BrandUpdated` |
| `activateBrand` | sobre `b1` inactiva | exactamente un `BrandUpdated` |
| `deactivateBrand` | sobre `b1` activa | exactamente un `BrandUpdated` |
| `createCategory` | alta de `Doble` | exactamente una categoría `Doble` y un `CategoryCreated` |
| `updateCategory` | `c1` a `name: "C-Doble"` | exactamente un `CategoryUpdated` |
| `activateCategory` | sobre `c1` inactiva | exactamente un `CategoryUpdated` |
| `deactivateCategory` | sobre `c1` activa | exactamente un `CategoryUpdated` |

**Then**, en cada fila:
1. O las dos responden con el mismo status y el mismo cuerpo, o una responde el éxito y la otra `409 IDEMPOTENCY_KEY_IN_PROGRESS`; nunca `5xx`.
2. El conteo de la última columna, leído por la API y por el canal, se cumple sea quien sea el ganador.

**Casos borde**: cualquiera de las 13 sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED` y ningún efecto.

## Seguridad

### FL-SEC-001: autorización por operación

**Given**: marca `b1`, categoría `c1`, producto `p1` en `draft` con imagen `i1`, producto `p2` **publicado** con una imagen, y marca `b9` y categoría `c9` sin productos.

**When**: cada operación protegida se llama sin credencial, con el token de `u-none` y, donde aplica, con el de `u-manager`.

**Then**:
1. **Sin credencial → `401`** en las 24 operaciones de gestión: `createProduct`, `updateProduct`, `publishProduct`, `unpublishProduct`, `retireProduct`, `getProduct`, `listProducts`, `addProductImage`, `updateProductImage`, `removeProductImage`, y las siete de marcas y las siete de categorías.
2. **Con `u-none` (sin rol del catálogo) → `403`** en esas mismas 24.
3. **Con rol `catalog-manager` → `403`** en `retireProduct p1`, `deleteBrand b9` y `deleteCategory c9`; y `200`/`201` en el resto (cubierto por los demás flujos).
4. Las cuatro operaciones públicas responden `200` **sin** credencial: `listPublicProducts`, `getPublicProduct` sobre `p2`, `listPublicCategories` y `listPublicBrands`.
5. Las operaciones con idempotencia llamadas sin credencial **y** sin `Idempotency-Key` responden `401`, no `400`.
6. Ninguna llamada rechazada produce efecto ni evento.

### FL-SEC-002: consumo desde el navegador

**Given**: `p1` publicado.

**When**: preflight — `OPTIONS /api/v1/products` sin credencial, desde un origen web permitido por la configuración, con `Access-Control-Request-Method: POST` y `Access-Control-Request-Headers: Authorization, Content-Type, Idempotency-Key`

**Then**:
1. Respuesta de éxito del preflight que admite el método `POST` y las cabeceras `Authorization`, `Content-Type` e `Idempotency-Key`, con `Access-Control-Max-Age: 3600` y **sin** `Access-Control-Allow-Credentials: true`.
2. Petición normal cross-origin: `GET /api/v1/public/products` desde ese origen → `200` con `Access-Control-Allow-Origin` para el origen y `Access-Control-Expose-Headers` que incluye `X-Correlation-Id`, presente en la respuesta.

## Fiabilidad de publicación

### FL-OBX-001: el evento sobrevive a un canal indisponible

**Given**: marca `b1` y categoría `c1`; el canal `productEvents` sin mensajes y el canal de eventos **indisponible**.

**When**: `createProduct` con `sku: "OBX-01"`

**Then**:
1. `201` con el cuerpo completo de la creación: la indisponibilidad del canal no llega al cliente.
2. `getProduct` devuelve el producto en `draft`.
3. `productEvents` no ha recibido ningún mensaje todavía.

**When**: el canal vuelve a estar disponible

**Then**:
4. En ≤ 10 s `productEvents` recibe **exactamente un** `ProductCreated` de `OBX-01`, con su payload completo y el `correlationId` de la petición del paso anterior.
5. Ningún evento abandonado.

### FL-OBX-002: el evento que el relay abandona no se pierde en silencio

**Given**: el canal indisponible y un `createProduct` (`sku: "OBX-02"`) ya confirmado, con su `ProductCreated` pendiente de salir.

**When**: se agota el presupuesto de reintentos de ese evento

**Then**:
1. El servidor informa de **un** evento abandonado.
2. Restablecido el canal, ese `ProductCreated` **no** se publica.
3. Y el canal no recibe ninguna otra cosa.

## Lo que no tiene escenario, y por qué

- **El `403` por scope insuficiente en la superficie M2M.** Los dos clientes declarados (`orders`, `cart`) tienen `product:read`, el único scope que exige esa superficie, así que no existe una credencial de máquina válida sin él. La autenticación M2M queda cubierta por `401` y por el token de otra audiencia (FL-M2M-003). Si mañana se declara un cliente con otro scope, este escenario pasa a ser obligatorio.
- **El borrado del binario huérfano.** Si la confirmación de una imagen falla después de subir el binario, o si el borrado del binario falla después de confirmar, queda un archivo sin referencia. El diseño lo acepta a propósito (nunca una imagen rota); no hay una puerta pública que provoque el fallo a mitad ni una operación que liste binarios huérfanos, así que su verificación es estática.
