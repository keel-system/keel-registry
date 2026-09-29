# catalog — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/catalog v0.1.1. Contrato de validación para la fase de generación.

## Convenciones de determinación

- **Ausencia vs nulo** (declarado en el manifiesto, `conventions.nulls: include`): un campo sin valor
  viaja como `null`, en respuestas y en eventos (`description: null`, `altText: null`,
  `firstPublishedAt: null`). Una colección vacía viaja como `[]`.
- **Fechas**: todo instante (`createdAt`, `updatedAt`, `firstPublishedAt`) es ISO-8601 en UTC. Se
  verifica por forma y por relación (`updatedAt` ≥ `createdAt`; un valor que «no cambia» es igual
  al leído antes), nunca por valor exacto.
- **Identificadores generados** (`id` de marca, categoría, producto e imagen): forma uuid, verificados
  por reutilización simbólica (`b1`, `c1`, `p1`, `i1`…), nunca por valor literal.
- **Autoría**: `createdBy` y `updatedBy` son el identificador del usuario autenticado que hizo la
  petición (el sujeto de su credencial). Los escenarios nombran al usuario por su rol (`editor` =
  usuario con rol `catalog-editor`; `admin` = usuario con rol `catalog-admin`) y afirman que el campo
  es el identificador **de ese** usuario.
- **Dinero**: `price.amount` con escala 2; un importe de entrada con más de 2 decimales se **rechaza**
  con `400` (declarado: `Money.amount` `scalePolicy: reject`), no se redondea. `price.currency` es
  el parámetro `currency`, que en pruebas vale `EUR`. Los importes se escriben como número JSON
  (`19.90`).
- **Mayúsculas y acentos** (declarado en el YAML): los nombres de marca y categoría son únicos sin
  distinguir mayúsculas ni acentos (`compare: ignore-case-accents`: `Acme`, `ACME` y `Acmé`
  colisionan). El sku se compara exacto tras normalizarlo a mayúsculas (`ab-1` y `AB-1` son el mismo
  sku). El filtro `name` casa por contenido ignorando mayúsculas y acentos (`match: contains`,
  `compare: ignore-case-accents`); el filtro `sku` de gestión casa por prefijo sin distinguir
  mayúsculas.
- **Orden por nombre**: `name` ascendente con desempate por `id`. El orden relativo entre nombres que
  solo difieren en mayúsculas o acentos **no** es contrato; los escenarios usan nombres sin esa
  ambigüedad (todos con inicial mayúscula y sin acento en la primera letra distinta).
- **Imágenes**: dentro de un producto viajan siempre ordenadas por `position` ascendente. Cada imagen
  lleva `id`, `file`, `altText`, `position`, `main` y `productId` (el id de su producto). En las
  respuestas HTTP `file` es una **URL absoluta** (bucket `public`); en los eventos es la **key** del
  objeto.
- **Cuerpo de error**: forma fija del generador —`{timestamp, status, error, code, message,
  details}` más `correlationId`—. Los escenarios fijan solo `code` y status. El `message` es texto
  libre, no contractual.
- **Errores del mecanismo** (canónicos de `framework-errors.md`): entrada que no cumple las cotas →
  `400 VALIDATION_ERROR`; sin credencial → `401 UNAUTHENTICATED`; sin permiso, sin scope o token de
  otra audiencia → `403 ACCESS_DENIED`; transición no declarada → `409 INVALID_STATE_TRANSITION`;
  misma clave de idempotencia a la vez → `409 IDEMPOTENCY_KEY_IN_PROGRESS`; misma clave con otro
  cuerpo → `409 IDEMPOTENCY_KEY_REUSED`; versión obsoleta → `409 CONCURRENT_MODIFICATION`.
- **Idempotencia**: la clave viaja en la cabecera `Idempotency-Key` y dura 24 h. Sin cabecera, las
  cuatro altas responden `400 IDEMPOTENCY_KEY_REQUIRED`.
- **Versión**: los cuatro PATCH exigen `version`, la `lockVersion` que el cliente leyó. Toda
  escritura confirmada sube la `lockVersion` de su raíz (en Product, también las de imágenes); un
  cambio nulo no la sube.
- **Paginación** (sobre canónico): `{ items, page, size, totalElements, totalPages }`, `page` base 0,
  `size` por defecto 20, tope 100 (un `size` mayor se recorta a 100). Página vacía: `items: []`,
  `totalElements: 0`, `totalPages: 0`.
- **Eventos**: envoltura Keel `{ metadata, data }` en el canal `productEvents`. Los escenarios
  afirman `metadata.eventType` y el `data` completo; `metadata.eventId` y `occurredAt` se verifican
  por forma. El `data` de `ProductCreated` y `ProductUpdated` tiene siempre los mismos diez campos:
  `productId, sku, name, description, price, status, brand{id,name}, category{id,name}, images[],
  version`, donde `version` es la `lockVersion` del producto tras el cambio.
- **Rutas**: todas bajo `/api/v1`.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|-----------|--------|------------|
| createBrand | FL-BRD-001, FL-BRD-002, FL-BRD-040 | usuarios |
| updateBrand | FL-BRD-010 | usuarios |
| deactivateBrand | FL-BRD-020, FL-PRD-003 | usuarios |
| reactivateBrand | FL-BRD-020 | usuarios |
| getBrand | FL-BRD-001, FL-BRD-010, FL-BRD-040 | usuarios |
| listBrands | FL-BRD-030 | usuarios |
| createCategory | FL-CAT-001, FL-CAT-002 | usuarios |
| updateCategory | FL-CAT-010 | usuarios |
| deactivateCategory | FL-CAT-020, FL-PRD-003 | usuarios |
| reactivateCategory | FL-CAT-020 | usuarios |
| getCategory | FL-CAT-001, FL-CAT-010 | usuarios |
| listCategories | FL-CAT-030 | usuarios |
| createProduct | FL-PRD-001, FL-PRD-002, FL-PRD-003 | usuarios |
| updateProduct | FL-PRD-010, FL-PRD-020 | usuarios |
| publishProduct | FL-PRD-020, FL-PRD-021 | usuarios |
| unpublishProduct | FL-PRD-020 | usuarios |
| retireProduct | FL-PRD-020, FL-PRD-021, FL-PRD-040 | usuarios |
| getProduct | FL-PRD-001, FL-PRD-020 | usuarios |
| searchProducts | FL-PRD-030 | usuarios |
| addProductImage | FL-IMG-001, FL-IMG-002, FL-IMG-003 | usuarios |
| updateProductImage | FL-IMG-010 | usuarios |
| removeProductImage | FL-IMG-020 | usuarios |
| searchPublishedProducts | FL-SHP-001 | público |
| getPublishedProduct | FL-SHP-010 | público |
| listPublicBrands | FL-SHP-020 | público |
| listPublicCategories | FL-SHP-020 | público |
| getProductForServices | FL-M2M-001 | **servidores (M2M)** |
| getProductsBatch | FL-M2M-010 | **servidores (M2M)** |

Transversales: el outbox (FL-EVT-001, FL-EVT-002) y CORS (FL-SEC-001).

---

## Marcas

### FL-BRD-001: alta de marca, lectura y colisión de nombre

**Given**: no existe ninguna marca. `admin` es un usuario con rol `catalog-admin`.

**When**: `createBrand` — `POST /api/v1/admin/brands` como `admin`, cabecera `Idempotency-Key: k-brd-1`
```json
{ "name": "  Acme  ", "description": "Calzado deportivo" }
```

**Then**:
1. Status `201`.
2. Cabecera `Location` = `/api/v1/admin/brands/{b1}`, donde `b1` es el `id` devuelto.
3. El cuerpo es exactamente `{ id: b1, name: "Acme", description: "Calzado deportivo", active: true,
   lockVersion: <int>, createdAt: <instante>, updatedAt: <instante>, createdBy: <id de admin>,
   updatedBy: <id de admin> }`: el nombre llega recortado y no hay ningún otro campo.
4. `getBrand` — `GET /api/v1/admin/brands/{b1}` como `admin` responde `200` con el mismo cuerpo.
5. Repetir la misma petición con la misma `Idempotency-Key: k-brd-1` responde `201`, el mismo cuerpo
   (mismo `b1`) y la misma cabecera `Location`; `listBrands` sigue devolviendo una sola marca
   (`totalElements: 1`).
6. La misma clave `k-brd-1` con el cuerpo `{ "name": "Otra" }` responde `409 IDEMPOTENCY_KEY_REUSED`
   y no crea nada.

**Orden de evaluación**:
1. Cabecera `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Cotas del input (`name` 1..80, `description` ≤ 2000) → `VALIDATION_ERROR` (`400`).
3. Nombre libre sin distinguir mayúsculas ni acentos → `BRAND_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- Sin cabecera `Idempotency-Key` y con `name: "Acme"` (duplicado) → `400 IDEMPOTENCY_KEY_REQUIRED`:
  la guarda 1 precede a la 3.
- Con clave nueva `k-brd-2` y `{ "name": "ACMÉ" }` → `409 BRAND_NAME_ALREADY_EXISTS`.
- `{ "name": "   " }` (vacío tras recortar) o `name` de 81 caracteres → `400 VALIDATION_ERROR`.
- `getBrand` de un uuid que no existe → `404 BRAND_NOT_FOUND`.

### FL-BRD-002: dos altas de marca con la misma clave a la vez

**Given**: no existe ninguna marca.

**When**: `createBrand` dos veces **a la vez**, las dos como `admin`, las dos con
`Idempotency-Key: k-race-b` y el cuerpo `{ "name": "Zeta" }`.

**Then**:
1. O las dos responden `201` con el mismo cuerpo (mismo `id`), o una responde `201` y la otra
   `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `listBrands` con `name=Zeta` devuelve **exactamente una** marca, sea cual sea la petición ganadora.

### FL-BRD-010: edición de marca con versión

**Given**: `admin` crea la marca `b1` (`name: "Acme"`, `description: null`) y la marca `b2`
(`name: "Bolt"`). Lee `b1` y obtiene `lockVersion: v1`.

**When**: `updateBrand` — `PATCH /api/v1/admin/brands/{b1}` como `admin`
```json
{ "version": v1, "description": "Marca histórica" }
```

**Then**:
1. Status `200`; el cuerpo es la marca completa con `name: "Acme"` (no enviado, se conserva),
   `description: "Marca histórica"`, `active: true`, `lockVersion` mayor que `v1`, `createdAt` y
   `createdBy` iguales a los del alta, `updatedAt` ≥ el anterior, `updatedBy`: id de `admin`.
2. `getBrand(b1)` devuelve ese mismo cuerpo.
3. Un segundo `PATCH` con `{ "version": v1, "name": "Acme Sports" }` (la versión vieja) responde
   `409 CONCURRENT_MODIFICATION`, y `getBrand(b1)` sigue con `name: "Acme"`.
4. `PATCH` con la versión vigente y `{ "description": null }` responde `200` con `description: null`.

**Orden de evaluación**:
1. Cotas del input → `VALIDATION_ERROR` (`400`).
2. La marca existe → `BRAND_NOT_FOUND` (`404`).
3. `version` es la vigente → `CONCURRENT_MODIFICATION` (`409`).
4. El nombre nuevo no lo tiene otra marca → `BRAND_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `PATCH` sin `version` → `400 VALIDATION_ERROR`.
- `PATCH` sobre un uuid inexistente con cualquier `version` → `404 BRAND_NOT_FOUND`.
- `PATCH` de `b1` con versión vigente y `{ "name": "bolt" }` → `409 BRAND_NAME_ALREADY_EXISTS`.
- `PATCH` de `b1` con versión **vieja** y `{ "name": "bolt" }` → `409 CONCURRENT_MODIFICATION`: la
  guarda 3 precede a la 4.
- `{ "name": null }` → `400 VALIDATION_ERROR`.

### FL-BRD-020: desactivar y reactivar una marca

**Given**: `admin` crea la marca `b1` (`name: "Acme"`).

**When**: `deactivateBrand` — `POST /api/v1/admin/brands/{b1}/deactivate` como `admin`, sin cuerpo.

**Then**:
1. Status `200`; el cuerpo es la marca completa con `active: false` y `lockVersion` mayor que la del alta.
2. `listPublicBrands` (`GET /api/v1/brands`, sin credencial) no incluye `b1`.
3. Repetir `deactivateBrand(b1)` responde `200` con el mismo cuerpo y la misma `lockVersion` (sin cambios).
4. `reactivateBrand` — `POST /api/v1/admin/brands/{b1}/reactivate` responde `200` con `active: true`,
   y `listPublicBrands` vuelve a incluir `b1`.
5. Repetir `reactivateBrand(b1)` responde `200` sin cambios.

**Casos borde**:
- `deactivateBrand` o `reactivateBrand` de un uuid inexistente → `404 BRAND_NOT_FOUND`.

### FL-BRD-030: listado de marcas de gestión

**Given**: `admin` crea 21 marcas con nombres `Marca 01` … `Marca 21` y desactiva `Marca 02`.

**When**: `listBrands` — `GET /api/v1/admin/brands` como `admin`, sin parámetros.

**Then**:
1. Status `200`; `page: 0`, `size: 20`, `totalElements: 21`, `totalPages: 2`.
2. `items` trae 20 marcas en orden `Marca 01` … `Marca 20`, cada una con la proyección completa de
   gestión (incluidos `lockVersion` y los cuatro campos de auditoría); `Marca 02` con `active: false`.
3. `?page=1` trae solo `Marca 21`.
4. `?page=5` trae `items: []`, `totalElements: 21`, `totalPages: 2`, `page: 5`, `size: 20`.
5. `?size=500` responde con `size: 100` y las 21 marcas.
6. `?active=false` trae solo `Marca 02`; `?name=rca 1` trae `Marca 10` … `Marca 19` en orden.
7. `?name=Inexistente` trae la página vacía canónica: `items: []`, `totalElements: 0`, `totalPages: 0`.

### FL-BRD-040: autorización de la gestión de marcas

**Given**: `admin` crea la marca `b1`. `editor` es un usuario con rol `catalog-editor`.

**When / Then**:
1. `createBrand` sin credencial → `401 UNAUTHENTICATED`.
2. `createBrand` como `editor` (con `Idempotency-Key`) → `403 ACCESS_DENIED`; no se crea nada.
3. `updateBrand`, `deactivateBrand` y `reactivateBrand` de `b1` como `editor` → `403 ACCESS_DENIED`.
4. `getBrand(b1)` y `listBrands` como `editor` → `200` (tiene `catalog:read`).
5. `getBrand(b1)` y `listBrands` sin credencial → `401 UNAUTHENTICATED`.

---

## Categorías

### FL-CAT-001: alta de categoría, lectura y colisión de nombre

**Given**: no existe ninguna categoría.

**When**: `createCategory` — `POST /api/v1/admin/categories` como `admin`, `Idempotency-Key: k-cat-1`
```json
{ "name": "Calzado", "description": null }
```

**Then**:
1. Status `201`; cabecera `Location` = `/api/v1/admin/categories/{c1}`.
2. El cuerpo es exactamente `{ id: c1, name: "Calzado", description: null, active: true,
   lockVersion, createdAt, updatedAt, createdBy: <id de admin>, updatedBy: <id de admin> }`.
3. `getCategory` — `GET /api/v1/admin/categories/{c1}` responde `200` con el mismo cuerpo.
4. Reintento con la misma clave → `201`, mismo cuerpo, misma `Location`; sigue existiendo una sola.

**Orden de evaluación**:
1. `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Cotas del input → `VALIDATION_ERROR` (`400`).
3. Nombre libre → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- Sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`.
- Clave nueva y `{ "name": "CALZADO" }` → `409 CATEGORY_NAME_ALREADY_EXISTS`.
- `getCategory` de un uuid inexistente → `404 CATEGORY_NOT_FOUND`.
- `createCategory` como `editor` → `403 ACCESS_DENIED`; sin credencial → `401 UNAUTHENTICATED`.

### FL-CAT-002: dos altas de categoría con la misma clave a la vez

**Given**: no existe ninguna categoría.

**When**: `createCategory` dos veces **a la vez** como `admin` con `Idempotency-Key: k-race-c` y
`{ "name": "Hogar" }`.

**Then**:
1. O las dos responden `201` con el mismo `id`, o una `201` y la otra `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `listCategories?name=Hogar` devuelve **exactamente una** categoría.

### FL-CAT-010: edición de categoría con versión

**Given**: `admin` crea `c1` (`name: "Calzado"`) y `c2` (`name: "Hogar"`); lee `c1` con `lockVersion: v1`.

**When**: `updateCategory` — `PATCH /api/v1/admin/categories/{c1}` con `{ "version": v1, "name": "Zapatos" }`.

**Then**:
1. Status `200`; cuerpo completo con `name: "Zapatos"`, `description: null`, `lockVersion` > `v1`,
   `updatedBy`: id de `admin`.
2. `getCategory(c1)` devuelve el mismo cuerpo.
3. Otro `PATCH` con `version: v1` → `409 CONCURRENT_MODIFICATION`; `c1` sigue llamándose `Zapatos`.

**Orden de evaluación**:
1. Cotas del input → `VALIDATION_ERROR` (`400`).
2. La categoría existe → `CATEGORY_NOT_FOUND` (`404`).
3. `version` vigente → `CONCURRENT_MODIFICATION` (`409`).
4. Nombre libre → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).

**Casos borde**:
- Versión vigente y `{ "name": "hogar" }` → `409 CATEGORY_NAME_ALREADY_EXISTS`.
- Uuid inexistente → `404 CATEGORY_NOT_FOUND`.
- `updateCategory` como `editor` → `403 ACCESS_DENIED`.

### FL-CAT-020: desactivar y reactivar una categoría

**Given**: `admin` crea `c1` (`name: "Calzado"`).

**When**: `deactivateCategory` — `POST /api/v1/admin/categories/{c1}/deactivate`, sin cuerpo.

**Then**:
1. Status `200`; `active: false`.
2. `listPublicCategories` (`GET /api/v1/categories`, sin credencial) no incluye `c1`.
3. Repetir → `200` sin cambios (misma `lockVersion`).
4. `reactivateCategory` — `POST /api/v1/admin/categories/{c1}/reactivate` → `200`, `active: true`;
   `listPublicCategories` vuelve a incluirla. Repetir → `200` sin cambios.

**Casos borde**:
- `deactivateCategory` / `reactivateCategory` de un uuid inexistente → `404 CATEGORY_NOT_FOUND`.

### FL-CAT-030: listado de categorías de gestión

**Given**: `admin` crea `Calzado`, `Deporte` y `Hogar`, y desactiva `Deporte`.

**When**: `listCategories` — `GET /api/v1/admin/categories` como `editor`.

**Then**:
1. Status `200`; `totalElements: 3`, `totalPages: 1`, `items` en orden `Calzado`, `Deporte`, `Hogar`,
   con la proyección completa de gestión.
2. `?active=true` trae `Calzado`, `Hogar`; `?name=calz` trae `Calzado`.
3. `?page=1` trae la página vacía (`items: []`, `totalElements: 3`, `totalPages: 1`).
4. `?size=1` trae solo `Calzado` con `totalPages: 3`; `?size=1&page=1` trae `Deporte`.

---

## Productos

### FL-PRD-001: alta de producto, evento y lectura de gestión

**Given**: `admin` crea la marca `b1` (`Acme`) y la categoría `c1` (`Calzado`). `editor` es un usuario
con rol `catalog-editor`. El canal `productEvents` está vacío.

**When**: `createProduct` — `POST /api/v1/admin/products` como `editor`, `Idempotency-Key: k-prd-1`
```json
{ "sku": "ab-100", "name": " Zapatilla Runner ", "description": null,
  "price": { "amount": 59.90, "currency": "EUR" }, "brandId": "b1", "categoryId": "c1" }
```

**Then**:
1. Status `201`; cabecera `Location` = `/api/v1/admin/products/{p1}`.
2. El cuerpo es exactamente: `id: p1`, `sku: "AB-100"`, `name: "Zapatilla Runner"`,
   `description: null`, `price: { amount: 59.90, currency: "EUR" }`, `status: "draft"`,
   `firstPublishedAt: null`, `lockVersion`, `createdAt`, `updatedAt`, `createdBy` y `updatedBy` (id
   de `editor`), `brand` (objeto con la proyección completa de gestión de `b1`: `id, name,
   description, active, lockVersion, createdAt, updatedAt, createdBy, updatedBy`), `category` (ídem
   para `c1`) e `images: []`. No trae `brandId` ni `categoryId` ni ningún otro campo.
3. `getProduct` — `GET /api/v1/admin/products/{p1}` como `editor` responde `200` con el mismo cuerpo.
4. El canal `productEvents` recibe **un** `ProductCreated` con `data`: `productId: p1`,
   `sku: "AB-100"`, `name: "Zapatilla Runner"`, `description: null`,
   `price: { amount: 59.90, currency: "EUR" }`, `status: "draft"`,
   `brand: { id: b1, name: "Acme" }`, `category: { id: c1, name: "Calzado" }`, `images: []`,
   `version`: la `lockVersion` del cuerpo; y `metadata.source: "catalog"`.
5. Reintentar la misma petición con `k-prd-1` → `201`, mismo cuerpo, misma `Location`, y **ningún**
   `ProductCreated` más en el canal.

**Orden de evaluación**:
1. `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Cotas del input (patrón del sku, `name` 1..160, `description` ≤ 4000, `price` con escala 2 y
   currency de 3 letras) → `VALIDATION_ERROR` (`400`).
3. `price.currency` es la moneda del catálogo → `CURRENCY_NOT_SUPPORTED` (`422`).
4. La marca existe → `BRAND_REFERENCE_NOT_FOUND` (`422`); está activa → `BRAND_INACTIVE` (`422`).
5. La categoría existe → `CATEGORY_REFERENCE_NOT_FOUND` (`422`); está activa → `CATEGORY_INACTIVE` (`422`).
6. El sku normalizado está libre → `PRODUCT_SKU_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `price.amount: 59.999` → `400 VALIDATION_ERROR` (escala que se rechaza, no se redondea).
- `price.amount: 0` → `201` (un draft admite precio cero; solo impide publicar).
- `sku: "-x"` (no casa el patrón) → `400 VALIDATION_ERROR`.
- `getProduct` de un uuid inexistente → `404 PRODUCT_NOT_FOUND`.
- `createProduct` sin credencial → `401 UNAUTHENTICATED`.

### FL-PRD-002: dos altas de producto con la misma clave a la vez

**Given**: `admin` crea `b1` y `c1`. Canal `productEvents` vacío.

**When**: `createProduct` dos veces **a la vez** como `editor` con `Idempotency-Key: k-race-p` y el mismo
cuerpo (`sku: "RACE-1"`, `name: "Carrera"`, `price 10.00 EUR`, `b1`, `c1`).

**Then**:
1. O las dos responden `201` con el mismo `id`, o una `201` y la otra `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `searchProducts?sku=RACE-1` devuelve **exactamente un** producto.
3. El canal recibe **exactamente un** `ProductCreated` con `sku: "RACE-1"`.

### FL-PRD-003: referencias, moneda y unicidad en el alta

**Given**: `admin` crea `b1` (`Acme`), `b2` (`Bolt`), `c1` (`Calzado`), `c2` (`Hogar`); desactiva `b2`
(`deactivateBrand`) y `c2` (`deactivateCategory`). `editor` crea `p1` con `sku: "AB-100"`, `b1`, `c1`. El canal
`productEvents` estaba vacío antes del alta de `p1`.
Cada petición de abajo lleva una `Idempotency-Key` nueva.

**When / Then** (`createProduct` como `editor`, con `name: "X"`, `price 10.00 EUR`, `brandId: b1`,
`categoryId: c1` y un sku libre, salvo lo que cada paso cambia):
1. `brandId` de un uuid inexistente → `422 BRAND_REFERENCE_NOT_FOUND`.
2. `categoryId` de un uuid inexistente → `422 CATEGORY_REFERENCE_NOT_FOUND`.
3. `brandId: b2` (inactiva) → `422 BRAND_INACTIVE`.
4. `categoryId: c2` (inactiva) → `422 CATEGORY_INACTIVE`.
5. `price.currency: "USD"` → `422 CURRENCY_NOT_SUPPORTED`.
6. `sku: "ab-100"` → `409 PRODUCT_SKU_ALREADY_EXISTS` (misma tras normalizar).
7. Precedencia: `currency: "USD"` **y** `brandId: b2` → `422 CURRENCY_NOT_SUPPORTED`.
8. Precedencia: `brandId: b2` **y** `categoryId` inexistente → `422 BRAND_INACTIVE`.
9. Precedencia: `categoryId: c2` **y** `sku: "AB-100"` → `422 CATEGORY_INACTIVE`.
10. Ninguna de las nueve crea producto: `searchProducts` sigue con `totalElements: 1`, y el canal solo
    tiene el `ProductCreated` de `p1`.

### FL-PRD-010: edición de producto

**Given**: `admin` crea `b1` (`Acme`), `b2` (`Bolt`), `c1` (`Calzado`). `editor` crea `p1`
(`sku: "AB-100"`, `name: "Runner"`, `price 59.90 EUR`, `b1`, `c1`) y `p2` (`sku: "AB-200"`). Lee `p1`
con `lockVersion: v1`. El canal se purga.

**When**: `updateProduct` — `PATCH /api/v1/admin/products/{p1}` como `editor`
```json
{ "version": v1, "sku": "ab-101", "name": "Runner Pro", "price": { "amount": 64.90, "currency": "EUR" }, "brandId": "b2" }
```

**Then**:
1. Status `200`; cuerpo completo de gestión con `sku: "AB-101"`, `name: "Runner Pro"`,
   `price: { amount: 64.90, currency: "EUR" }`, `brand` = objeto de `b2`, `category` = `c1`,
   `status: "draft"`, `firstPublishedAt: null`, `lockVersion: v2` > `v1`, `updatedBy`: id de `editor`.
2. El canal recibe **un** `ProductUpdated` con `data` completo: `productId: p1`, `sku: "AB-101"`,
   `name: "Runner Pro"`, `description: null`, `price: { amount: 64.90, currency: "EUR" }`, `brand: { id: b2, name: "Bolt" }`, `category: { id: c1, name: "Calzado" }`,
   `images: []`, `status: "draft"`, `version: v2`.
3. `PATCH` con `{ "version": v2, "name": "Runner Pro" }` (nada cambia) → `200` con el mismo cuerpo,
   la misma `lockVersion: v2`, y **ningún** `ProductUpdated` nuevo.
4. `PATCH` con `{ "version": v1, "name": "Otro" }` → `409 CONCURRENT_MODIFICATION`; `p1` no cambia.
5. `admin` desactiva `b2`. `PATCH` de `p1` con `{ "version": v2, "brandId": "b2", "name": "Runner Max" }`
   → `200` (reasignar la marca actual no falla aunque esté inactiva), `name: "Runner Max"`.
6. `PATCH` de `p1` con la versión vigente y `{ "brandId": "b1" }` → `200`, `brand` = `b1`.
7. `admin` crea la marca `b3` (`Ciclo`) y la desactiva; `PATCH` de `p1` con la versión vigente y
   `{ "brandId": "b3" }` → `422 BRAND_INACTIVE`, y `p1` conserva `brand` = `b1`.

**Orden de evaluación**:
1. Cotas del input (`version` presente, patrón de sku, longitudes, escala de `price`) → `VALIDATION_ERROR` (`400`).
2. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
3. No está retirado → `PRODUCT_RETIRED` (`409`).
4. `version` vigente → `CONCURRENT_MODIFICATION` (`409`).
5. Si cambia el sku: el producto nunca se publicó → `PRODUCT_SKU_IMMUTABLE` (`409`); el sku está libre
   → `PRODUCT_SKU_ALREADY_EXISTS` (`409`).
6. `price.currency` es la moneda del catálogo → `CURRENCY_NOT_SUPPORTED` (`422`).
7. Si está publicado: `price.amount` > 0 → `PRODUCT_PRICE_NOT_POSITIVE` (`409`).
8. Marca nueva existe y está activa → `BRAND_REFERENCE_NOT_FOUND` / `BRAND_INACTIVE` (`422`).
9. Categoría nueva existe y está activa → `CATEGORY_REFERENCE_NOT_FOUND` / `CATEGORY_INACTIVE` (`422`).

**Ramas condicionales**:
- Un campo ausente conserva su valor; `description: null` lo vacía; `name`, `price`, `brandId`,
  `categoryId` o `sku` a `null` → `400 VALIDATION_ERROR`.
- `BRAND_INACTIVE` / `CATEGORY_INACTIVE` solo cuando la referencia cambia.

**Casos borde**:
- Versión vigente y `{ "sku": "ab-200" }` → `409 PRODUCT_SKU_ALREADY_EXISTS`.
- Versión vigente y `{ "categoryId": <uuid inexistente> }` → `422 CATEGORY_REFERENCE_NOT_FOUND`.
- Versión vigente y `{ "brandId": <uuid inexistente> }` → `422 BRAND_REFERENCE_NOT_FOUND`.
- `admin` crea `c3` y la desactiva; versión vigente y `{ "categoryId": "c3" }` → `422 CATEGORY_INACTIVE`.
- Versión vigente y `{ "price": { "amount": 5.00, "currency": "USD" } }` → `422 CURRENCY_NOT_SUPPORTED`.
- `PATCH` sin `version` → `400 VALIDATION_ERROR`.
- Uuid inexistente → `404 PRODUCT_NOT_FOUND`.
- Precedencia: versión **vieja** y `sku: "ab-200"` → `409 CONCURRENT_MODIFICATION` (guarda 4 antes que 5).


### FL-PRD-020: ciclo de vida completo de un producto

**Given**: `admin` crea `b1` (`Acme`) y `c1` (`Calzado`). `editor` crea `p1` (`sku: "LC-1"`,
`name: "Ciclo"`, `price 0.00 EUR`, `b1`, `c1`). El canal se purga.

**When / Then** (todo como `editor`, salvo la retirada, como `admin`):
1. `publishProduct` — `POST /api/v1/admin/products/{p1}/publish`, sin cuerpo → `409 PRODUCT_HAS_NO_IMAGES`
   (sin imágenes y con precio cero: la guarda de imágenes precede a la de precio).
2. `addProductImage` sube una imagen JPEG válida (`Idempotency-Key: k-lc-img`) → `201`.
3. `publishProduct(p1)` → `409 PRODUCT_PRICE_NOT_POSITIVE`.
4. `updateProduct` con la versión vigente y `price 25.00 EUR` → `200`, sigue `draft`.
5. `admin` desactiva `b1`. `publishProduct(p1)` → `200`: `status: "active"`, `firstPublishedAt` es un
   instante, `lockVersion` sube. El canal recibe un `ProductUpdated` con `status: "active"` y la
   imagen en `images` (`imageId`, `file` = key, `altText: null`, `position: 0`, `main: true`).
   (Una marca inactiva no impide publicar.)
6. `getProduct(p1)` devuelve `status: "active"` y el mismo `firstPublishedAt`.
7. `publishProduct(p1)` otra vez → `409 INVALID_STATE_TRANSITION`; ningún evento nuevo.
8. `updateProduct` con la versión vigente y `price 0.00 EUR` → `409 PRODUCT_PRICE_NOT_POSITIVE`.
9. `updateProduct` con la versión vigente y `sku: "LC-2"` → `409 PRODUCT_SKU_IMMUTABLE`;
   con `sku: "lc-1"` (el mismo) → `200` sin cambios.
10. `unpublishProduct` — `POST /api/v1/admin/products/{p1}/unpublish`, sin cuerpo → `200`,
    `status: "draft"`, `firstPublishedAt` **igual** al del paso 5. Un `ProductUpdated` con `status: "draft"`.
11. `unpublishProduct(p1)` otra vez → `409 INVALID_STATE_TRANSITION`.
12. `updateProduct` con la versión vigente y `sku: "LC-2"` sigue dando `409 PRODUCT_SKU_IMMUTABLE` (ya se publicó una vez).
13. `retireProduct` — `POST /api/v1/admin/products/{p1}/retire` como `admin`, sin cuerpo → `200`,
    `status: "retired"`. Un `ProductUpdated` con `status: "retired"`.
14. `retireProduct(p1)` otra vez → `409 INVALID_STATE_TRANSITION`.
15. `publishProduct(p1)` → `409 INVALID_STATE_TRANSITION`; `unpublishProduct(p1)` →
    `409 INVALID_STATE_TRANSITION`.
16. `updateProduct` con la versión vigente y `name: "Y"` → `409 PRODUCT_RETIRED`.
17. `getProduct(p1)` → `200` con `status: "retired"`, todos sus datos y su imagen intactos.
18. Los `ProductUpdated` de los pasos 4, 5, 10 y 13 llevan `version` estrictamente creciente.

**Orden de evaluación** (`publishProduct`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. Está en `draft` → `INVALID_STATE_TRANSITION` (`409`).
3. Tiene al menos una imagen → `PRODUCT_HAS_NO_IMAGES` (`409`).
4. `price.amount` > 0 → `PRODUCT_PRICE_NOT_POSITIVE` (`409`).

**Casos borde**:
- `publishProduct`, `unpublishProduct` y `retireProduct` de un uuid inexistente → `404 PRODUCT_NOT_FOUND`.
- Paso 7 muestra la precedencia 2 sobre 3/4 no aplicable; paso 1 muestra la 3 sobre la 4.

### FL-PRD-021: retirada directa desde borrador y desde publicado

**Given**: `admin` crea `b1`, `c1`. `editor` crea `p1` (draft) y `p2` (`price 10.00 EUR`, con una
imagen, publicado con `publishProduct`). Canal purgado.

**When**: `retireProduct(p1)` y `retireProduct(p2)` como `admin`, sin cuerpo.

**Then**:
1. Las dos responden `200` con `status: "retired"`.
2. El canal recibe dos `ProductUpdated`, uno por producto, cada uno con `status: "retired"`.
3. `getPublishedProduct(p2)` (público) → `404 PRODUCT_NOT_FOUND`.
4. `getProductForServices(p2)` (M2M) → `200` con `status: "retired"`.

### FL-PRD-030: búsqueda de gestión

**Given**: `admin` crea `b1` (`Acme`), `b2` (`Bolt`), `c1` (`Calzado`), `c2` (`Hogar`). `editor` crea:
`Alfa` (`AL-1`, 10.00, `b1`, `c1`), `Beta` (`BE-1`, 20.00, `b1`, `c2`), `Gama` (`GA-1`, 30.00, `b2`,
`c1`); publica `Beta` (con una imagen) y `admin` retira `Gama`.

**When**: `searchProducts` — `GET /api/v1/admin/products` como `editor`, sin parámetros.

**Then**:
1. `200`; `totalElements: 3`, `items` en orden `Alfa`, `Beta`, `Gama`, con la proyección completa de
   gestión y los tres estados (`draft`, `active`, `retired`).
2. `?status=active` → solo `Beta`.
3. `?brandId=b1` → `Alfa`, `Beta`; `?categoryId=c1` → `Alfa`, `Gama`.
4. `?name=ALF` → `Alfa`; `?sku=be` → `Beta`.
5. `?minPrice=10.00&maxPrice=20.00` → `Alfa`, `Beta` (extremos incluidos).
6. `?brandId=b1&categoryId=c1&maxPrice=15` → `Alfa`.
7. `?brandId=<uuid inexistente>` → página vacía canónica (`items: []`, `totalElements: 0`, `totalPages: 0`).
8. `?minPrice=30&maxPrice=10` → `400 INVALID_PRICE_RANGE`.
9. `?minPrice=1.234` → `400 VALIDATION_ERROR`.
10. `?size=2` → `Alfa`, `Beta`, `totalPages: 2`; `?size=2&page=1` → `Gama`; `?size=1000` → `size: 100`.
11. Sin credencial → `401 UNAUTHENTICATED`.

### FL-PRD-040: autorización de la gestión de productos

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1`.

**When / Then**:
1. `retireProduct(p1)` como `editor` → `403 ACCESS_DENIED`; `p1` sigue en `draft`.
2. `retireProduct(p1)` sin credencial → `401 UNAUTHENTICATED`.
3. `createProduct`, `updateProduct`, `publishProduct`, `unpublishProduct`, `addProductImage`,
   `updateProductImage` y `removeProductImage` sin credencial → `401 UNAUTHENTICATED` cada una.
4. `getProduct(p1)` y `searchProducts` sin credencial → `401 UNAUTHENTICATED`.

---

## Imágenes

### FL-IMG-001: subida de imágenes

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1` (`price 10.00 EUR`) y lee su `lockVersion: v1`.
Canal purgado.

**When**: `addProductImage` — `POST /api/v1/admin/products/{p1}/images` como `editor`, multipart con
`file` = un PNG válido de 200 KB, `altText: "Vista lateral"`, sin `main`; `Idempotency-Key: k-img-1`.

**Then**:
1. Status `201`; cabecera `Location` = `/api/v1/admin/products/{p1}`.
2. El cuerpo es la ficha completa de gestión de `p1` con `lockVersion` > `v1` e `images` con un
   elemento: `{ id: i1, file: <URL absoluta>, altText: "Vista lateral", position: 0, main: true,
   productId: p1 }` (la primera imagen es principal aunque no se pida).
3. `GET` a la URL de `file`, sin credencial → `200` con el contenido PNG subido (bucket `public`).
4. El canal recibe un `ProductUpdated` con `images: [{ imageId: i1, file: <key>, altText:
   "Vista lateral", position: 0, main: true }]`.
5. Reintento con `k-img-1` → `201`, mismo cuerpo; `getProduct(p1)` sigue con una sola imagen y el
   canal no recibe otro evento.
6. Segunda subida (JPEG, `main: true`, `k-img-2`) → `201`; `images` en orden: `i1` (`position: 0`,
   `main: false`), `i2` (`position: 1`, `main: true`). La imagen nueva es la de la última position.
7. Tras subir 8 imágenes más (10 en total, claves nuevas), la undécima → `409 IMAGE_LIMIT_REACHED`.

**Orden de evaluación**:
1. `Idempotency-Key` presente → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. Tamaño ≤ 5 MB → `FILE_TOO_LARGE` (`413`).
3. Formato admitido (jpeg, png, webp) y firma del contenido coherente con el tipo declarado →
   `UNSUPPORTED_CONTENT_TYPE` (`415`).
4. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
5. No está retirado → `PRODUCT_RETIRED` (`409`).
6. Tiene menos de 10 imágenes → `IMAGE_LIMIT_REACHED` (`409`).

**Casos borde**:
- Archivo de 6 MB → `413 FILE_TOO_LARGE`.
- Un GIF → `415 UNSUPPORTED_CONTENT_TYPE`; un texto plano declarado como `image/png` → `415
  UNSUPPORTED_CONTENT_TYPE` (la firma no coincide).
- Sin `Idempotency-Key` → `400 IDEMPOTENCY_KEY_REQUIRED`.
- Precedencia: archivo de 6 MB sobre un `productId` inexistente → `413 FILE_TOO_LARGE` (el archivo se
  comprueba antes que el producto).
- PNG válido sobre un uuid inexistente → `404 PRODUCT_NOT_FOUND`.
- `altText` de 161 caracteres → `400 VALIDATION_ERROR`.
- Sobre un producto retirado (`editor` crea `p9`, `admin` lo retira) → `409 PRODUCT_RETIRED`.
- Ninguna subida rechazada cambia `images` ni publica evento.

### FL-IMG-002: dos subidas con la misma clave a la vez

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1`. Canal purgado.

**When**: `addProductImage` dos veces **a la vez** sobre `p1`, con el mismo PNG y la misma
`Idempotency-Key: k-race-i`.

**Then**:
1. O las dos responden `201` con el mismo cuerpo, o una `201` y la otra `409 IDEMPOTENCY_KEY_IN_PROGRESS`.
2. `getProduct(p1)` tiene **exactamente una** imagen, en `position: 0`, `main: true`.
3. El canal recibe **exactamente un** `ProductUpdated`.

### FL-IMG-003: dos subidas distintas a la vez sobre el mismo producto

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1`.

**When**: `addProductImage` dos veces **a la vez** sobre `p1`, con PNG distintos y claves distintas
(`k-a`, `k-b`).

**Then**:
1. Cada respuesta es `201` o `409 CONCURRENT_MODIFICATION`, y al menos una es `201`.
2. `getProduct(p1)` tiene tantas imágenes como respuestas `201`, con posiciones exactamente `0..n-1`
   sin repetir, y exactamente una con `main: true`.

### FL-IMG-010: edición de imágenes

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1` y sube `i1`, `i2`, `i3` (posiciones 0, 1, 2;
`i1` principal). Lee `lockVersion: v`. Canal purgado.

**When**: `updateProductImage` — `PATCH /api/v1/admin/products/{p1}/images/{i3}` como `editor`
```json
{ "version": v, "position": 0, "main": true, "altText": "Portada" }
```

**Then**:
1. Status `200`; ficha completa con `images` en orden `i3` (0, `main: true`, `altText: "Portada"`),
   `i1` (1, `main: false`), `i2` (2, `main: false`); `lockVersion` > `v`.
2. Un `ProductUpdated` con esas `images` en ese orden.
3. `PATCH` de `i3` con la versión vigente y `{ "main": false }` → `409 MAIN_IMAGE_REQUIRED`.
4. `PATCH` de `i2` con la versión vigente y `{ "position": 9 }` → `200`, `i2` queda en `position: 2`
   (al final); como ya estaba al final, es un cambio nulo: misma `lockVersion` y ningún evento.
5. `PATCH` con `version: v` (vieja) → `409 CONCURRENT_MODIFICATION`.
6. `PATCH` de `i1` con `{ "version": <vigente>, "altText": null }` → `200`, `altText: null`.

**Orden de evaluación**:
1. Cotas del input (`version` presente, `position` 0..9, `altText` ≤ 160) → `VALIDATION_ERROR` (`400`).
2. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
3. No está retirado → `PRODUCT_RETIRED` (`409`).
4. La imagen es del producto → `IMAGE_NOT_FOUND` (`404`).
5. `version` vigente → `CONCURRENT_MODIFICATION` (`409`).
6. No se deja sin principal → `MAIN_IMAGE_REQUIRED` (`409`).

**Casos borde**:
- `imageId` inexistente → `404 IMAGE_NOT_FOUND`; `productId` inexistente → `404 PRODUCT_NOT_FOUND`.
- `position: 10` → `400 VALIDATION_ERROR`.
- Tras retirar `p1` (`admin`), cualquier `PATCH` de imagen → `409 PRODUCT_RETIRED`.

### FL-IMG-020: quitar imágenes

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1` (`price 10.00 EUR`, `b1`, `c1`), sube `i1`,
`i2`, `i3` (`i1` principal); crea `p2` (`price 10.00 EUR`, `b1`, `c1`), le sube una sola imagen `j1`
y lo publica. Canal purgado.

**When**: `removeProductImage` — `DELETE /api/v1/admin/products/{p1}/images/{i1}` como `editor`.

**Then**:
1. Status `200`; ficha de `p1` con `images`: `i2` (`position: 0`, `main: true`), `i3` (`position: 1`,
   `main: false`) — se renumera y la principal pasa a la de `position` 0.
2. Un `ProductUpdated` con esas dos imágenes.
3. Repetir el mismo `DELETE` → `404 IMAGE_NOT_FOUND`, sin evento.
4. `DELETE` de `j1` en `p2` (publicado, única imagen) → `409 LAST_IMAGE_OF_PUBLISHED_PRODUCT`; `p2`
   conserva su imagen.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. No está retirado → `PRODUCT_RETIRED` (`409`).
3. La imagen es del producto → `IMAGE_NOT_FOUND` (`404`).
4. No es la última imagen de un producto publicado → `LAST_IMAGE_OF_PUBLISHED_PRODUCT` (`409`).

**Casos borde**:
- `productId` inexistente → `404 PRODUCT_NOT_FOUND`.
- Tras retirar `p1`, `DELETE` de `i2` → `409 PRODUCT_RETIRED`.
- Quitar la última imagen de un producto en `draft` → `200` con `images: []`.

---

## Tienda (público)

### FL-SHP-001: búsqueda pública con filtros

**Given**: `admin` crea `b1` (`Acme`), `b2` (`Bolt`), `c1` (`Calzado`), `c2` (`Hogar`). `editor` crea
y publica (con una imagen cada uno): `Alta Bota` (`AB-1`, 30.00, `b1`, `c1`), `Bota Trail` (`BT-1`,
50.00, `b2`, `c1`), `Cojín` (`CO-1`, 15.00, `b1`, `c2`); crea sin publicar `Dron` (`DR-1`, 20.00,
`b1`, `c1`); crea `Espejo` (`ES-1`, 40.00, `b1`, `c2`), le sube una imagen, lo publica y `admin` lo
retira.

**When**: `searchPublishedProducts` — `GET /api/v1/products`, **sin credencial**.

**Then**:
1. `200`; `totalElements: 3`; `items` en orden `Alta Bota`, `Bota Trail`, `Cojín` (solo `active`).
2. Cada elemento trae exactamente: `id, sku, name, description, price, status, firstPublishedAt,
   brand { id, name, description, active }, category { id, name, description, active }, images[]`.
   **No** trae `lockVersion`, `createdAt`, `updatedAt`, `createdBy` ni `updatedBy`, ni en el producto
   ni en `brand`/`category`.
3. `?categoryId=c1` → `Alta Bota`, `Bota Trail`; `?brandId=b1` → `Alta Bota`, `Cojín`.
4. `?name=bota` → `Alta Bota`, `Bota Trail`; `?name=COJIN` → `Cojín` (ignora mayúsculas y acentos).
5. `?minPrice=15&maxPrice=30` → `Alta Bota`, `Cojín` (extremos incluidos).
6. `?categoryId=c1&maxPrice=40&name=bot` → `Alta Bota`.
7. `admin` desactiva `b2`: `?brandId=b2` sigue devolviendo `Bota Trail` (el filtro se aplica igual).
8. `?minPrice=50&maxPrice=10` → `400 INVALID_PRICE_RANGE`.
9. `?size=2` → 2 elementos, `totalPages: 2`; `?size=2&page=1` → `Cojín`; `?size=2&page=9` → `items: []`,
   `page: 9`, `size: 2`; `?size=500` → `size: 100`.
10. `?name=zzz` → página vacía canónica (`items: []`, `totalElements: 0`, `totalPages: 0`).
11. **Coste**: la página completa (3 productos con marca y categoría anidadas) no cuesta más trabajo
    de almacén que `?size=2`: el trabajo de la operación no crece con el tamaño de la página.

**Notas de determinación**: ningún par de nombres del Given difiere solo en mayúsculas o acentos, así
que el orden no depende de la colación. El filtro `?name=COJIN` sí ejercita la comparación sin acentos.

### FL-SHP-010: ficha pública

**Given**: `admin` crea `b1`, `c1`. `editor` crea `p1`, `p2` y `p3`, los tres con `price 10.00 EUR`,
`b1` y `c1`; sube una imagen a `p1` y a `p3` y los publica; `admin` retira `p3`. `p2` queda en draft.

**When**: `getPublishedProduct` — `GET /api/v1/products/{p1}`, sin credencial.

**Then**:
1. `200` con la proyección pública (misma forma que un elemento de FL-SHP-001), `status: "active"`.
2. `GET /api/v1/products/{p2}` → `404 PRODUCT_NOT_FOUND` (draft no se revela).
3. `GET /api/v1/products/{p3}` → `404 PRODUCT_NOT_FOUND` (retired no se revela).
4. `GET` de un uuid inexistente → `404 PRODUCT_NOT_FOUND`.

### FL-SHP-020: marcas y categorías para los filtros

**Given**: `admin` crea marcas `Acme`, `Bolt`, `Ciclo` y desactiva `Bolt`; crea categorías `Calzado`,
`Hogar` y desactiva `Hogar`.

**When**: `listPublicBrands` — `GET /api/v1/brands`, sin credencial.

**Then**:
1. `200` con una lista (no paginada) en orden `Acme`, `Ciclo`; cada elemento exactamente
   `{ id, name, description, active: true }`, sin `lockVersion` ni auditoría.
2. `listPublicCategories` — `GET /api/v1/categories` → `200` con `[ Calzado ]`, misma forma.
3. Sin ninguna marca activa (tras desactivar `Acme` y `Ciclo`) → `200` con `[]`.

---

## Servidor a servidor (M2M)

### FL-M2M-001: resolver un producto por id

**Given**: `admin` crea `b1` (`Acme`), `c1` (`Calzado`); `editor` crea `p1` (`sku: "M2M-1"`,
`price 12.50 EUR`, una imagen, publicado) y `p2` (draft).

**When**: `getProductForServices` — `GET /api/v1/internal/products/{p1}` con credencial de máquina del
cliente `orders`.

**Then**:
1. `200` con exactamente: `id: p1, sku: "M2M-1", name, description, price: { amount: 12.50, currency:
   "EUR" }, status: "active", firstPublishedAt, lockVersion, brand { id, name, description, active },
   category { id, name, description, active }, images[]` (con URL absoluta en `file`).
   **No** trae `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, ni en `brand`/`category` su
   `lockVersion` ni su auditoría.
2. `p2` con credencial de máquina del cliente `cart` → `200` con `status: "draft"` (cualquier estado).
3. Uuid inexistente → `404 PRODUCT_NOT_FOUND`.
4. Sin credencial → `401 UNAUTHENTICATED`.
5. Con un token de usuario `admin` (no lleva el scope `product:read-internal`) → `403 ACCESS_DENIED`.
6. Con un token de máquina emitido para otra audiencia → `403 ACCESS_DENIED`.

### FL-M2M-010: resolver un lote de productos

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1` (`sku: "ZZ-1"`), `p2` (`sku: "AA-1"`) y `p3`
(`sku: "MM-1"`), y `admin` retira `p3`.

**When**: `getProductsBatch` — `POST /api/v1/internal/products/batch` con credencial de máquina del
cliente `cart`
```json
{ "ids": ["p1", "<uuid inexistente>", "p3", "p2", "p1"] }
```

**Then**:
1. `200` con una lista de **tres** productos en orden de sku: `AA-1` (`p2`), `MM-1` (`p3`, `status:
   "retired"`), `ZZ-1` (`p1`); cada uno con la proyección M2M de FL-M2M-001. El id inexistente se
   omite y `p1` repetido aparece una vez.
2. `{ "ids": [] }` → `400 VALIDATION_ERROR`; 101 ids → `400 VALIDATION_ERROR`; sin cuerpo → `400
   VALIDATION_ERROR`.
3. `{ "ids": ["<uuid inexistente>"] }` → `200` con `[]`.
4. Con credencial de máquina del cliente `orders` → el mismo resultado que con `cart`.
5. Sin credencial → `401 UNAUTHENTICATED`; con token de usuario `admin` → `403 ACCESS_DENIED`.
6. **Coste**: un lote de 100 ids no cuesta más trabajo de almacén que uno de 2: el trabajo no crece
   con el tamaño de la lista.

---

## Eventos (outbox)

### FL-EVT-001: el evento sobrevive a un canal indisponible

**Given**: `admin` crea `b1`, `c1`. El canal `productEvents` sin mensajes y el canal de eventos
**indisponible**.

**When**: `createProduct` como `editor` (`sku: "OBX-1"`, `Idempotency-Key: k-obx-1`).

**Then**:
1. `201` con el cuerpo completo del alta: la indisponibilidad del canal no llega al cliente.
2. `getProduct` devuelve el producto en `draft`.
3. El canal `productEvents` no ha recibido ningún mensaje todavía.

**When**: el canal vuelve a estar disponible.

**Then**:
4. En ≤ 10 s el canal recibe **exactamente un** `ProductCreated` con `sku: "OBX-1"`, y su
   `correlationId` es el de la petición del paso 1.
5. Ningún evento abandonado.

### FL-EVT-002: el evento que el relay abandona no se pierde en silencio

**Given**: el canal indisponible y un `createProduct` ya ejecutado (`sku: "OBX-2"`), con su
`ProductCreated` pendiente de salir.

**When**: se agota el presupuesto de reintentos de ese evento.

**Then**:
1. El servidor informa de **un** evento abandonado.
2. Restablecido el canal, ese `ProductCreated` **no** se publica.
3. El canal no recibe ninguna otra cosa.

---

## Transversal

### FL-SEC-001: CORS desde el navegador

**Given**: `admin` crea `b1`, `c1`; `editor` crea `p1` (`price 10.00 EUR`, `b1`, `c1`), le sube una
imagen y lo publica.

**When**: preflight `OPTIONS /api/v1/admin/products` **sin credencial**, desde un origen web permitido,
pidiendo método `POST` y cabeceras `Authorization, Content-Type, Idempotency-Key`.

**Then**:
1. Respuesta de preflight aceptada: permite `POST` y las tres cabeceras, con `max-age` 3600.
2. No exige credencial (no es `401`).

**When**: `GET /api/v1/products` desde ese origen (petición normal cross-origin).

**Then**:
3. `200`, y la respuesta expone al navegador `X-Correlation-Id`.
4. `createProduct` desde ese origen con credencial de `editor` → `201`, y el navegador puede leer
   `Location`.
