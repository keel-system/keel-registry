# catalog — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/catalog v0.3.0. Contrato de validación para la fase de generación.

## Convenciones de determinación

Valen para **todo** el servicio y ningún escenario las repite.

- **Ausencia vs nulo**: un campo sin valor **no aparece** en la respuesta. Nunca viaja como `null`.
- **Fecha y hora**: instantes en UTC, ISO-8601 con milisegundos (`2026-03-14T09:21:07.482Z`).
  `createdAt`, `updatedAt`, `retiredAt` y `occurredAt` se verifican **por forma o por rango**, jamás
  por valor literal.
- **Identificadores generados**: los `id` son uuid; se verifican por forma y por **reutilización
  simbólica** dentro del flujo (el id devuelto en un escenario es el que usa el siguiente).
- **Números y dinero**: `price.amount` es decimal de **escala 2** (`12.50`, nunca `12.5`);
  `price.currency` es ISO-4217 en mayúsculas. El servicio no calcula importes, así que no hay redondeo.
- **Mayúsculas y acentos**: la unicidad de `name` y el filtro `q` comparan **ignorando mayúsculas,
  minúsculas y acentos**. `ACME`, `acme` y `Ácme` son el mismo nombre y colisionan entre sí.
- **Slug**: se deriva del `name` en kebab-case ASCII (acentos transliterados, signos eliminados,
  espacios a guiones): `"Ácme Deportes!"` → `acme-deportes`.
- **Proyección de un archivo**: un campo `type: file` de un bucket `public` se proyecta en las
  respuestas y en los payloads de evento como la **URL absoluta de lectura** del objeto, no como una
  clave opaca ni como una ruta relativa. El host base es **configuración de despliegue, no contrato**:
  los escenarios verifican esa URL **por forma y por comportamiento** —una URL absoluta que, pedida
  sin credencial ni firma, devuelve el contenido subido con su `Content-Type`—, jamás por valor
  literal.
- **Forma del cuerpo de error**: la fija el generador — en keel-spring,
  `{timestamp, status, error, code, message, details}` más `correlationId`. Los escenarios solo
  especifican el **`code`** y el **status**; el `message` no es contrato.
- **Paginación**: se pide con `page` (base 0) y `size`; el sobre de respuesta es siempre
  `{ items, page, size, totalElements, totalPages }`. `defaultSize` 20, `maxSize` 100: un `size`
  mayor se **recorta** a 100, no da error.
- **Cabecera `Location`**: toda creación `201` la emite con la URI de la petición más el id devuelto.
- **Idempotencia**: la clave viaja en la cabecera `Idempotency-Key`, con retención de 24 h.
- **Concurrencia**: `optimisticLocking: all` — dos escrituras sobre la misma raíz producen
  `CONCURRENT_MODIFICATION` (`409`) en la segunda, nunca un último-gana silencioso.
- **Credencial de máquina**: donde se diga "credencial de máquina del cliente `order-service`" se
  entiende un token de cliente con `scope: product:read` y audiencia `catalog`. El proveedor concreto
  no es contrato.
- **Envoltura de eventos**: todo evento publicado viaja como `{ metadata, data }`, con
  `metadata.eventType`, `metadata.eventId`, `metadata.occurredAt` y `metadata.source: "catalog"`.
  Los escenarios verifican `eventType`, el `channel` y el contenido de `data`.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|---|---|---|
| createBrand | FL-BRD-001, FL-SEC-001 | usuarios |
| getBrand | FL-BRD-001 | usuarios |
| updateBrand | FL-BRD-010 | usuarios |
| deactivateBrand | FL-BRD-020 | usuarios |
| activateBrand | FL-BRD-020 | usuarios |
| listBrands | FL-BRD-030 | usuarios |
| createCategory | FL-CTG-001 | usuarios |
| getCategory | FL-CTG-001 | usuarios |
| updateCategory | FL-CTG-010 | usuarios |
| deactivateCategory | FL-CTG-020, FL-SEC-001 | usuarios |
| activateCategory | FL-CTG-020 | usuarios |
| listCategories | FL-CTG-030 | usuarios |
| createProduct | FL-PRD-001, FL-SEC-001 | usuarios |
| addProductImage | FL-PRD-010, FL-SEC-001 | usuarios |
| removeProductImage | FL-PRD-011 | usuarios |
| reorderProductImages | FL-PRD-012 | usuarios |
| publishProduct | FL-PRD-020, FL-SEC-001 | usuarios |
| updateProduct | FL-PRD-030 | usuarios |
| retireProduct | FL-PRD-040 | usuarios |
| getProduct | FL-PRD-001, FL-PRD-040 | usuarios |
| listProducts | FL-PRD-050 | usuarios |
| getPublishedProduct | FL-PUB-001, FL-PUB-010 | usuarios (público) |
| listPublishedProducts | FL-PUB-001 | usuarios (público) |
| listPublishedBrands | FL-PUB-020 | usuarios (público) |
| listPublishedCategories | FL-PUB-020 | usuarios (público) |
| getProductForServices | FL-M2M-001, FL-M2M-020 | **servidores (M2M)** |
| getProductsByIds | FL-M2M-010, FL-M2M-020 | **servidores (M2M)** |

---

## Marcas

### FL-BRD-001: alta de marca, consulta y colisión de nombre

**Given**: no existe ninguna marca.

**When**: `createBrand` — `POST /api/v1/brands`
```json
{ "name": "Ácme Deportes", "description": "Material deportivo" }
```

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta de la marca creada.
3. El cuerpo trae `id` (forma de uuid), `name: "Ácme Deportes"`, `slug: "acme-deportes"`,
   `description: "Material deportivo"`, `createdAt` y `updatedAt` (forma de instante UTC).
4. El cuerpo no trae ningún campo adicional.
5. `getBrand` sobre el `id` devuelto (`GET /api/v1/brands/{id}`) responde `200` con el mismo cuerpo.

**Escenarios encadenados**:
- `createBrand` con `{ "name": "acme deportes" }` → `409` `BRAND_NAME_ALREADY_EXISTS`: la comparación
  ignora mayúsculas y acentos.
- `createBrand` con `{ "name": "Acme Deportes!" }` → `409` `SLUG_ALREADY_EXISTS`: el `name` difiere de
  los anteriores, pero produce el mismo `slug` `acme-deportes`.
- `getBrand` con un uuid que no existe → `404` `BRAND_NOT_FOUND`.

**Orden de evaluación**:
1. No existe otra marca con ese `name` → `BRAND_NAME_ALREADY_EXISTS` (`409`).
2. No existe otra marca con el `slug` derivado → `SLUG_ALREADY_EXISTS` (`409`).

**Casos borde**:
- `name` ausente → `400`.
- `name` de 121 caracteres → `400` (`maxLength: 120`).
- `name: "ACME DEPORTES"` **y** slug ya usado a la vez → `BRAND_NAME_ALREADY_EXISTS` (`409`): la
  guarda 1 precede a la 2.

---

### FL-BRD-010: edición de marca, recálculo de slug y conflicto de concurrencia

**Given**: `createBrand` deja la marca `b1` (`name: "Ácme Deportes"`, `slug: "acme-deportes"`,
`description: "Material deportivo"`) y `createBrand` deja la marca `b2` (`name: "Delta"`).

**When**: `updateBrand` — `PUT /api/v1/brands/{b1}`
```json
{ "name": "Acme Sport" }
```

**Then**:
1. Status `200`.
2. El cuerpo trae `id: b1`, `name: "Acme Sport"` y `slug: "acme-sport"` (recalculado).
3. El cuerpo **no** trae `description`: es un reemplazo total y el campo se omitió en la petición.
4. `updatedAt` es posterior o igual a `createdAt`.
5. `getBrand` sobre `b1` devuelve el mismo cuerpo, sin `description`.

**Escenarios encadenados**:
- `updateBrand` sobre `b2` con `{ "name": "acme sport" }` → `409` `BRAND_NAME_ALREADY_EXISTS`.
- `updateBrand` sobre `b2` con `{ "name": "Acme Sport." }` → `409` `SLUG_ALREADY_EXISTS`.
- `updateBrand` sobre un uuid inexistente → `404` `BRAND_NOT_FOUND`.
- **Concurrencia**: dos `updateBrand` sobre `b1` partiendo ambos de la misma lectura, ejecutados en
  paralelo → uno responde `200` y el otro `409` `CONCURRENT_MODIFICATION`. Nunca dos `200`.
  Una relectura de `b1` devuelve exactamente el cuerpo del que obtuvo `200`.

**Orden de evaluación**:
1. La marca existe → `BRAND_NOT_FOUND` (`404`).
2. No existe otra marca con ese `name` → `BRAND_NAME_ALREADY_EXISTS` (`409`).
3. No existe otra marca con el `slug` derivado → `SLUG_ALREADY_EXISTS` (`409`).
4. La marca no cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**:
- Marca inexistente **y** `name` colisionando en la misma llamada → `BRAND_NOT_FOUND` (`404`): la
  guarda 1 precede a la 2.

---

### FL-BRD-020: desactivación y reactivación de una marca

**Given**: `createBrand` deja `b1` (`status: "active"`); `createCategory` deja `c1`; `createProduct`
deja el producto `p1` en `draft`, con `brandId: b1` y `categoryId: c1`.

**When**: `deactivateBrand` — `POST /api/v1/brands/{b1}/deactivate`

**Then**:
1. Status `200`.
2. El cuerpo trae `id: b1`, `status: "inactive"` y el resto de campos sin cambios.
3. `getBrand` sobre `b1` responde `200` con `status: "inactive"`: la marca **no** se ha eliminado.
4. `getProduct` sobre `p1` sigue respondiendo `200`, con su `brand` embebido: el producto conserva su
   referencia y su estado.

**Escenarios encadenados**:
1. `createProduct` con `brandId: b1` da `422` `BRAND_INACTIVE`: una marca desactivada no admite
   productos nuevos.
2. `updateProduct` sobre `p1` **conservando** `brandId: b1` da `200`: mantener la marca que ya tenía
   es válido aunque esté desactivada.
3. `createBrand` deja `b2` (`active`); `updateProduct` sobre `p1` con `brandId: b2` da `200`, y
   `updateProduct` de vuelta con `brandId: b1` da `422` `BRAND_INACTIVE`: reasignar **hacia** una
   marca desactivada no se permite.
4. `deactivateBrand` sobre `b1` otra vez da `409` `ILLEGAL_STATUS_TRANSITION`.
5. `activateBrand` sobre `b1` da `200` con `status: "active"`, y `createProduct` con `brandId: b1`
   vuelve a dar `201`.
6. `activateBrand` sobre `b1` otra vez da `409` `ILLEGAL_STATUS_TRANSITION`.
7. `deactivateBrand` y `activateBrand` sobre un uuid inexistente dan `404` `BRAND_NOT_FOUND`.
8. `listBrands` sin filtro devuelve `b1` y `b2`; `?status=inactive` devuelve solo las desactivadas.

**Orden de evaluación** (`deactivateBrand` y `activateBrand`):
1. La marca existe → `BRAND_NOT_FOUND` (`404`).
2. Está en el estado de origen de la transición → `ILLEGAL_STATUS_TRANSITION` (`409`).
3. No cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**:
- **Concurrencia**: dos `deactivateBrand` simultáneos sobre `b1` (`active`) → uno `200` y el otro
  `409` (`ILLEGAL_STATUS_TRANSITION` o `CONCURRENT_MODIFICATION`); nunca dos `200`.
- Ninguna ruta del servicio permite **eliminar** una marca: `DELETE /api/v1/brands/{b1}` no existe.

### FL-BRD-030: listado de marcas, orden y paginación

**Given**: `createBrand` deja, en este orden de creación, `"Zeta"`, `"alfa"`, `"Ómega"` y `"Beta"`.

**When**: `listBrands` — `GET /api/v1/brands`

**Then**:
1. Status `200`.
2. El sobre trae `page: 0`, `size: 20`, `totalElements: 4`, `totalPages: 1`.
3. `items` trae los cuatro en orden `alfa`, `Beta`, `Ómega`, `Zeta` — ascendente por `name`
   ignorando mayúsculas y acentos, distinto del orden de creación.
4. Cada elemento trae `id`, `name`, `slug`, `createdAt` y `updatedAt`.

**Escenarios encadenados**:
- `GET /api/v1/brands?size=2` → `items` con `alfa` y `Beta`, `totalPages: 2`.
- `GET /api/v1/brands?page=1&size=2` → `items` con `Ómega` y `Zeta`.
- `GET /api/v1/brands?page=9&size=2` → `items` vacío, `totalElements: 4`, status `200`.
- `GET /api/v1/brands?size=500` → `size: 100` en el sobre (recortado a `maxSize`), no `400`.
- `GET /api/v1/brands?q=ome` → solo `Ómega`: el filtro ignora acentos.

**Notas de determinación**: el desempate por `id` ascendente no es observable con estos datos porque
los cuatro `name` difieren; un quinto alta con `name: "alfa"` es imposible (colisiona), así que el
orden queda totalmente determinado por `name`.

---

## Categorías

### FL-CTG-001: alta de categoría, consulta y colisiones

**Given**: no existe ninguna categoría.

**When**: `createCategory` — `POST /api/v1/categories`
```json
{ "name": "Calzado", "description": "Zapatillas y botas" }
```

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta de la categoría creada.
3. El cuerpo trae `id` (uuid), `name: "Calzado"`, `slug: "calzado"`,
   `description: "Zapatillas y botas"`, `createdAt` y `updatedAt`.
4. `getCategory` sobre el `id` devuelto responde `200` con el mismo cuerpo.

**Escenarios encadenados**:
- `createCategory` con `{ "name": "CALZADO" }` → `409` `CATEGORY_NAME_ALREADY_EXISTS`.
- `createCategory` con `{ "name": "¡Calzado!" }` → `409` `SLUG_ALREADY_EXISTS`.
- `getCategory` con un uuid inexistente → `404` `CATEGORY_NOT_FOUND`.

**Orden de evaluación**:
1. No existe otra categoría con ese `name` → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).
2. No existe otra categoría con el `slug` derivado → `SLUG_ALREADY_EXISTS` (`409`).

**Casos borde**: `name` ausente → `400`; `name` de 121 caracteres → `400`.

---

### FL-CTG-010: edición de categoría y conflicto de concurrencia

**Given**: existen las categorías `c1` (`name: "Calzado"`) y `c2` (`name: "Ropa"`), creadas en este flujo.

**When**: `updateCategory` — `PUT /api/v1/categories/{c1}`
```json
{ "name": "Calzado deportivo", "description": "Zapatillas de running" }
```

**Then**:
1. Status `200`.
2. El cuerpo trae `name: "Calzado deportivo"`, `slug: "calzado-deportivo"` y
   `description: "Zapatillas de running"`.
3. `getCategory` sobre `c1` devuelve el mismo cuerpo.

**Escenarios encadenados**:
- `updateCategory` sobre `c2` con `{ "name": "calzado deportivo" }` → `409`
  `CATEGORY_NAME_ALREADY_EXISTS`.
- `updateCategory` sobre `c2` con `{ "name": "Calzado-deportivo" }` → `409` `SLUG_ALREADY_EXISTS`.
- `updateCategory` sobre un uuid inexistente → `404` `CATEGORY_NOT_FOUND`.
- **Concurrencia**: dos `updateCategory` sobre `c1` desde la misma lectura → uno `200`, el otro `409`
  `CONCURRENT_MODIFICATION`.
- `updateCategory` sobre `c1` omitiendo `description` → `200` y el cuerpo **sin** `description`.

**Orden de evaluación**:
1. La categoría existe → `CATEGORY_NOT_FOUND` (`404`).
2. `name` libre → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).
3. `slug` libre → `SLUG_ALREADY_EXISTS` (`409`).
4. No cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

---

### FL-CTG-020: desactivación y reactivación de una categoría

**Given**: `createBrand` deja `b1`, `createCategory` deja `c1` (`status: "active"`) y `createProduct`
deja `p1` en `draft` con `categoryId: c1`.

**When**: `deactivateCategory` — `POST /api/v1/categories/{c1}/deactivate`

**Then**:
1. Status `200`.
2. El cuerpo trae `id: c1` y `status: "inactive"`.
3. `getCategory` sobre `c1` responde `200`: la categoría no se ha eliminado.
4. `getProduct` sobre `p1` sigue respondiendo `200` con su `category` embebida.

**Escenarios encadenados**:
1. `createProduct` con `categoryId: c1` da `422` `CATEGORY_INACTIVE`.
2. `updateProduct` sobre `p1` conservando `categoryId: c1` da `200`.
3. `deactivateCategory` sobre `c1` otra vez da `409` `ILLEGAL_STATUS_TRANSITION`.
4. `activateCategory` sobre `c1` da `200` con `status: "active"`; repetirlo da `409`
   `ILLEGAL_STATUS_TRANSITION`.
5. Ambas operaciones sobre un uuid inexistente dan `404` `CATEGORY_NOT_FOUND`.
6. `listCategories?status=active` no incluye las desactivadas.

**Orden de evaluación**:
1. La categoría existe → `CATEGORY_NOT_FOUND` (`404`).
2. Está en el estado de origen de la transición → `ILLEGAL_STATUS_TRANSITION` (`409`).
3. No cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**: `DELETE /api/v1/categories/{c1}` no existe; una categoría nunca se elimina.

### FL-CTG-030: listado de categorías, orden y paginación

**Given**: `createCategory` deja `"Ropa"`, `"Accesorios"` y `"Calzado"`, en ese orden de creación.

**When**: `listCategories` — `GET /api/v1/categories`

**Then**:
1. Status `200`, sobre con `page: 0`, `size: 20`, `totalElements: 3`, `totalPages: 1`.
2. `items` en orden `Accesorios`, `Calzado`, `Ropa` — ascendente por `name`, distinto del de creación.

**Escenarios encadenados**:
- `?size=1&page=1` → `items` solo con `Calzado`, `totalPages: 3`.
- `?page=7` → `items` vacío, `200`.
- `?size=500` → `size: 100` en el sobre.
- `?q=CALZ` → solo `Calzado`.

---

## Productos: gestión

### FL-PRD-001: alta de producto, evento e idempotencia

**Given**: `createBrand` deja `b1` (`name: "Acme"`) y `createCategory` deja `c1` (`name: "Calzado"`).
No existe ningún producto.

**When**: `createProduct` — `POST /api/v1/products`, cabecera `Idempotency-Key: key-001`
```json
{
  "sku": "acme-run-01",
  "name": "Zapatilla Run",
  "description": "Zapatilla de running neutra",
  "price": { "amount": 89.90, "currency": "EUR" },
  "brandId": "b1",
  "categoryId": "c1",
  "tags": ["running", "neutra"]
}
```

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta del producto creado.
3. El cuerpo trae `id` (uuid), `sku: "ACME-RUN-01"` (normalizado a mayúsculas),
   `name: "Zapatilla Run"`, `description: "Zapatilla de running neutra"`,
   `price: { "amount": 89.90, "currency": "EUR" }`, `status: "draft"`,
   `tags: ["running", "neutra"]`, `createdAt` y `updatedAt`.
4. El cuerpo trae `brand` como objeto anidado con `id: b1`, `name: "Acme"` y `slug: "acme"`, **no**
   `brandId`; e igual `category` con `id: c1` y `name: "Calzado"`, **no** `categoryId`.
5. El objeto `brand` **no** trae relaciones anidadas propias (profundidad 1).
6. El cuerpo trae `images` como lista **vacía**: el producto nace con la galería sin poblar.
7. Se publica `ProductCreated` en el canal `productEvents`, con `data.productId` (el id devuelto),
   `data.sku: "ACME-RUN-01"`, `data.status: "draft"`, `data.brandId: b1`, `data.categoryId: c1`,
   `data.price: { "amount": 89.90, "currency": "EUR" }` y `data.createdAt`.
8. `data` **no** trae `brandName` ni `categoryName`.
9. `getProduct` sobre el id devuelto responde `200` con el mismo cuerpo que la aserción 3–6.

**Escenarios encadenados**:
- Repetir la **misma** llamada con `Idempotency-Key: key-001` y **el mismo cuerpo** → mismo status
  `201` y **exactamente el mismo cuerpo** (mismo `id`). `listProducts` sigue devolviendo
  `totalElements: 1`, y **no** se publica un segundo `ProductCreated`.
- Repetir con `Idempotency-Key: key-001` y un cuerpo distinto (`name: "Otro"`) → `409`
  `IDEMPOTENCY_KEY_REUSED`.
- `createProduct` con `Idempotency-Key: key-002` y `sku: "ACME-RUN-01"` → `409` `SKU_ALREADY_EXISTS`.
- `createProduct` con `brandId` inexistente → `422` `BRAND_NOT_FOUND`.
- `createProduct` con `categoryId` inexistente → `422` `CATEGORY_NOT_FOUND`.

**Orden de evaluación**:
1. La clave de idempotencia no se usó con otro contenido → `IDEMPOTENCY_KEY_REUSED` (`409`).
2. No existe producto con ese `sku`, incluidos los retirados → `SKU_ALREADY_EXISTS` (`409`).
3. La marca existe → `BRAND_NOT_FOUND` (`422`).
4. La categoría existe → `CATEGORY_NOT_FOUND` (`422`).
5. La marca está `active` → `BRAND_INACTIVE` (`422`).
6. La categoría está `active` → `CATEGORY_INACTIVE` (`422`).

**Casos borde**:
- `sku: "AB"` → `400` (patrón de `SKU`: 4 a 20 caracteres).
- `sku` con espacios → `400`.
- `price.currency: "eur"` → `400` (patrón `^[A-Z]{3}$`).
- `price.amount: -1` → `400` (`min: 0`).
- 11 elementos en `tags` → `400` (`maxItems: 10`).
- `brandId` **y** `categoryId` inexistentes a la vez → `BRAND_NOT_FOUND` (`422`): la guarda 3 precede
  a la 4.

---

### FL-PRD-010: alta de imágenes en la galería

**Given**: `createBrand`, `createCategory` y `createProduct` dejan `p1` en `draft` con `images` vacío.

**When**: `addProductImage` — `POST /api/v1/products/{p1}/images`, multipart con un `image/jpeg` de
1 MB y `altText: "Vista lateral"`.

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta del producto.
3. El cuerpo trae `images` con **un** elemento: `id` (uuid), `position: 1`,
   `altText: "Vista lateral"`, `file` con la referencia en el bucket `productImages`, y `createdAt`.
4. El bucket `productImages` es `public`: la URL de ese `file` se lee **directamente**, sin credencial
   ni firma, y devuelve el contenido subido con `Content-Type: image/jpeg`.
5. `getProduct` sobre `p1` devuelve la misma galería.
6. Se publica `ProductUpdated` en `productEvents` con `data.productId: p1` y `data.imageUrls` como
   lista de **una** URL, la del archivo subido.

**Escenarios encadenados**:
1. Un segundo `addProductImage` sobre `p1` (`image/png`, sin `altText`) da `201`, y `images` trae dos
   elementos con `position: 1` y `position: 2`, en ese orden. El segundo **no** trae `altText`.
2. El `ProductUpdated` correspondiente trae `data.imageUrls` con **dos** URLs, en orden de posición.
3. Añadir hasta completar **ocho** imágenes: todas `201`, con `position` de 1 a 8.
4. Un noveno `addProductImage` da `409` `MAX_IMAGES_REACHED`, y `images` sigue con ocho elementos.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está retirado → `PRODUCT_RETIRED` (`409`).
3. Quedan menos de ocho imágenes → `MAX_IMAGES_REACHED` (`409`).
4. El tamaño cabe en el bucket → `FILE_TOO_LARGE` (`413`).
5. El content-type está permitido → `UNSUPPORTED_CONTENT_TYPE` (`415`).

**Casos borde**:
- **Concurrencia**: dos `addProductImage` simultáneos sobre `p1` desde la misma lectura → uno `201` y
  el otro `409` `CONCURRENT_MODIFICATION`; la galería queda con **una** imagen nueva, no dos.
- Archivo `image/jpeg` de 6 MB → `413` `FILE_TOO_LARGE` (`maxSizeMb: 5`).
- Archivo `application/pdf` → `415` `UNSUPPORTED_CONTENT_TYPE`.
- `productId` inexistente → `404` `PRODUCT_NOT_FOUND`.
- Producto retirado (ver FL-PRD-040) → `409` `PRODUCT_RETIRED`.
- Producto con ocho imágenes **y** archivo de 6 MB → `MAX_IMAGES_REACHED` (`409`): la guarda 3
  precede a la 4.
- `altText` de 201 caracteres → `400` (`maxLength: 200`).

---

### FL-PRD-011: borrado de imágenes y recompactación de posiciones

**Given**: en este flujo se crean `b1`, `c1` y `p1` en `draft`, y se añaden tres imágenes: `i1`
(`position: 1`), `i2` (`position: 2`) e `i3` (`position: 3`).

**When**: `removeProductImage` — `DELETE /api/v1/products/{p1}/images/{i1}`

**Then**:
1. Status `200`.
2. El cuerpo trae `images` con **dos** elementos: `i2` con `position: 1` e `i3` con `position: 2`.
   Las posiciones se recompactan desde 1 y `i2` pasa a ser la principal.
3. La URL del archivo de `i1` deja de resolver: el objeto se eliminó del bucket.
4. Se publica `ProductUpdated` con `data.imageUrls` de **dos** URLs, encabezada por la de `i2`.
5. `getProduct` sobre `p1` devuelve la misma galería.

**Escenarios encadenados**:
- `removeProductImage` con un `imageId` que no existe da `404` `IMAGE_NOT_FOUND`.
- `removeProductImage` con el `imageId` de una imagen de **otro** producto da `404`
  `IMAGE_NOT_FOUND`, no `403`.
- `removeProductImage` sobre un `productId` inexistente da `404` `PRODUCT_NOT_FOUND`.
- Borrar `i3` y luego `i2` sobre el producto en `draft` da `200` en ambos, y `images` queda vacío: un
  producto en `draft` sí puede quedarse sin galería.
- **Última imagen de un publicado**: con `p2` publicado (`active`) y una sola imagen,
  `removeProductImage` da `409` `LAST_IMAGE_OF_ACTIVE_PRODUCT`; `getProduct` sobre `p2` sigue con
  `status: "active"` y su imagen intacta, y no se publica ningún evento.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está retirado → `PRODUCT_RETIRED` (`409`).
3. La imagen existe y es de ese producto → `IMAGE_NOT_FOUND` (`404`).
4. No es la única imagen de un producto `active` → `LAST_IMAGE_OF_ACTIVE_PRODUCT` (`409`).
5. El producto no cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**: dos `removeProductImage` simultáneos sobre imágenes distintas de `p1` → uno `200` y
el otro `409` `CONCURRENT_MODIFICATION`; las posiciones resultantes son consecutivas desde 1, sin
huecos, porque la recompactación ocurre dentro de la transacción del agregado.

---

### FL-PRD-012: reordenación de la galería

**Given**: en este flujo se crean `b1`, `c1` y `p1` en `draft`, con tres imágenes `i1`
(`position: 1`), `i2` (`position: 2`) e `i3` (`position: 3`).

**When**: `reorderProductImages` — `PUT /api/v1/products/{p1}/images/order`
```json
{ "imageIds": ["i3", "i1", "i2"] }
```

**Then**:
1. Status `200`.
2. El cuerpo trae `images` en el orden `i3` (`position: 1`), `i1` (`position: 2`), `i2`
   (`position: 3`).
3. `i3` es ahora la imagen principal.
4. Se publica `ProductUpdated` con `data.imageUrls` en el orden nuevo, encabezada por la de `i3`.
5. `getProduct` sobre `p1` devuelve la galería en el orden nuevo.

**Escenarios encadenados**:
- `{ "imageIds": ["i3", "i1"] }` (falta `i2`) da `422` `INCOMPLETE_IMAGE_ORDER`, y el orden anterior
  se conserva.
- `{ "imageIds": ["i3", "i3", "i1"] }` (repetida) da `422` `INCOMPLETE_IMAGE_ORDER`.
- `{ "imageIds": ["i3", "i1", "<uuid-ajeno>"] }` da `404` `IMAGE_NOT_FOUND`.
- `productId` inexistente da `404` `PRODUCT_NOT_FOUND`.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está retirado → `PRODUCT_RETIRED` (`409`).
3. Todos los ids son imágenes de ese producto → `IMAGE_NOT_FOUND` (`404`).
4. La lista cubre todas las imágenes, cada una una vez → `INCOMPLETE_IMAGE_ORDER` (`422`).
5. El producto no cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**:
- `imageIds` vacío da `400` (`minItems: 1`).
- `imageIds` con 9 elementos da `400` (`maxItems: 8`).
- Id ajeno **y** lista incompleta a la vez da `IMAGE_NOT_FOUND` (`404`): la guarda 3 precede a la 4.
- **Concurrencia**: un `reorderProductImages` y un `addProductImage` simultáneos sobre `p1` → uno
  responde con éxito y el otro `409` `CONCURRENT_MODIFICATION`.

---

### FL-PRD-020: publicación del producto (draft → active)

**Given**: `createBrand` deja `b1`, `createCategory` deja `c1`, `createProduct` deja `p1` en `draft`
con `price.amount: 89.90` y la galería **vacía**.

**When**: `publishProduct` — `POST /api/v1/products/{p1}/publish`

**Then**:
1. Status `422`, `code: IMAGE_REQUIRED`.
2. `getProduct` sobre `p1` sigue devolviendo `status: "draft"`.
3. No se publica ningún evento.

**Escenarios encadenados**:
1. `addProductImage` sobre `p1` con un `image/jpeg` válido → `201`.
2. `publishProduct` sobre `p1` → `200`, cuerpo con `status: "active"` y el resto de campos intactos.
3. Se publica `ProductUpdated` en `productEvents` con `data.productId: p1`, `data.status: "active"`,
   `data.sku`, `data.price`, `data.brandId`, `data.categoryId`, `data.imageUrls` (una URL) y `data.updatedAt`.
4. `publishProduct` sobre `p1` otra vez → `409` `ILLEGAL_STATUS_TRANSITION` (ya está `active`).
5. `publishProduct` sobre un uuid inexistente → `404` `PRODUCT_NOT_FOUND`.
6. `createProduct` deja `p2` con `price.amount: 0.00`; `addProductImage` le pone una imagen;
   `publishProduct` sobre `p2` → `422` `PRICE_MUST_BE_POSITIVE`.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. Está en `draft` → `ILLEGAL_STATUS_TRANSITION` (`409`).
3. Tiene precio positivo → `PRICE_MUST_BE_POSITIVE` (`422`).
4. Tiene al menos una imagen → `IMAGE_REQUIRED` (`422`).
5. No cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Casos borde**:
- Producto ya `active` **y** con la galería vacía → `ILLEGAL_STATUS_TRANSITION` (`409`): la guarda 2
  precede a la 4.
- **Concurrencia**: dos `publishProduct` simultáneos sobre el mismo `p1` en `draft` → uno `200` y el
  otro `409` (`ILLEGAL_STATUS_TRANSITION` o `CONCURRENT_MODIFICATION`); nunca dos `200` ni dos
  `ProductUpdated` con `status: "active"`.

---

### FL-PRD-030: edición de la ficha y reemplazo total

**Given**: `p1` existe en estado `active` (creado, con una imagen y publicado en este flujo), con
`description: "Zapatilla de running neutra"`, `tags: ["running", "neutra"]`, marca `b1` y
categoría `c1`. Existe además la marca `b2` (`name: "Delta"`).

**When**: `updateProduct` — `PUT /api/v1/products/{p1}`
```json
{
  "name": "Zapatilla Run II",
  "price": { "amount": 99.00, "currency": "EUR" },
  "brandId": "b2",
  "categoryId": "c1"
}
```

**Then**:
1. Status `200`.
2. El cuerpo trae `name: "Zapatilla Run II"`, `price: { "amount": 99.00, "currency": "EUR" }`,
   `status: "active"` (la edición no cambia el estado) y `sku` **sin cambios**.
3. El cuerpo **no** trae `description` ni `tags`: es un reemplazo total y se omitieron.
4. `brand` es el objeto anidado de `b2` (`name: "Delta"`).
5. Se publica `ProductUpdated` en `productEvents` con `data.productId: p1`, `data.name`,
   `data.price.amount: 99.00`, `data.brandId: b2` y `data.status: "active"`.
6. `data` **no** trae `description` ni `tags`.

**Escenarios encadenados**:
- `updateProduct` con `brandId` inexistente → `422` `BRAND_NOT_FOUND`.
- `updateProduct` con `categoryId` inexistente → `422` `CATEGORY_NOT_FOUND`.
- `updateProduct` sobre `p1` (`active`) con `price.amount: 0.00` → `422` `PRICE_MUST_BE_POSITIVE`.
- `updateProduct` sobre un uuid inexistente → `404` `PRODUCT_NOT_FOUND`.
- **Concurrencia**: dos `updateProduct` sobre `p1` desde la misma lectura → uno `200`, el otro `409`
  `CONCURRENT_MODIFICATION`. Una relectura coincide con el cuerpo del ganador, y se publicó **un
  solo** `ProductUpdated`.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. No está retirado → `PRODUCT_RETIRED` (`409`).
3. La marca existe → `BRAND_NOT_FOUND` (`422`).
4. La categoría existe → `CATEGORY_NOT_FOUND` (`422`).
5. La marca de destino está `active`, salvo que sea la que ya tenía → `BRAND_INACTIVE` (`422`).
6. La categoría de destino está `active`, salvo que sea la que ya tenía → `CATEGORY_INACTIVE` (`422`).
7. Un producto `active` conserva precio positivo → `PRICE_MUST_BE_POSITIVE` (`422`).
8. No cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

**Ramas condicionales**: omitir `description` o `tags` los deja **sin valor**; enviarlos los
sustituye por completo (`tags` no se fusiona con el valor anterior).

---

### FL-PRD-040: retirada del producto y sku reservado

**Given**: `p1` existe en estado `active` (creado, con una imagen y publicado en este flujo), con
`sku: "ACME-RUN-01"`.

**When**: `retireProduct` — `POST /api/v1/products/{p1}/retire`

**Then**:
1. Status `200`.
2. El cuerpo trae `status: "retired"` y conserva `sku`, `name`, `price`, `brand`, `category` y su
   galería `images` completa.
3. Se publica `ProductRetired` en `productEvents` con `data.productId: p1`,
   `data.sku: "ACME-RUN-01"` y `data.retiredAt`.
4. `data` **no** trae `price` ni `status`.
5. `getProduct` sobre `p1` responde `200` con `status: "retired"` (la gestión sigue viéndolo).

**Escenarios encadenados**:
- `retireProduct` sobre `p1` otra vez → `409` `ILLEGAL_STATUS_TRANSITION` (`retired` es terminal).
- `updateProduct` sobre `p1` → `409` `PRODUCT_RETIRED`.
- `addProductImage` sobre `p1` → `409` `PRODUCT_RETIRED`.
- `removeProductImage` sobre una imagen de `p1` → `409` `PRODUCT_RETIRED`.
- `reorderProductImages` sobre `p1` → `409` `PRODUCT_RETIRED`.
- `publishProduct` sobre `p1` → `409` `ILLEGAL_STATUS_TRANSITION`: no hay vuelta desde `retired`.
- `createProduct` con `sku: "ACME-RUN-01"` → `409` `SKU_ALREADY_EXISTS`: el sku de un retirado queda
  reservado.
- `retireProduct` sobre un uuid inexistente → `404` `PRODUCT_NOT_FOUND`.
- Un producto en `draft` (`p2`, creado en este flujo) también se retira: `retireProduct` sobre `p2`
  → `200` con `status: "retired"` y su `ProductRetired`. Cubre la transición `draft → retired`.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. Está en `draft` o `active` → `ILLEGAL_STATUS_TRANSITION` (`409`).
3. No cambió desde que se leyó → `CONCURRENT_MODIFICATION` (`409`).

---

### FL-PRD-050: listado de gestión, filtros y paginación

**Given**: en este flujo se crean la marca `b1` y `b2`, las categorías `c1` y `c2`, y cuatro
productos: `p1` (`draft`, `b1`/`c1`), `p2` (`active`, `b1`/`c1`), `p3` (`active`, `b2`/`c2`) y
`p4` (`retired`, `b2`/`c1`), en ese orden temporal de última modificación.

**When**: `listProducts` — `GET /api/v1/products`

**Then**:
1. Status `200`, sobre con `page: 0`, `size: 20`, `totalElements: 4`, `totalPages: 1`.
2. `items` en orden `p4`, `p3`, `p2`, `p1` — descendente por `updatedAt`, el inverso del orden de
   modificación.
3. El listado incluye los tres estados: `draft`, `active` y `retired`.
4. Cada elemento trae `brand` y `category` como objetos anidados, nunca `brandId`/`categoryId`.

**Escenarios encadenados**:
- `?status=active` → `items` con `p3` y `p2`, en ese orden; `totalElements: 2`.
- `?brandId={b1}` → `items` con `p2` y `p1`.
- `?categoryId={c1}&status=active` → `items` solo con `p2`.
- `?brandId={b1}&categoryId={c2}` → `items` vacío, `200`, `totalElements: 0`.
- `?brandId=` un uuid inexistente → `items` vacío, `200`, no `404`.
- `?q=RUN` → los productos cuyo `name` o `sku` contiene `run` ignorando mayúsculas y acentos.
- `?size=2` → `items` con `p4` y `p3`, `totalPages: 2`; `?page=1&size=2` → `p2` y `p1`.
- `?size=500` → `size: 100` en el sobre.

**Notas de determinación**: si dos productos comparten `updatedAt` con precisión de milisegundo, el
desempate es por `id` ascendente. Los datos del flujo se generan con modificaciones separadas para
que el orden por `updatedAt` sea observable sin depender del desempate.

---

## Catálogo público

### FL-PUB-001: el visitante solo ve productos publicados

**Given**: en este flujo se crean `b1`, `c1` y tres productos: `pa` (`active`, `name: "Alfa"`),
`pb` (`active`, `name: "Beta"`) y `pd` (`draft`, `name: "Gamma"`). Además `pr`, que se publica y se
retira. Ninguna llamada de este flujo envía credencial.

**When**: `listPublishedProducts` — `GET /api/v1/catalog/products` (sin cabecera `Authorization`)

**Then**:
1. Status `200` (endpoint público).
2. Sobre con `page: 0`, `size: 20`, `totalElements: 2`, `totalPages: 1`.
3. `items` en orden `Alfa`, `Beta` — ascendente por `name`.
4. `items` **no** contiene `pd` (`draft`) ni `pr` (`retired`).
5. Cada elemento trae `status: "active"`, `brand` y `category` anidados, e `images` como lista
   ordenada por `position`.

**Escenarios encadenados**:
- `getPublishedProduct` sobre `pa` sin credencial → `200` con el cuerpo completo de `pa`.
- `getPublishedProduct` sobre `pd` (`draft`) → `404` `PRODUCT_NOT_FOUND`, no `403`: para el visitante
  un producto no publicado es inexistente.
- `getPublishedProduct` sobre `pr` (`retired`) → `404` `PRODUCT_NOT_FOUND`.
- `getPublishedProduct` sobre un uuid inexistente → `404` `PRODUCT_NOT_FOUND`.
- `?categoryId={c1}` → `Alfa` y `Beta`; `?brandId=` uuid inexistente → `items` vacío, `200`.
- `?size=1` → `items` con `Alfa`, `totalPages: 2`; `?page=1&size=1` → `Beta`; `?page=9` → vacío.
- `?size=500` → `size: 100` en el sobre.

---

### FL-PUB-010: caché de la ficha pública — invalidación y retención

**Given**: en este flujo se crean `b1`, `c1` y el producto `pa`, publicado (`active`,
`name: "Alfa"`, `price.amount: 10.00`).

**When**: se ejecuta la secuencia de **invalidación por cada vía declarada** en
`getPublishedProduct.cache.invalidatedBy`.

**Then**:
1. `getPublishedProduct` sobre `pa` → `200` con `name: "Alfa"` (la caché queda poblada).
2. `updateProduct` sobre `pa` con `name: "Alfa II"` → `200`, y se publica `ProductUpdated`.
3. `getPublishedProduct` sobre `pa`, **dentro** de los 300 s de TTL → `200` con `name: "Alfa II"`:
   `ProductUpdated` invalidó la entrada.
4. `retireProduct` sobre `pa` → `200`, y se publica `ProductRetired`.
5. `getPublishedProduct` sobre `pa`, dentro del TTL → `404` `PRODUCT_NOT_FOUND`: `ProductRetired`
   invalidó la entrada. Sin invalidación seguiría sirviendo la ficha de un producto retirado.

**Escenario de retención** (con el producto `pb`, publicado en este mismo flujo):
1. `getPublishedProduct` sobre `pb` → `200` con `name: "Beta"` (caché poblada).
2. Se cambia el `name` de `pb` por una vía que **no** está en `invalidatedBy`: `updateBrand` sobre la
   marca `b1` a la que pertenece, que altera el objeto `brand` embebido en la ficha y **no** emite
   ningún evento de producto.
3. `getPublishedProduct` sobre `pb`, dentro del TTL → `200` sirviendo el **nombre de marca antiguo**.
   Es lo que demuestra que la caché existe: sin ella, la lectura devolvería ya el nombre nuevo y este
   escenario no distinguiría una caché sana de una inexistente.
4. Pasado el TTL de 300 s, la lectura devuelve el nombre de marca nuevo.

**Notas de determinación**: los pasos "dentro del TTL" se ejecutan sin esperas artificiales; el paso
4 del escenario de retención es el único que requiere superar los 300 s.

---

### FL-PUB-020: navegación de la tienda — marcas y categorías con catálogo publicado

**Given**: en este flujo se crean las marcas `bA` (`name: "Ácme"`), `bB` (`name: "delta"`) y `bC`
(`name: "Zeta"`), y las categorías `cA` (`name: "Calzado"`), `cB` (`name: "Ropa"`) y `cC`
(`name: "Accesorios"`). Se crean además tres productos:

- `p1`, **publicado** (`active`), con marca `bA` y categoría `cA`;
- `p2`, **publicado** (`active`), con marca `bB` y categoría `cA`;
- `p3`, en `draft`, con marca `bC` y categoría `cB`.

La categoría `cC` y ningún producto la referencian.

**When**: `listPublishedBrands` — `GET /api/v1/catalog/brands` (sin cabecera `Authorization`)

**Then**:
1. Status `200` (endpoint público, sin credencial).
2. Sobre con `page: 0`, `size: 20`, `totalElements: 2`, `totalPages: 1`.
3. `items` trae **dos** elementos, en orden `Ácme`, `delta` — ascendente por `name` ignorando
   mayúsculas y acentos.
4. `items` **no** contiene `bC`: su único producto está en `draft`, así que no tiene catálogo
   publicado.
5. Cada elemento trae `id`, `name`, `slug` y —si la tiene— `description`.
6. Ningún elemento trae `createdAt` ni `updatedAt`: la superficie pública los excluye.

**When**: `listPublishedCategories` — `GET /api/v1/catalog/categories` (sin credencial)

**Then**:
7. Status `200`, sobre con `totalElements: 1`.
8. `items` trae solo `cA` (`Calzado`): `cB` solo tiene un producto en `draft` y `cC` no tiene ninguno.
9. Los elementos traen `id`, `name`, `slug`, y no traen `createdAt` ni `updatedAt`.

**Escenarios encadenados**:
1. `publishProduct` sobre `p3` (tras darle imagen y precio positivo) da `200`; `listPublishedBrands`
   pasa a `totalElements: 3` e incluye `Zeta`, y `listPublishedCategories` pasa a `totalElements: 2`
   e incluye `Ropa`. Una faceta **aparece** al publicarse su primer producto.
2. `retireProduct` sobre `p3` da `200`; ambas listas vuelven a `totalElements: 2` y `1`
   respectivamente. Una faceta **desaparece** al retirarse su último producto publicado.
3. `updateBrand` sobre `bA` con `name: "Acme Sport"` da `200`; `listPublishedBrands` devuelve
   **inmediatamente** `Acme Sport`, sin esperar ningún TTL: estas dos listas no llevan caché.
4. `GET /api/v1/catalog/brands?size=1` devuelve `items` con un elemento y `totalPages: 2`;
   `?page=1&size=1` devuelve el siguiente; `?page=9` devuelve `items` vacío con `200`.
5. `GET /api/v1/catalog/brands?size=500` devuelve `size: 100` en el sobre, recortado a `maxSize`.
6. **Marca desactivada**: `deactivateBrand` sobre `bA` da `200`; `listPublishedBrands` deja de
   incluirla y baja a `totalElements: 1`, **pero** `listPublishedProducts` sigue devolviendo `p1` y
   `getPublishedProduct` sobre `p1` sigue respondiendo `200` con su `brand` embebido (`status`
   `inactive`). Desactivar una marca la retira de la navegación, no del catálogo.
7. Lo mismo con `deactivateCategory` sobre `cA` y `listPublishedCategories`.

**Notas de determinación**: la aserción 3 del escenario encadenado es la que distingue "sin caché" de
"con caché invalidada por eventos de producto" — un renombrado de marca no emite ningún evento, así
que con caché el nombre viejo persistiría hasta expirar el TTL.

---

## Superficie servidor-a-servidor

### FL-M2M-001: resolución de un producto por id para otro servidor

**Given**: en este flujo se crean `b1` (`name: "Acme"`), `c1` (`name: "Calzado"`) y tres productos:
`pa` (`active`), `pd` (`draft`) y `pr` (`retired`). Todas las llamadas usan credencial de máquina del
cliente `order-service`.

**When**: `getProductForServices` — `GET /api/v1/internal/products/{pa}`

**Then**:
1. Status `200`.
2. El cuerpo trae `id: pa`, `sku`, `name`, `description`, `price` (`amount` escala 2 + `currency`),
   `status: "active"`, `tags`, `images` (lista ordenada por `position`, con `id`, `position`,
   `altText` y `file` en cada elemento), `createdAt` y `updatedAt`.
3. El cuerpo trae `brand` anidado (`id`, `name: "Acme"`, `slug`) y `category` anidado
   (`id`, `name: "Calzado"`, `slug`), **no** `brandId` ni `categoryId`.
4. Los objetos anidados no traen a su vez relaciones (profundidad 1).
5. El cuerpo es el contrato que documenta `INTEGRATION.md` para este endpoint.

**Escenarios encadenados**:
- `getProductForServices` sobre `pd` → `200` con `status: "draft"`: la superficie M2M **no** filtra
  por estado; el consumidor decide.
- `getProductForServices` sobre `pr` → `200` con `status: "retired"`.
- `getProductForServices` sobre un uuid inexistente → `404` `PRODUCT_NOT_FOUND`.
- **Invalidación de caché**: leer `pa`, `updateProduct` sobre `pa` con `name` nuevo, releer dentro del
  TTL de 300 s → el `name` nuevo. Y `retireProduct` sobre `pa`, releer → `200` con
  `status: "retired"`, no `404` (a diferencia del público).

---

### FL-M2M-010: resolución por lote

**Given**: en este flujo se crean `b1`, `c1` y tres productos `p1` (`active`), `p2` (`draft`) y `p3`
(`retired`). Se usa credencial de máquina de `order-service`.

**When**: `getProductsByIds` — `POST /api/v1/internal/products/batch`
```json
{ "productIds": ["p3", "p1", "<uuid-inexistente>", "p2"] }
```

**Then**:
1. Status `200`.
2. El cuerpo es una **lista** (no un sobre de paginación: la operación no está paginada).
3. La lista trae **tres** elementos, en el orden pedido: `p3`, `p1`, `p2`.
4. El uuid inexistente se **omite** sin error y sin hueco en la lista.
5. Cada elemento trae el mismo contrato completo que `getProductForServices` (aserciones 2–4 de
   FL-M2M-001), incluidos `brand` y `category` anidados.
6. Los tres estados (`retired`, `active`, `draft`) aparecen con su `status` real.

**Escenarios encadenados**:
- Petición con un id **repetido** (`["p1", "p1"]`) → lista con **un** elemento.
- Petición con **solo** ids inexistentes → `200` con lista **vacía**, no `404`.
- `productIds` con 100 ids → `200`.

**Casos borde**:
- `productIds` vacío → `400` (`minItems: 1`).
- `productIds` ausente → `400`.
- `productIds` con 101 ids → `400` (`maxItems: 100`).
- Un elemento que no es uuid → `400`.

---

### FL-M2M-020: autenticación y autorización de la superficie de máquina

**Given**: existe el producto `pa` (`active`), creado en este flujo con credencial de usuario
`catalog-admin`.

**When / Then** — por cada uno de los dos endpoints M2M
(`GET /api/v1/internal/products/{pa}` y `POST /api/v1/internal/products/batch`):

1. **Sin cabecera `Authorization`** → `401`.
2. Con credencial de máquina **sin** el scope `product:read` → `403`.
3. Con credencial de máquina emitida para **otra audiencia** (distinta de `catalog`) → `403`
   (`validateAudience: true`).
4. Con credencial de máquina del cliente `order-service` (scope `product:read`, audiencia `catalog`)
   → `200` con el contrato completo de FL-M2M-001 / FL-M2M-010.

**Notas de determinación**: el fallo de audiencia responde `403`, no `401` — es contrato del
generador, no del diseño.

---

## Seguridad transversal

### FL-SEC-001: autenticación y permisos de la superficie de usuarios

**Given**: existen `b1`, `c1` y el producto `p1` en `draft` con una imagen en su galería, creados en
este flujo con credencial de `catalog-admin`.

**When / Then** — sin cabecera `Authorization`:

1. `POST /api/v1/products` → `401`.
2. `GET /api/v1/products` → `401`.
3. `POST /api/v1/brands` → `401`.
4. `GET /api/v1/catalog/products` → `200`: el catálogo público no exige credencial.

**When / Then** — con credencial de usuario con rol `catalog-editor`
(permisos `product:read`, `product:write`, `brand:read`, `category:read`):

5. `POST /api/v1/products` (`createProduct`) → `201`: tiene `product:write`.
6. `PUT /api/v1/products/{p1}` (`updateProduct`) → `200`.
6b. `POST /api/v1/products/{p1}/images` (`addProductImage`) → `201`: entra en `product:write`.
7. `POST /api/v1/products/{p1}/publish` (`publishProduct`) → `403`: le falta `product:publish`.
8. `POST /api/v1/products/{p1}/retire` (`retireProduct`) → `403`.
9. `POST /api/v1/brands` (`createBrand`) → `403`: le falta `brand:write`.
10. `POST /api/v1/categories/{c1}/deactivate` (`deactivateCategory`) → `403`: le falta `category:write`.
11. `GET /api/v1/brands` (`listBrands`) → `200`: tiene `brand:read`.
12. `GET /api/v1/products` (`listProducts`) → `200`, y devuelve el catálogo **entero**: no hay
    noción de "producto propio", cualquier editor ve y edita todo.

**When / Then** — con credencial de usuario con rol `catalog-admin`:

13. `POST /api/v1/products/{p1}/publish` → `200`.
14. `POST /api/v1/brands` → `201`.
15. `POST /api/v1/categories/{c1}/deactivate` → `200` con `status: "inactive"`.

**Notas de determinación**: `403` es "autenticado sin permiso" y `401` "sin credencial válida"; la
distinción es contrato y ningún escenario la intercambia.
