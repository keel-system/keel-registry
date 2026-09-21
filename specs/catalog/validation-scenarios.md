# catalog — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/catalog v0.1.0. Contrato de validación para la fase de generación.

## Convenciones de determinación

Valen para **todo** el servicio y ningún escenario las repite.

**Tiempo.** Todo instante viaja en UTC ISO-8601 (`2026-05-14T09:21:07.482Z`). `createdAt` y
`updatedAt` los estampa el servidor: se verifican **por forma y por rango** (posteriores al inicio
del escenario, `updatedAt >= createdAt`), jamás por valor literal.

**Identificadores.** Todo `id` es un uuid: se verifica por forma y por **reutilización simbólica**
— el id devuelto en un escenario es el que usan los siguientes de su mismo flujo. Nunca por valor.

**Números.** `price` es decimal de escala **2**, y la escala se **valida, no se ajusta**: `19.99` y
`19.9` y `19` se aceptan; `19.999` se rechaza con `400`. El servicio no hace aritmética sobre el
precio, así que no hay redondeo que fijar.

**Ausencia.** Un campo sin valor **no aparece** en la respuesta; nunca viaja como `null`. Vale para
`description`, `altText` y el `image` de una galería vacía. Un `Then` que enumera el cuerpo da por
ausentes los campos que no nombra.

**Mayúsculas y acentos.** La unicidad de `name` y el filtro `name` ignoran mayúsculas y acentos:
`ACME`, `acme` y `Acmé` son el mismo nombre para la unicidad, y los tres casan el mismo filtro. El
`sku` se normaliza a mayúsculas antes de comprobar su unicidad: `sku-001` y `SKU-001` colisionan.

**Cuerpo de error.** El servicio tiene **una** forma de error, la que emite el generador:
`{timestamp, status, error, code, message, details}` más `correlationId`. Los escenarios afirman
**solo** el `code` y el status HTTP — nunca el texto de `message`, que no es contrato y puede
traducirse o corregirse sin romper a nadie.

**Paginación.** Sobre canónico `{items, page, size, totalElements, totalPages}`, pedido con `page`
(base 0) y `size`. Sin `size`, **20**; un `size` mayor que **100** se **recorta a 100**, no falla.
Toda lista lleva su orden declarado, con el `id` del agregado como desempate final.

**Idempotencia.** `createProduct`, `addProductImage`, `createBrand` y `createCategory` exigen la
cabecera `Idempotency-Key`: sin ella responden `400 IDEMPOTENCY_KEY_REQUIRED`. El reintento con la
misma clave reproduce el status y el cuerpo de la primera respuesta, sin segundo efecto; la misma
clave con un cuerpo distinto es `409 IDEMPOTENCY_KEY_REUSED`; dos peticiones con la misma clave a la
vez, `409 IDEMPOTENCY_KEY_IN_PROGRESS` para la que pierde la carrera.

**Concurrencia.** `Product` lleva control de versión: una escritura sobre una versión obsoleta es
`409 CONCURRENT_MODIFICATION`. `Brand` y `Category` no lo llevan: ahí gana la última escritura.

**Creaciones.** Todo `201` emite la cabecera `Location` con la URI de la petición más el id devuelto.

**Audiencia.** Un token de máquina emitido para otra audiencia responde **403** (es legítimo, pero
no está emitido para este servicio), no `401`.

**Identidades.** Los escenarios solo nombran las que el diseño declara: los roles `catalog-admin` y
`catalog-editor`, y las credenciales de máquina de los clientes `orders-service`, `cart-service` y
`search-service`.

**Archivos.** El campo `image` de una imagen viaja como la **URL pública** del objeto en el bucket
`productImages`, verificada por forma (absoluta, alcanzable sin credencial), nunca por valor.

**Proyecciones.** El cuerpo de cada respuesta se deriva del artefacto. Se definen aquí una vez y los
`Then` las nombran; ninguna respuesta trae campos adicionales a los de su proyección.

- **P-IMG** (`ProductImage`) — `id`, `image`, `position`, `primary`, y `altText` si tiene. **No** lleva
  `productId`: es la back-reference de una entidad hija hacia la raíz de su propio agregado, y el vínculo ya
  está en la estructura (la imagen llega anidada bajo su producto, o se pidió sobre la ruta de ese producto).
- **P-BRAND** (`Brand`) — `id`, `name`, `slug`, y `description` si tiene.
- **P-CAT** (`Category`) — `id`, `name`, `slug`, y `description` si tiene.
- **P-MGMT** (`Product` para el back-office) — `id`, `sku`, `name`, `slug`, `slugFrozen`, `price`,
  `status`, `lockVersion`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `description` si
  tiene, `brand` como objeto **P-BRAND** (no `brandId`), `category` como objeto **P-CAT** (no
  `categoryId`) e `images` como lista de **P-IMG** ordenada por `position`.
- **P-PUB** (`Product` para la tienda) — **P-MGMT** menos `slugFrozen`, `lockVersion`, `createdAt`,
  `updatedAt`, `createdBy` y `updatedBy`.
- **P-M2M** (`Product` para otros servidores) — **P-MGMT** menos `slugFrozen`, `lockVersion`,
  `createdBy` y `updatedBy`; conserva `createdAt` y `updatedAt`, que es de donde el consumidor sabe
  de cuándo es la ficha que recibe.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|---|---|---|
| createProduct | FL-PRD-001, FL-IDM-001 | usuarios |
| updateProduct | FL-PRD-010, FL-PRD-040 | usuarios |
| publishProduct | FL-PRD-020 | usuarios |
| unpublishProduct | FL-PRD-020 | usuarios |
| discontinueProduct | FL-PRD-020 | usuarios |
| reactivateProduct | FL-PRD-020 | usuarios |
| deleteProduct | FL-PRD-030 | usuarios |
| getProduct | FL-PRD-001, FL-PRD-010 | usuarios |
| listProducts | FL-PRD-050 | usuarios |
| addProductImage | FL-IMG-001, FL-IMG-010, FL-IMG-020 | usuarios |
| removeProductImage | FL-IMG-001, FL-IMG-010 | usuarios |
| setPrimaryProductImage | FL-IMG-001, FL-IMG-010 | usuarios |
| reorderProductImages | FL-IMG-001, FL-IMG-010 | usuarios |
| createBrand | FL-BRD-001, FL-BRD-010 | usuarios |
| updateBrand | FL-BRD-001 | usuarios |
| deleteBrand | FL-BRD-001 | usuarios |
| createCategory | FL-CAT-001, FL-CAT-010 | usuarios |
| updateCategory | FL-CAT-001 | usuarios |
| deleteCategory | FL-CAT-001 | usuarios |
| listPublicProducts | FL-PUB-001 | pública (anónima) |
| getPublicProduct | FL-PUB-010, FL-CCH-001 | pública (anónima) |
| listBrands | FL-PUB-020 | pública (anónima) |
| listCategories | FL-PUB-020 | pública (anónima) |
| getProductForServices | FL-SRV-001, FL-CCH-010 | **servidores (M2M)** |
| listProductsBatchForServices | FL-SRV-010 | **servidores (M2M)** |
| **outbox (canal indisponible)** | FL-OBX-001, FL-OBX-002 | — |
| **autorización (401/403)** | FL-SEC-001 | usuarios + M2M |
| **CORS** | FL-SEC-010, FL-SEC-020 | pública (navegador) |

---

## Productos

### FL-PRD-001: alta de producto y publicación de ProductCreated

**Given**: existe la marca `b1` (`name: "Acme"`, `slug: "acme"`) y la categoría `c1`
(`name: "Laptops"`, `slug: "laptops"`). No existe ningún producto con `sku: "SKU-001"`. Credencial de
usuario con rol `catalog-editor`.

**When**: `createProduct` — `POST /api/v1/management/products`, cabecera `Idempotency-Key: k-prd-1`

```json
{ "sku": "SKU-001", "name": "Laptop Pro 14", "description": "Portátil de 14 pulgadas.",
  "price": 1299.00, "brandId": "b1", "categoryId": "c1" }
```

**Then**:
1. Status `201`.
2. Cabecera `Location` con la URI de la petición más el `id` devuelto.
3. El cuerpo es **P-MGMT** y no trae ningún campo adicional.
4. `sku: "SKU-001"`, `name: "Laptop Pro 14"`, `description: "Portátil de 14 pulgadas."`,
   `price: 1299.00` (escala 2), `slug: "laptop-pro-14"`, `status: "draft"`, `slugFrozen: false`.
5. `images` es una lista **vacía** — el producto nace sin galería.
6. `brand` es el objeto **P-BRAND** de `b1` (`name: "Acme"`, `slug: "acme"`); el cuerpo **no** trae
   `brandId`. `category` es el objeto **P-CAT** de `c1`; el cuerpo **no** trae `categoryId`.
7. `createdBy` y `updatedBy` traen la identidad del usuario autenticado del `When`; `createdAt` y
   `updatedAt` se verifican por forma y rango.
8. `getProduct` — `GET /api/v1/management/products/{id}` responde `200` con el mismo cuerpo.
9. Se publica **exactamente un** `ProductCreated` en el canal `productEvents`, con
   `productId` (el devuelto), `sku: "SKU-001"`, `name: "Laptop Pro 14"`,
   `description: "Portátil de 14 pulgadas."`, `slug: "laptop-pro-14"`,
   `price: 1299.00`, `status: "draft"`, `brandId: "b1"`, `brandName: "Acme"`, `categoryId: "c1"`,
   `categoryName: "Laptops"`. El payload **no** trae `primaryImageUrl`: el producto no tiene imágenes.

**Orden de evaluación**:
1. La cabecera `Idempotency-Key` viene → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. El `sku` (normalizado a mayúsculas) no existe → `SKU_ALREADY_EXISTS` (`409`).
3. La marca existe → `BRAND_REFERENCE_NOT_FOUND` (`422`).
4. La categoría existe → `CATEGORY_REFERENCE_NOT_FOUND` (`422`).

**Casos borde**:
- Sin cabecera `Idempotency-Key` → `400` (`IDEMPOTENCY_KEY_REQUIRED`).
- `sku: "sku-001"` con `SKU-001` ya existente → `409` (`SKU_ALREADY_EXISTS`): se normaliza antes.
- `brandId` inexistente → `422` (`BRAND_REFERENCE_NOT_FOUND`).
- `categoryId` inexistente → `422` (`CATEGORY_REFERENCE_NOT_FOUND`).
- `brandId` **y** `categoryId` inexistentes a la vez → `422` (`BRAND_REFERENCE_NOT_FOUND`): la
  guarda 3 precede a la 4.
- `sku` ya existente **y** `brandId` inexistente → `409` (`SKU_ALREADY_EXISTS`): la guarda 2 precede.
- `sku: "ab"` → `400` (no casa el patrón: mínimo 3 caracteres).
- `price: 19.999` → `400` (escala máxima 2).
- `price` ausente → `400` (requerido).
- `name` de 141 caracteres → `400` (`maxLength: 140`).

**Notas de determinación**: el `slug` se deriva del `name` normalizado a kebab-case; con un segundo
producto de nombre `"Laptop Pro 14"` el segundo slug es `"laptop-pro-14-2"`.

### FL-PRD-010: edición de la ficha y ProductUpdated

**Given**: el flujo crea `b1`, `c1`, la marca `b2` (`name: "Globex"`) y el producto `p1` con
`createProduct` (`sku: "SKU-010"`, `name: "Teclado K1"`, `description: "Mecánico."`, `price: 79.90`,
marca `b1`). Credencial con rol `catalog-editor`.

**When**: `updateProduct` — `PUT /api/v1/management/products/{p1}`

```json
{ "name": "Teclado K1 Pro", "price": 89.90, "brandId": "b2", "categoryId": "c1" }
```

**Then**:
1. Status `200`.
2. El cuerpo es **P-MGMT** y no trae ningún campo adicional.
3. `name: "Teclado K1 Pro"`, `price: 89.90`, `slug: "teclado-k1-pro"` — el slug sigue al nombre
   porque `slugFrozen` es `false`.
4. El cuerpo **no** trae `description`: la petición no la mandó y la entrada es la representación
   completa, así que la descripción anterior se **vació**.
5. `brand` es el objeto **P-BRAND** de `b2`; `sku` sigue siendo `"SKU-010"` (no es parte de la entrada).
6. `updatedBy` es la identidad del `When` y `updatedAt` es posterior al `createdAt`.
7. `getProduct` sobre `p1` devuelve el mismo cuerpo.
8. Se publica **exactamente un** `ProductUpdated` en `productEvents` con `productId: p1`,
   `name: "Teclado K1 Pro"`, `price: 89.90`, `status: "draft"`, `brandId: "b2"`,
   `brandName: "Globex"`, y **sin** `description`.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. La marca existe → `BRAND_REFERENCE_NOT_FOUND` (`422`).
3. La categoría existe → `CATEGORY_REFERENCE_NOT_FOUND` (`422`).
4. Un producto `active` no se queda a precio cero → `PRICE_REQUIRED_FOR_ACTIVE_PRODUCT` (`422`).

**Ramas condicionales**: el `slug` se recalcula solo si `slugFrozen` es `false`. Sobre un producto ya
publicado (ver FL-PRD-020) el mismo `When` cambia el `name` y **conserva** el `slug`.

**Casos borde**:
- `productId` inexistente → `404` (`PRODUCT_NOT_FOUND`).
- Sobre un producto `active`, `price: 0` → `422` (`PRICE_REQUIRED_FOR_ACTIVE_PRODUCT`).
- Sobre un producto `draft`, `price: 0` → `200`: en borrador el precio cero es legítimo.
- Sobre un producto `discontinued`, la edición se **acepta** (`200`): su ficha la siguen mostrando
  los pedidos históricos.

### FL-PRD-020: ciclo de vida completo del producto

**Given**: el flujo crea `b1`, `c1` y el producto `p1` en `draft` (`sku: "SKU-020"`,
`name: "Monitor M27"`, `price: 249.00`) con **una** imagen en su galería (vía `addProductImage`).
Credencial con rol `catalog-editor`.

**When**: `publishProduct` — `POST /api/v1/management/products/{p1}/publish`

**Then**:
1. Status `200`, cuerpo **P-MGMT**.
2. `status: "active"` y `slugFrozen: true` — el slug queda congelado en `"monitor-m27"`.
3. Se publica **exactamente un** `ProductStatusChanged` en `productEvents` con `productId: p1`,
   `previousStatus: "draft"`, `status: "active"`, `price: 249.00` y `primaryImageUrl` con la URL de
   la única imagen.
4. `getPublicProduct` — `GET /api/v1/products/monitor-m27` responde `200`: ya está en la tienda.

**When**: `updateProduct` sobre `p1` con `name: "Monitor M27 UHD"`

**Then**:
5. `name: "Monitor M27 UHD"` y `slug` sigue siendo `"monitor-m27"`: congelado, los enlaces
   compartidos no se rompen.
6. `GET /api/v1/products/monitor-m27-uhd` responde `404` (`PRODUCT_NOT_FOUND`).

**When**: `unpublishProduct` — `POST /api/v1/management/products/{p1}/unpublish`

**Then**:
7. Status `200`, `status: "draft"`, y `slugFrozen` sigue en `true` — no se deshace.
8. `ProductStatusChanged` con `previousStatus: "active"`, `status: "draft"`.
9. `GET /api/v1/products/monitor-m27` responde `404`: sale de la tienda.

**When**: `publishProduct` otra vez, y después
`discontinueProduct` — `POST /api/v1/management/products/{p1}/discontinue`

**Then**:
10. Status `200`, `status: "discontinued"`.
11. `ProductStatusChanged` con `previousStatus: "active"`, `status: "discontinued"`.
12. `GET /api/v1/products/monitor-m27` responde `404`; `getProduct` sigue respondiendo `200`.

**When**: `reactivateProduct` — `POST /api/v1/management/products/{p1}/reactivate`

**Then**:
13. Status `200`, `status: "active"`.
14. `ProductStatusChanged` con `previousStatus: "discontinued"`, `status: "active"`.
15. `GET /api/v1/products/monitor-m27` vuelve a responder `200`.

**Orden de evaluación** (`publishProduct`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. Está en `draft` → `PRODUCT_NOT_IN_DRAFT` (`409`).
3. Su `price` es mayor que cero → `PRODUCT_PRICE_NOT_SET` (`422`).
4. Tiene al menos una imagen → `PRODUCT_HAS_NO_IMAGES` (`422`).

**Casos borde** (transiciones inválidas: cada operación aplicada desde un estado que no está en su `from`):
- `publishProduct` sobre un producto `active` → `409` (`PRODUCT_NOT_IN_DRAFT`).
- `publishProduct` sobre un producto `discontinued` → `409` (`PRODUCT_NOT_IN_DRAFT`).
- `unpublishProduct` sobre un `draft` → `409` (`PRODUCT_NOT_ACTIVE`).
- `discontinueProduct` sobre un `draft` → `409` (`PRODUCT_NOT_ACTIVE`).
- `reactivateProduct` sobre un `active` → `409` (`PRODUCT_NOT_DISCONTINUED`).
- `publishProduct` sobre un `draft` **sin imágenes** → `422` (`PRODUCT_HAS_NO_IMAGES`).
- `publishProduct` sobre un `draft` con `price: 0` y sin imágenes → `422`
  (`PRODUCT_PRICE_NOT_SET`): la guarda 3 precede a la 4.
- `reactivateProduct` sobre un `discontinued` con `price: 0` → `422` (`PRODUCT_PRICE_NOT_SET`).
- `reactivateProduct` sobre un `discontinued` sin imágenes → `422` (`PRODUCT_HAS_NO_IMAGES`).
- `publishProduct` con `productId` inexistente → `404` (`PRODUCT_NOT_FOUND`).
- `unpublishProduct` / `discontinueProduct` / `reactivateProduct` con `productId` inexistente →
  `404` (`PRODUCT_NOT_FOUND`).

**Notas de determinación**: los tres estados del lifecycle (`draft`, `active`, `discontinued`) los
alcanza este flujo, y las cuatro transiciones declaradas se ejercitan en los pasos 1, 7, 10 y 13.

### FL-PRD-030: borrado de un borrador y su vía de baja

**Given**: el flujo crea `b1`, `c1`, el producto `p1` en `draft` (nunca publicado, `sku: "SKU-030"`)
con una imagen, y el producto `p2` publicado y luego despublicado (`slugFrozen: true`). Credencial
con rol `catalog-admin`.

**When**: `deleteProduct` — `DELETE /api/v1/management/products/{p1}`

**Then**:
1. Status `204` y **sin cuerpo**.
2. `getProduct` sobre `p1` responde `404` (`PRODUCT_NOT_FOUND`).
3. La imagen de `p1` ya no es alcanzable por su URL pública: el objeto se retiró del bucket
   `productImages` junto con el producto.
4. Se publica **exactamente un** `ProductDeleted` en `productEvents` con `productId: p1`,
   `sku: "SKU-030"` y `name`. El payload **no** trae precio, marca ni categoría: el producto ya no
   existe y lleva lo justo para localizar la copia y borrarla.

**Orden de evaluación**:
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. Está en `draft` y nunca se publicó (`slugFrozen: false`) → `PRODUCT_NOT_DELETABLE` (`409`).

**Casos borde**:
- `deleteProduct` sobre `p2` (en `draft`, pero ya estuvo publicado) → `409`
  (`PRODUCT_NOT_DELETABLE`): otros servidores guardan su id.
- `deleteProduct` sobre un producto `active` → `409` (`PRODUCT_NOT_DELETABLE`).
- `deleteProduct` sobre un producto `discontinued` → `409` (`PRODUCT_NOT_DELETABLE`).
- `deleteProduct` con `productId` inexistente → `404` (`PRODUCT_NOT_FOUND`).
- `deleteProduct` con credencial de `catalog-editor` → `403`: el permiso `product:delete` solo lo
  tiene `catalog-admin`.

### FL-PRD-040: dos ediciones concurrentes del mismo producto

**Given**: el flujo crea `b1`, `c1` y el producto `p1` (`name: "Silla S1"`, `price: 149.00`). Se lee
`p1` con `getProduct` **dos veces**, obteniendo dos lecturas con el mismo `lockVersion`.

**When**: dos `updateProduct` sobre `p1` **a la vez**, cada uno partiendo de su lectura: uno con
`name: "Silla S1 Azul"` y otro con `price: 159.00`.

**Then**:
1. Exactamente una de las dos responde `200`.
2. La otra responde `409` con `code: CONCURRENT_MODIFICATION` — nadie pisa en silencio el cambio ajeno.
3. `getProduct` sobre `p1` devuelve el estado de la ganadora: o `name: "Silla S1 Azul"` con
   `price: 149.00`, o `name: "Silla S1"` con `price: 159.00`. Nunca una mezcla de los dos.
4. Su `lockVersion` es **exactamente uno** mayor que el de las lecturas del `Given`.
5. En `productEvents` hay **exactamente un** `ProductUpdated` de `p1`, coherente con la ganadora.

**Notas de determinación**: el `Then` no depende de quién gane. Las aserciones 4 y 5 son las que no
admiten disyunción y las que hacen que el escenario pueda fallar.

### FL-PRD-050: listado de back-office — orden, paginación y filtros

**Given**: el flujo crea `b1` (`"Acme"`), `b2` (`"Globex"`), `c1` (`"Laptops"`), `c2` (`"Monitores"`)
y **25** productos: `p1..p10` de `b1`/`c1`, `p11..p20` de `b2`/`c2`, `p21..p25` de `b1`/`c2`. De
ellos, `p1..p5` se publican (`active`), `p6` se publica y se descontinúa, y el resto queda en `draft`.
Primero se crean los 25 en orden y **después** se aplican las transiciones, en este orden: `p1..p5`
se publican, luego `p6` se publica y se descontinúa. Cada transición actualiza `updatedAt`, así que
`p6` (la última mutación) es el de `updatedAt` más reciente. Credencial con rol `catalog-editor`.

**When**: `listProducts` — `GET /api/v1/management/products`

**Then**:
1. Status `200` y el sobre `{items, page, size, totalElements, totalPages}`.
2. `page: 0`, `size: 20`, `totalElements: 25`, `totalPages: 2`.
3. `items` trae 20 elementos, cada uno con la proyección **P-MGMT**.
4. El orden es `updatedAt` descendente, con el `id` como desempate: el primer elemento es `p6`
   (la última mutación del Given).
5. Se devuelven productos en los **tres** estados: entre los `items` hay `status` `draft`, `active` y
   `discontinued`.

**When**: `GET /api/v1/management/products?page=1`

**Then**:
6. `page: 1`, `items` con los **5** restantes, sin repetir ninguno de la primera página.

**When**: `GET /api/v1/management/products?page=5`

**Then**:
7. `200` con `items` vacío, `page: 5`, `totalElements: 25`.

**When**: `GET /api/v1/management/products?size=500`

**Then**:
8. `size: 100` — el tope recorta la petición, no la rechaza.

**When**: `GET /api/v1/management/products?status=active&brandId={b1}&categoryId={c1}&name=lap`

**Then**:
9. Solo salen productos que cumplen **todos** los filtros a la vez (AND): los `active` de `b1` en
   `c1` cuyo nombre contiene `"lap"`.
10. `GET ...?name=LAP` y `GET ...?name=láp` devuelven el mismo `totalElements`.
11. `GET ...?brandId=<uuid inexistente>` responde `200` con `items` vacío y `totalElements: 0` — un
    filtro sin correspondencia no es un error.
12. `GET ...?sku=SKU-0001` devuelve el producto de ese sku, o la página vacía si no existe.

**Notas de determinación (coste)**: el listado resuelve `brand`, `category` e `images` por elemento.
El trabajo de la operación **no crece con el tamaño de la página**: se compara `size=2` con `size=20`
y el coste por petición no es proporcional al número de elementos devueltos. Se afirma por forma,
nunca fijando un número absoluto de consultas.

---

## Galería de imágenes

### FL-IMG-001: galería de un producto — alta, principal, orden y baja

**Given**: el flujo crea `b1`, `c1` y el producto `p1` en `draft` sin imágenes. Credencial con rol
`catalog-editor`.

**When**: `addProductImage` — `POST /api/v1/management/products/{p1}/images`, cabecera
`Idempotency-Key: k-img-1`, cuerpo multipart con un `image` de tipo `image/jpeg` de 120 KB y
`altText: "Vista frontal"`.

**Then**:
1. Status `201` y cabecera `Location` con la URI de la petición más el `id` devuelto.
2. El cuerpo es **P-IMG**: `id`, `image` (URL pública, por forma),
   `altText: "Vista frontal"`, `position: 0`, `primary: true`.
3. `primary` es `true` **aunque la petición no lo pidiera**: es la primera imagen del producto.
4. El `image` devuelto es alcanzable **sin credencial**: el bucket `productImages` es público.
5. `getProduct` sobre `p1` trae `images` con un elemento igual al cuerpo del paso 2.
6. Se publica **exactamente un** `ProductUpdated` en `productEvents`, con `primaryImageUrl` igual al
   `image` de esta imagen.

**When**: `addProductImage` dos veces más (clave `k-img-2` y `k-img-3`), sin `primary`

**Then**:
7. Las dos responden `201` con `position: 1` y `position: 2`, y `primary: false`: la principal sigue
   siendo la primera.
8. `getProduct` trae `images` con tres elementos, ordenados por `position` ascendente.

**When**: `setPrimaryProductImage` — `PUT /api/v1/management/products/{p1}/images/{img3}/primary`

**Then**:
9. Status `204` y sin cuerpo.
10. `getProduct` trae exactamente **una** imagen con `primary: true`, y es `img3`; la que lo era ha
    dejado de serlo en la misma transacción.
11. Se publica un `ProductUpdated` cuyo `primaryImageUrl` es el `image` de `img3`.
12. Repetir el mismo `When` responde `204` y no cambia nada: marcar como principal la que ya lo es no
    es un error.

**When**: `reorderProductImages` — `PUT /api/v1/management/products/{p1}/images/order` con
`{ "imageIds": ["{img3}", "{img1}", "{img2}"] }`

**Then**:
13. Status `204`.
14. `getProduct` trae `images` con `position` `0`, `1`, `2` en ese orden: `img3`, `img1`, `img2`.
15. `img3` sigue siendo la `primary`: reordenar no cambia cuál es la principal.
16. Se publica un `ProductUpdated`.

**When**: `removeProductImage` — `DELETE /api/v1/management/products/{p1}/images/{img3}`

**Then**:
17. Status `204` y sin cuerpo.
18. El objeto de `img3` ya no es alcanzable por su URL pública.
19. `getProduct` trae `images` con dos elementos, y la `primary` es ahora la de **menor** `position`
    de las que quedan (`img1`): el ascenso es predecible, y por eso el `204` no necesita cuerpo.
20. Se publica un `ProductUpdated` con el `primaryImageUrl` de `img1`.

**Orden de evaluación** (`addProductImage`):
1. La cabecera `Idempotency-Key` viene → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
3. No está `discontinued` → `PRODUCT_DISCONTINUED` (`409`).
4. No tiene ya diez imágenes → `TOO_MANY_PRODUCT_IMAGES` (`422`).
5. El archivo no supera los 5 MB del bucket → `FILE_TOO_LARGE` (`413`).
6. El content-type está entre los admitidos → `UNSUPPORTED_CONTENT_TYPE` (`415`).

**Orden de evaluación** (`removeProductImage`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. No está `discontinued` → `PRODUCT_DISCONTINUED` (`409`).
3. La imagen existe y es de ese producto → `PRODUCT_IMAGE_NOT_FOUND` (`404`).
4. Si el producto está `active`, no es la última de su galería → `LAST_IMAGE_OF_ACTIVE_PRODUCT` (`409`).

**Casos borde**:
- Subir una undécima imagen → `422` (`TOO_MANY_PRODUCT_IMAGES`).
- Subir un `image/jpeg` de 6 MB → `413` (`FILE_TOO_LARGE`).
- Subir un `application/pdf` → `415` (`UNSUPPORTED_CONTENT_TYPE`).
- Subir un `image/gif` → `415` (`UNSUPPORTED_CONTENT_TYPE`): solo jpeg, png y webp.
- `addProductImage` con `productId` inexistente → `404` (`PRODUCT_NOT_FOUND`).
- `removeProductImage` con `imageId` de **otro** producto → `404` (`PRODUCT_IMAGE_NOT_FOUND`).
- `setPrimaryProductImage` con `imageId` inexistente → `404` (`PRODUCT_IMAGE_NOT_FOUND`).
- Sobre un producto **publicado** con una sola imagen, `removeProductImage` → `409`
  (`LAST_IMAGE_OF_ACTIVE_PRODUCT`); sobre el mismo producto en `draft`, `204`.
- `reorderProductImages` con una lista que **omite** una imagen del producto → `422`
  (`IMAGE_SET_MISMATCH`).
- `reorderProductImages` con una lista que **repite** un id → `422` (`IMAGE_SET_MISMATCH`).
- `reorderProductImages` con un id ajeno al producto → `422` (`IMAGE_SET_MISMATCH`).
- `reorderProductImages` con `imageIds: []` → `400` (`minItems: 1`).
- `reorderProductImages` con 11 ids → `400` (`maxItems: 10`).
- `altText` de 201 caracteres → `400` (`maxLength: 200`).

### FL-IMG-010: la galería de un producto descontinuado no se toca

**Given**: el flujo crea `b1`, `c1` y el producto `p1` con dos imágenes, lo publica y lo descontinúa.
Credencial con rol `catalog-editor`.

**When**: las cuatro operaciones de galería sobre `p1`

**Then**:
1. `addProductImage` → `409` con `code: PRODUCT_DISCONTINUED`.
2. `removeProductImage` → `409` con `code: PRODUCT_DISCONTINUED`.
3. `setPrimaryProductImage` → `409` con `code: PRODUCT_DISCONTINUED`.
4. `reorderProductImages` → `409` con `code: PRODUCT_DISCONTINUED`.
5. `getProduct` sobre `p1` sigue trayendo sus dos imágenes intactas, con el mismo orden y la misma
   principal que antes del `When`.
6. `updateProduct` sobre `p1` responde `200`: la ficha sí se edita, la galería no.
7. En `productEvents` no aparece ningún `ProductUpdated` derivado de los cuatro rechazos.

### FL-IMG-020: idempotencia de la subida de una imagen

**Given**: el flujo crea `b1`, `c1` y el producto `p1` sin imágenes. Credencial con rol
`catalog-editor`.

**When**: `addProductImage` con `Idempotency-Key: k-img-r` y un `image/png` de 200 KB, y **después**
la misma petición con la **misma** clave y el mismo cuerpo.

**Then**:
1. La primera responde `201` con su cuerpo **P-IMG**.
2. La segunda responde **el mismo status y el mismo cuerpo** que la primera, con el mismo `id`.
3. `getProduct` sobre `p1` trae `images` con **exactamente un** elemento: no hay segunda imagen.
4. En `productEvents` hay **exactamente un** `ProductUpdated`.

**When**: la misma petición con una clave **distinta** (`k-img-r2`) y el mismo cuerpo

**Then**:
5. Responde `201` con un `id` **nuevo**: dos fotos iguales son legítimas, y la clave es lo único que
   distingue un reintento de una subida a propósito.
6. `getProduct` trae `images` con dos elementos.

**When**: la clave `k-img-r` otra vez, con un `altText` distinto

**Then**:
7. Responde `409` con `code: IDEMPOTENCY_KEY_REUSED`.
8. `getProduct` sigue trayendo dos imágenes.

**When**: **dos peticiones a la vez** con la misma clave `k-img-race` y el mismo cuerpo

**Then**:
9. O las dos devuelven la misma respuesta de creación, o una devuelve `201` y la otra `409` con
   `code: IDEMPOTENCY_KEY_IN_PROGRESS`.
10. Sea quien sea la ganadora, `getProduct` trae **exactamente una** imagen más que antes del `When`.
11. Y en `productEvents` hay **exactamente un** `ProductUpdated` nuevo.

**Notas de determinación**: las aserciones 10 y 11 son las que no dependen del ganador; sin ellas el
escenario solo enumeraría desenlaces admisibles y no podría fallar.

---

## Marcas

### FL-BRD-001: alta, edición y borrado de una marca

**Given**: no existe ninguna marca. Credencial con rol `catalog-admin`.

**When**: `createBrand` — `POST /api/v1/management/brands`, `Idempotency-Key: k-brd-1`

```json
{ "name": "Acme", "description": "Fabricante de periféricos." }
```

**Then**:
1. Status `201` y cabecera `Location` con la URI de la petición más el `id`.
2. El cuerpo es **P-BRAND**: `id`, `name: "Acme"`, `slug: "acme"`,
   `description: "Fabricante de periféricos."`. **No** trae `createdAt`, `updatedAt`, `createdBy` ni
   `updatedBy`: el rastro de autoría solo está declarado en `Product`.
3. `GET /api/v1/brands` incluye la marca creada.
4. Se publica **exactamente un** `BrandCreated` en el canal `taxonomyEvents`, con `brandId`,
   `name: "Acme"`, `slug: "acme"` y `description`.

**When**: `updateBrand` — `PUT /api/v1/management/brands/{b1}` con `{ "name": "Acme Corp" }`

**Then**:
5. Status `200`, cuerpo **P-BRAND** con `name: "Acme Corp"` y `slug: "acme-corp"`: al cambiar el
   nombre cambia la URL pública de la marca.
6. El cuerpo **no** trae `description`: la entrada es la representación completa y la reseña se vació.
7. Se publica un `BrandUpdated` en `taxonomyEvents` con el nombre y el slug nuevos.

**When**: se crea un producto `p1` con `brandId: b1` y se intenta
`deleteBrand` — `DELETE /api/v1/management/brands/{b1}`

**Then**:
8. Status `409` con `code: BRAND_IN_USE`.
9. `GET /api/v1/brands` sigue incluyendo `b1`.

**When**: se borra `p1` (`deleteProduct`) y se repite `deleteBrand`

**Then**:
10. Status `204` y sin cuerpo.
11. `GET /api/v1/brands` ya no incluye `b1`.
12. Se publica **exactamente un** `BrandDeleted` en `taxonomyEvents` con `brandId` y `name`.

**Orden de evaluación** (`createBrand`):
1. La cabecera `Idempotency-Key` viene → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. No hay otra marca con ese `name` → `BRAND_NAME_ALREADY_EXISTS` (`409`).
3. El slug derivado no lo usa otra marca → `BRAND_SLUG_ALREADY_EXISTS` (`409`).

**Orden de evaluación** (`deleteBrand`):
1. La marca existe → `BRAND_NOT_FOUND` (`404`).
2. Ningún producto la referencia → `BRAND_IN_USE` (`409`).

**Casos borde**:
- `createBrand` con `name: "ACME"` existiendo `"Acme"` → `409` (`BRAND_NAME_ALREADY_EXISTS`): la
  unicidad ignora mayúsculas.
- `createBrand` con `name: "Acmé"` existiendo `"Acme"` → `409` (`BRAND_NAME_ALREADY_EXISTS`): y
  también ignora acentos.
- `createBrand` con `name: "Acme!"` existiendo `"Acme"` → `409` (`BRAND_SLUG_ALREADY_EXISTS`): los
  nombres **no** coinciden, pero los dos dan el slug `"acme"`. Es el `code` que distingue los dos
  fallos.
- `updateBrand` con el nombre de otra marca → `409` (`BRAND_NAME_ALREADY_EXISTS`).
- `updateBrand` con `brandId` inexistente → `404` (`BRAND_NOT_FOUND`).
- `deleteBrand` con `brandId` inexistente → `404` (`BRAND_NOT_FOUND`).
- `name` de 81 caracteres → `400` (`maxLength: 80`).
- `name` ausente → `400` (requerido).
- Cualquiera de las tres con credencial de `catalog-editor` → `403`: la taxonomía es del admin.

### FL-BRD-010: idempotencia del alta de marca

**Given**: no existe ninguna marca. Credencial con rol `catalog-admin`.

**When**: `createBrand` con `Idempotency-Key: k-brd-r` y `{ "name": "Globex" }`, y **después** la
misma petición con la misma clave.

**Then**:
1. La primera responde `201`; la segunda, el **mismo** status y el **mismo** cuerpo, con el mismo `id`.
2. `GET /api/v1/brands` trae `totalElements: 1`.
3. En `taxonomyEvents` hay **exactamente un** `BrandCreated`.

**When**: **dos peticiones a la vez** con la clave `k-brd-race` y `{ "name": "Initech" }`

**Then**:
4. O las dos devuelven la misma respuesta de creación, o una devuelve `201` y la otra `409` con
   `code: IDEMPOTENCY_KEY_IN_PROGRESS`.
5. `GET /api/v1/brands?name=Initech` devuelve **exactamente una** marca, sea quien sea el ganador.
6. En `taxonomyEvents` hay **exactamente un** `BrandCreated` de `"Initech"`.

---

## Categorías

### FL-CAT-001: alta, edición y borrado de una categoría

**Given**: no existe ninguna categoría. Credencial con rol `catalog-admin`.

**When**: `createCategory` — `POST /api/v1/management/categories`, `Idempotency-Key: k-cat-1`

```json
{ "name": "Laptops", "description": "Portátiles de todas las gamas." }
```

**Then**:
1. Status `201` y cabecera `Location`.
2. El cuerpo es **P-CAT**: `id`, `name: "Laptops"`, `slug: "laptops"`, `description`.
3. `GET /api/v1/categories` incluye la categoría creada.
4. Se publica **exactamente un** `CategoryCreated` en `taxonomyEvents` con `categoryId`, `name`,
   `slug` y `description`.

**When**: `updateCategory` — `PUT /api/v1/management/categories/{c1}` con
`{ "name": "Portátiles", "description": "Gama completa." }`

**Then**:
5. Status `200`, `name: "Portátiles"`, `slug: "portatiles"` — el slug pierde el acento.
6. Se publica un `CategoryUpdated` con el nombre y el slug nuevos.

**When**: se crea un producto en `c1` y se intenta `deleteCategory`

**Then**:
7. Status `409` con `code: CATEGORY_IN_USE`.

**When**: se borra el producto y se repite `deleteCategory`

**Then**:
8. Status `204` y sin cuerpo.
9. `GET /api/v1/categories` ya no la incluye.
10. Se publica **exactamente un** `CategoryDeleted` con `categoryId` y `name`.

**Orden de evaluación** (`createCategory`):
1. La cabecera `Idempotency-Key` viene → `IDEMPOTENCY_KEY_REQUIRED` (`400`).
2. No hay otra categoría con ese `name` → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).
3. El slug derivado no lo usa otra categoría → `CATEGORY_SLUG_ALREADY_EXISTS` (`409`).

**Orden de evaluación** (`deleteCategory`):
1. La categoría existe → `CATEGORY_NOT_FOUND` (`404`).
2. Ningún producto la referencia → `CATEGORY_IN_USE` (`409`).

**Casos borde**:
- `createCategory` con `name: "LAPTOPS"` existiendo `"Laptops"` → `409`
  (`CATEGORY_NAME_ALREADY_EXISTS`).
- `createCategory` con `name: "Laptops+"` existiendo `"Laptops"` → `409`
  (`CATEGORY_SLUG_ALREADY_EXISTS`).
- `updateCategory` con `categoryId` inexistente → `404` (`CATEGORY_NOT_FOUND`).
- `deleteCategory` con `categoryId` inexistente → `404` (`CATEGORY_NOT_FOUND`).
- Cualquiera con credencial de `catalog-editor` → `403`.

### FL-CAT-010: idempotencia del alta de categoría

**Given**: no existe ninguna categoría. Credencial con rol `catalog-admin`.

**When**: `createCategory` con `Idempotency-Key: k-cat-r` y `{ "name": "Monitores" }`, repetida con
la misma clave; y después **dos peticiones a la vez** con la clave `k-cat-race` y
`{ "name": "Teclados" }`.

**Then**:
1. El reintento secuencial devuelve el mismo status y el mismo cuerpo, con el mismo `id`.
2. `GET /api/v1/categories?...` refleja **una sola** categoría `"Monitores"`.
3. En la carrera, o las dos devuelven la misma respuesta, o una devuelve `201` y la otra `409`
   (`IDEMPOTENCY_KEY_IN_PROGRESS`).
4. Existe **exactamente una** categoría `"Teclados"`, y en `taxonomyEvents` **un solo**
   `CategoryCreated` de ese nombre.

---

## Idempotencia del alta de producto

### FL-IDM-001: reintento, reutilización de clave y carrera en createProduct

**Given**: el flujo crea `b1` y `c1`. No existe ningún producto. Credencial con rol `catalog-editor`.

**When**: `createProduct` con `Idempotency-Key: k-1` y el cuerpo de `SKU-100`, y **después** la misma
petición con la misma clave y el mismo cuerpo.

**Then**:
1. La primera responde `201` con su cuerpo **P-MGMT**.
2. La segunda responde el **mismo** status `201` y el **mismo** cuerpo, con el mismo `id` — no es un
   rechazo, es la respuesta original reproducida.
3. `listProducts` devuelve `totalElements: 1`.
4. En `productEvents` hay **exactamente un** `ProductCreated`.

**When**: la misma petición con `Idempotency-Key: k-2` (clave distinta) y el mismo `sku`

**Then**:
5. Responde `409` con `code: SKU_ALREADY_EXISTS`: la clave nueva hace que la operación se ejecute de
   verdad, y ahí se topa con la unicidad del sku.

**When**: `createProduct` con `Idempotency-Key: k-1` (la primera clave) y un `sku` **distinto**

**Then**:
6. Responde `409` con `code: IDEMPOTENCY_KEY_REUSED`.
7. `listProducts` sigue devolviendo `totalElements: 1`.

**When**: `createProduct` **sin** cabecera `Idempotency-Key`

**Then**:
8. Responde `400` con `code: IDEMPOTENCY_KEY_REQUIRED`.
9. `listProducts` sigue devolviendo `totalElements: 1`.

**When**: **dos peticiones a la vez** con `Idempotency-Key: k-race` y el mismo cuerpo (`SKU-200`)

**Then**:
10. O las dos devuelven la misma respuesta de creación, o una devuelve `201` y la otra `409` con
    `code: IDEMPOTENCY_KEY_IN_PROGRESS`.
11. `listProducts?sku=SKU-200` devuelve **exactamente un** producto, sea quien sea el ganador.
12. En `productEvents` hay **exactamente un** `ProductCreated` de `SKU-200`.

**Notas de determinación**: la 11 y la 12 son las aserciones que no dependen del ganador. El
reintento secuencial (pasos 1-2) encuentra el registro de la clave ya confirmado; la carrera (10)
cae en la ventana en la que todavía no lo está, que es donde vive el fallo real.

---

## Tienda pública

### FL-PUB-001: escaparate con filtros de categoría, marca, nombre y precio

**Given**: el flujo crea `b1` (`"Acme"`, slug `acme`), `b2` (`"Globex"`, slug `globex`), `c1`
(`"Laptops"`, slug `laptops`), `c2` (`"Monitores"`, slug `monitores`) y **25** productos con una
imagen cada uno: `q1..q22` publicados (`active`), `q23` en `draft`, `q24` `discontinued` y `q25` en
`draft`. Entre los publicados, `q1` (`"Laptop A"`, `b1`/`c1`, `price: 700.00`), `q2`
(`"Laptop B"`, `b1`/`c1`, `price: 1200.00`), `q3` (`"Monitor C"`, `b2`/`c2`, `price: 300.00`). **Sin
credencial**: la tienda es anónima.

**When**: `listPublicProducts` — `GET /api/v1/products`

**Then**:
1. Status `200` **sin credencial** y el sobre `{items, page, size, totalElements, totalPages}`.
2. `page: 0`, `size: 20`, `totalElements: 22`, `totalPages: 2` — solo los `active`.
3. Ni `q23`, ni `q24`, ni `q25` están entre los `items`: los borradores y los descontinuados nunca
   salen aquí.
4. Cada elemento tiene la proyección **P-PUB** y **no** trae `slugFrozen`, `lockVersion`,
   `createdAt`, `updatedAt`, `createdBy` ni `updatedBy`.
5. `brand` y `category` vienen como objetos anidados (**P-BRAND**, **P-CAT**), no como `brandId` ni
   `categoryId`.
6. El orden es `name` ascendente con el `id` como desempate: `"Laptop A"` va antes que `"Laptop B"`,
   y las dos antes que `"Monitor C"`.

**When**: `GET /api/v1/products?page=1`, `GET /api/v1/products?page=9` y
`GET /api/v1/products?size=500`

**Then**:
7. La segunda página trae los 2 restantes, sin repetir ninguno de la primera.
8. `page=9` responde `200` con `items` vacío y `totalElements: 22`.
9. `size=500` responde con `size: 100`.

**When**: los filtros, de uno en uno y combinados

**Then**:
10. `?categorySlug=laptops` devuelve solo productos de `c1`.
11. `?brandSlug=acme` devuelve solo productos de `b1`.
12. `?name=laptop` devuelve `"Laptop A"` y `"Laptop B"`; `?name=LAPTOP` y `?name=láptop` devuelven
    el mismo `totalElements`.
13. `?minPrice=500&maxPrice=1200` incluye `q2` (`1200.00`, extremo **inclusivo**) y `q1` (`700.00`),
    y excluye `q3` (`300.00`).
14. `?minPrice=700&maxPrice=700` incluye `q1`: los dos extremos son inclusivos.
15. `?categorySlug=laptops&brandSlug=acme&name=laptop&minPrice=1000` devuelve solo `q2`: los filtros
    se combinan con **AND**.
16. `?categorySlug=no-existe` responde `200` con `items` vacío y `totalElements: 0` — no es un error.
17. `?brandSlug=no-existe` responde igual.

**Casos borde**:
- `?minPrice=1000&maxPrice=500` → `422` con `code: INVALID_PRICE_RANGE`.
- `?minPrice=10.999` → `400` (escala máxima 2).
- `?minPrice=500` sin `maxPrice` → `200`: el rango abierto es válido.

**Notas de determinación (coste)**: el listado resuelve `brand`, `category` e `images` por elemento.
El trabajo no crece con el tamaño de la página: se compara `size=2` con `size=20` y se afirma por
forma, no contra un número absoluto.

### FL-PUB-010: ficha pública por slug

**Given**: el flujo crea `b1` (`"Acme"`), `c1` (`"Laptops"`) y tres productos con una imagen cada
uno: `q1` publicado (`name: "Laptop Pro 14"`, slug `laptop-pro-14`, `price: 1299.00`,
`description: "Portátil de 14 pulgadas."`), `q2` en `draft` (slug `teclado-k1`) y `q3`
`discontinued` (slug `monitor-viejo`). **Sin credencial**.

**When**: `getPublicProduct` — `GET /api/v1/products/laptop-pro-14`

**Then**:
1. Status `200` sin credencial.
2. El cuerpo es **P-PUB** y no trae ningún campo adicional: `id`, `sku`, `name: "Laptop Pro 14"`,
   `slug: "laptop-pro-14"`, `description: "Portátil de 14 pulgadas."`, `price: 1299.00`,
   `status: "active"`, `brand` (**P-BRAND** de `b1`), `category` (**P-CAT** de `c1`) e `images`.
3. El cuerpo **no** trae `slugFrozen`, `lockVersion`, `createdAt`, `updatedAt`, `createdBy` ni
   `updatedBy`.
4. `images` trae un elemento con la proyección **P-IMG** y `primary: true`, y su `image` es
   alcanzable sin credencial.
5. Un producto sin `description` devuelve el cuerpo **sin la clave** `description`, no con `null`.

**Casos borde**:
- `GET /api/v1/products/teclado-k1` (borrador) → `404` con `code: PRODUCT_NOT_FOUND`: la tienda no
  revela borradores, y responde como si no existiera.
- `GET /api/v1/products/monitor-viejo` (descontinuado) → `404` (`PRODUCT_NOT_FOUND`).
- `GET /api/v1/products/no-existe` → `404` (`PRODUCT_NOT_FOUND`). Los tres casos son
  indistinguibles para el consumidor, que es el objetivo.

### FL-PUB-020: navegación de la tienda — marcas y categorías

**Given**: el flujo crea 3 marcas (`"Acme"`, `"Globex"`, `"Initech"`) y 3 categorías
(`"Laptops"`, `"Monitores"`, `"Teclados"`). **Sin credencial**.

**When**: `listBrands` — `GET /api/v1/brands` y `listCategories` — `GET /api/v1/categories`

**Then**:
1. Las dos responden `200` **sin credencial**, con el sobre de paginación.
2. `totalElements: 3` en cada una, `page: 0`, `size: 20`.
3. Cada elemento es **P-BRAND** / **P-CAT**: `id`, `name`, `slug`, y `description` solo si la tiene.
4. El orden es `name` ascendente con el `id` como desempate: `"Acme"`, `"Globex"`, `"Initech"`.
5. `GET /api/v1/brands?page=1` responde `200` con `items` vacío.
6. `GET /api/v1/brands?size=500` responde con `size: 100`.
7. Ninguna de las dos respuestas trae campos de autoría.

---

## Caché

### FL-CCH-001: invalidación y retención de la ficha pública

**Given**: el flujo crea `b1` (`"Acme"`), `c1` (`"Laptops"`) y el producto `q1` publicado
(slug `laptop-pro-14`, `price: 1299.00`) con una imagen. La caché de `getPublicProduct` tiene un TTL
de 60 s y se invalida por `ProductUpdated`, `ProductStatusChanged`, `BrandUpdated` y
`CategoryUpdated`.

**When**: se lee `GET /api/v1/products/laptop-pro-14` (la caché se puebla), se muta por **cada** vía
declarada y se vuelve a leer **dentro del TTL** después de cada mutación.

**Then**:
1. Vía `ProductUpdated`: tras `updateProduct` con `price: 1199.00`, la relectura devuelve
   `price: 1199.00`.
2. Vía `ProductUpdated` (galería): tras `addProductImage`, la relectura trae la imagen nueva en
   `images`.
3. Vía `BrandUpdated`: tras `updateBrand` con `name: "Acme Corp"`, la relectura trae
   `brand.name: "Acme Corp"` y `brand.slug: "acme-corp"`.
4. Vía `CategoryUpdated`: tras `updateCategory` con `name: "Portátiles"`, la relectura trae
   `category.name: "Portátiles"`.
5. Vía `ProductStatusChanged`: tras `discontinueProduct`, la relectura responde `404`
   (`PRODUCT_NOT_FOUND`) — la ficha cacheada no sobrevive a la salida del catálogo.

**When**: **retención** — con `q1` publicado otra vez y su ficha recién leída (caché poblada), se
sustituye el objeto de su imagen en el bucket conservando la misma clave, que **no** es ninguna de
las vías de `invalidatedBy`, y se vuelve a leer dentro del TTL.

**Then**:
6. La relectura devuelve **el mismo cuerpo** que antes de la sustitución, con el mismo `image`: se
   sirve el valor cacheado.
7. Pasado el TTL, la lectura vuelve a reflejar el estado real.

**Notas de determinación**: la aserción 6 es la única que distingue una caché sana de un servicio que
no cachea nada — sin ella, todas las de invalidación pasarían igual sin caché alguna.

### FL-CCH-010: invalidación y retención de la ficha M2M

**Given**: el flujo crea `b1`, `c1` y el producto `q1` publicado con una imagen. Credencial de
máquina del cliente `orders-service`. La caché de `getProductForServices` tiene TTL de 60 s y las
mismas cuatro vías de invalidación.

**When**: se lee `GET /api/v1/services/products/{q1}` (la caché se puebla) y se muta por cada vía,
releyendo dentro del TTL.

**Then**:
1. Vía `ProductUpdated`: la relectura trae el `price` nuevo y un `updatedAt` posterior.
2. Vía `ProductUpdated` (galería): la relectura trae la imagen nueva.
3. Vía `BrandUpdated`: la relectura trae `brand.name` actualizado.
4. Vía `CategoryUpdated`: la relectura trae `category.name` actualizado.
5. Vía `ProductStatusChanged`: tras `discontinueProduct`, la relectura sigue respondiendo `200` con
   `status: "discontinued"` — a diferencia de la tienda, el M2M sí resuelve los descontinuados.

**When**: **retención** — se sustituye el objeto de la imagen en el bucket con la misma clave (vía no
declarada en `invalidatedBy`) y se relee dentro del TTL.

**Then**:
6. La relectura devuelve el mismo cuerpo cacheado.
7. Pasado el TTL, refleja el estado real.

---

## Superficie servidor-a-servidor

### FL-SRV-001: ficha de producto para otro servidor

**Given**: el flujo crea `b1` (`"Acme"`), `c1` (`"Laptops"`), el producto `q1` publicado
(`sku: "SKU-M01"`, `price: 1299.00`) con una imagen, `q2` en `draft` y `q3` `discontinued`.
Credencial de máquina del cliente `orders-service`, con el scope `product:read`.

**When**: `getProductForServices` — `GET /api/v1/services/products/{q1}`

**Then**:
1. Status `200`.
2. El cuerpo es **P-M2M** y no trae ningún campo adicional: `id`, `sku: "SKU-M01"`, `name`, `slug`,
   `description` (si la tiene), `price: 1299.00`, `status: "active"`, `createdAt`, `updatedAt`,
   `brand` (objeto **P-BRAND**), `category` (objeto **P-CAT**) e `images` (lista de **P-IMG**).
3. El cuerpo **no** trae `slugFrozen`, `lockVersion`, `createdBy` ni `updatedBy`: quién gestiona la
   tienda no sale de ella.
4. `updatedAt` viene: es de donde el consumidor sabe de cuándo es la ficha que recibe.
5. La misma llamada con la credencial de `cart-service` y con la de `search-service` devuelve
   **exactamente el mismo cuerpo**: la respuesta no depende de quién pregunte.

**When**: `GET /api/v1/services/products/{q3}` (descontinuado)

**Then**:
6. Status `200` con `status: "discontinued"`: un pedido histórico referencia artículos que ya no se
   venden.

**Casos borde**:
- `GET /api/v1/services/products/{q2}` (borrador) → `404` con `code: PRODUCT_NOT_FOUND`: un borrador
  no es todavía un artículo del catálogo.
- `productId` inexistente → `404` (`PRODUCT_NOT_FOUND`).
- **Sin credencial** → `401`.
- Con credencial de máquina **sin** el scope `product:read` → `403`.
- Con un token de máquina emitido para **otra audiencia** → `403`: el token es legítimo, pero no está
  emitido para este servicio.

### FL-SRV-010: resolución por lote

**Given**: el flujo crea `b1`, `c1` y cinco productos: `q1` (`sku: "SKU-B01"`), `q2`
(`sku: "SKU-B02"`) y `q3` (`sku: "SKU-B03"`) publicados, `q4` en `draft`, `q5` `discontinued`.
Credencial de máquina del cliente `cart-service`.

**When**: `listProductsBatchForServices` — `GET /api/v1/services/products` con los ids de
`q1`, `q2`, `q3`, `q5`, `q4`, un uuid inexistente y `q1` repetido.

**Then**:
1. Status `200` y el cuerpo es una **lista** (no un sobre paginado).
2. Trae **cuatro** elementos: `q1`, `q2`, `q3` y `q5`. La respuesta puede traer menos elementos que
   la petición y eso no es un error.
3. `q4` no está (es `draft`), el uuid inexistente no está, y `q1` aparece **una sola vez** aunque se
   pidiera dos.
4. Cada elemento tiene la proyección **P-M2M**.
5. El orden es `sku` ascendente con el `id` como desempate: `SKU-B01`, `SKU-B02`, `SKU-B03`, y el de
   `q5` en su sitio alfabético — **no** el orden de la petición.

**Orden de evaluación**:
1. La lista llega presente y con al menos un elemento → `EMPTY_ID_LIST` (`422`).
2. La lista no trae más de cien ids → `TOO_MANY_IDS` (`422`).

**Casos borde**:
- Sin el parámetro de ids → `422` con `code: EMPTY_ID_LIST`.
- Lista vacía → `422` (`EMPTY_ID_LIST`).
- Lista con 101 ids → `422` (`TOO_MANY_IDS`).
- Lista con exactamente 100 ids → `200`.
- Un id que no es un uuid → `400`.
- **Sin credencial** → `401`; con credencial de máquina sin `product:read` → `403`; con token de otra
  audiencia → `403`.

**Notas de determinación (coste)**: la operación resuelve `brand`, `category` e `images` de cada
elemento del lote. El trabajo **no crece con el tamaño de la lista**: se compara un lote de 2 ids con
uno de 100 y se afirma por forma, nunca contra un número absoluto de consultas. Es la afirmación que
distingue una resolución por lote de un bucle de búsquedas individuales, que devolvería exactamente
el mismo cuerpo.

---

## Fiabilidad de la publicación

### FL-OBX-001: el evento sobrevive a un canal indisponible

**Given**: el flujo crea `b1` y `c1`. El canal `productEvents` está **sin mensajes** e
**indisponible**. Credencial con rol `catalog-editor`.

**When**: `createProduct` — `POST /api/v1/management/products` con `Idempotency-Key: k-obx-1` y un
cuerpo válido (`sku: "SKU-OBX"`).

**Then**:
1. Status `201` con el cuerpo completo **P-MGMT**: la indisponibilidad del canal **no llega al
   cliente**.
2. El producto es legible con `getProduct`, con `status: "draft"`.
3. El canal `productEvents` **no ha recibido ningún mensaje** todavía.

**When**: el canal `productEvents` vuelve a estar disponible

**Then**:
4. En <= 10 s el canal recibe **exactamente un** `ProductCreated`.
5. Su payload es el del producto creado (`productId` el devuelto, `sku: "SKU-OBX"`, `brandName`,
   `categoryName`), y su `correlationId` es el de la petición del paso anterior.
6. El servidor no se ha rendido con ningún evento: **ningún evento abandonado**.

**Notas de determinación**: la aserción 3 es la que separa el outbox de publicar directamente dentro
de la operación —un servicio que publica en línea también acabaría entregando el mensaje— y el
«exactamente un» de la 4 es la que separa un relay que marca lo entregado de uno que reentrega para
siempre. El escenario habla del **canal lógico**, nunca del broker ni de cómo se detiene.

### FL-OBX-002: el evento que el relay abandona no se pierde en silencio

**Given**: el flujo crea `b1` y `c1`. El canal `productEvents` está indisponible y ya se ejecutó un
`createProduct` (`sku: "SKU-OBX2"`), con su evento pendiente de salir.

**When**: se agota el presupuesto de reintentos de ese evento

**Then**:
1. El servidor lo dice: informa de **un** evento abandonado.
2. Restablecido el canal, ese evento **no** se publica — el relay respeta que se rindió.
3. Y el canal no recibe ninguna otra cosa.

**Notas de determinación**: la aserción 2 es la que lo hace algo más que una prueba de la señal: sin
ella, un relay que ignorase su propio presupuesto reintentaría para siempre una fila ya dada por
perdida y el escenario pasaría igual. El escenario habla del **presupuesto de reintentos**, nunca de
su número ni del nombre de la métrica.

---

## Autorización y consumo desde el navegador

### FL-SEC-001: quién puede llamar a qué

**Given**: el flujo crea `b1`, `c1` y un producto `p1` en `draft` con una imagen. Se dispone de tres
identidades: sin credencial, credencial de usuario con rol `catalog-editor`, y credencial de usuario
con rol `catalog-admin`.

**When**: cada operación protegida se llama **sin credencial**

**Then**:
1. Las 21 operaciones de `/api/v1/management/**` responden `401`.
2. Las 2 operaciones de `/api/v1/services/**` responden `401`.
3. Ninguna produce efecto: `listProducts` (con credencial) devuelve el mismo `totalElements` que
   antes del `When`, y en `productEvents` no ha aparecido ningún mensaje.

**When**: las operaciones reservadas al admin se llaman con credencial de `catalog-editor`

**Then**:
4. `deleteProduct` → `403` (exige `product:delete`).
5. `createBrand`, `updateBrand`, `deleteBrand` → `403` (exigen `taxonomy:write`).
6. `createCategory`, `updateCategory`, `deleteCategory` → `403`.
7. Ninguna produce efecto observable: la marca no se crea, el producto no se borra, y no se publica
   ningún evento.

**When**: las operaciones de producto y galería se llaman con credencial de `catalog-editor`

**Then**:
8. `createProduct`, `updateProduct`, `publishProduct`, `unpublishProduct`, `discontinueProduct`,
   `reactivateProduct`, `addProductImage`, `removeProductImage`, `setPrimaryProductImage`,
   `reorderProductImages`, `getProduct` y `listProducts` responden su status de éxito: el editor
   alcanza `product:read` y `product:write`.
9. Las mismas llamadas con credencial de `catalog-admin` responden igual: el admin alcanza todo lo
   del editor.

**When**: las cuatro operaciones públicas se llaman **sin credencial**

**Then**:
10. `listPublicProducts`, `getPublicProduct`, `listBrands` y `listCategories` responden `200`: son el
    escaparate.
11. Una llamada a `/api/v1/services/products/{p1}` con credencial de **usuario** (no de máquina) sin
    el scope `product:read` responde `403`.

### FL-SEC-010: preflight del navegador

**Given**: la política `cors` del servicio declara las cabeceras `Authorization`, `Content-Type` e
`Idempotency-Key`, expone `X-Correlation-Id`, no admite credenciales y cachea el preflight 3600 s.

**When**: una petición `OPTIONS` **sin credencial** desde un origen web sobre
`/api/v1/management/products`, anunciando el método `POST` y las cabeceras `Authorization`,
`Content-Type` e `Idempotency-Key`.

**Then**:
1. La respuesta es de éxito: el preflight **no** muere en la cadena de seguridad, aunque la ruta esté
   protegida y la petición no lleve credencial.
2. El método `POST` figura entre los admitidos.
3. Las tres cabeceras anunciadas figuran entre las admitidas — `Idempotency-Key` incluida, sin ella
   el back-office no podría cumplir la exigencia de la clave.
4. El tiempo de cacheo del preflight es 3600 s.
5. La respuesta **no** admite credenciales (el token viaja en `Authorization`, no en cookie).

### FL-SEC-020: petición normal cross-origin

**Given**: la misma política, y un producto `q1` publicado.

**When**: `getPublicProduct` — `GET /api/v1/products/{slug de q1}` desde un origen web, con la
cabecera de origen.

**Then**:
1. Status `200` con el cuerpo **P-PUB**.
2. El origen figura como permitido en la respuesta.
3. `X-Correlation-Id` figura entre las cabeceras **expuestas**: el navegador puede leerla.
4. Ninguna otra cabecera propia se expone.

**Notas de determinación**: este escenario y el anterior prueban fallos distintos — un servicio que
contesta al preflight y no expone sus cabeceras pasa FL-SEC-010 y rompe al cliente igual. Los
escenarios hablan de «un origen web» y de la **política**, nunca de los orígenes concretos, que son
despliegue y no están en el diseño.
