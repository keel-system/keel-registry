# catalog — Escenarios de validación

> Escenarios de aceptación ejecutables (Given/When/Then) derivados de
> specs/catalog v0.3.0. Contrato de validación para la fase de generación.

## Convenciones de determinación

Valen para **todo** el servicio y ningún escenario las repite.

- **Base de rutas**: todas las rutas van bajo `/api/v1`. Los prefijos separan superficies:
  `/products`, `/brands`, `/categories` (tienda, sin credencial), `/management/...` (back-office,
  token de usuario) y `/services/...` (otros servidores, credencial de máquina).
- **Fecha y hora**: instantes en UTC, ISO-8601 con milisegundos (`2026-08-02T09:21:07.482Z`).
  `createdAt` y `updatedAt` los estampa el servidor y **se verifican por forma o por rango**, nunca
  por valor literal. El negocio no tiene ninguna noción de "hoy" local: no hay fechas locales.
- **Identificadores generados**: todos los `id` son uuid. Se verifican por forma y por reutilización
  simbólica — el `id` devuelto en un escenario es el que usan los siguientes de su flujo. Los
  identificadores simbólicos de este documento (`b1`, `c1`, `p1`, `img1`) nombran la entidad, no un
  literal que viaje por el cable.
- **Números y dinero**: `price` es decimal de **escala 2**, siempre serializado con dos decimales
  (`12.50`, no `12.5`). No hay cálculo aritmético en el servicio, así que no hay regla de redondeo
  que fijar: el precio se almacena y se devuelve tal cual llegó, ya normalizado a escala 2. Un
  `price` con más de dos decimales en la petición es `400`.
- **Ausencia vs nulo**: un campo opcional sin valor **aparece en la respuesta con valor `null`**;
  no se omite. Vale para `description` de producto, marca y categoría. Las colecciones vacías
  aparecen como `[]`, nunca como `null` ni ausentes.
- **Mayúsculas y acentos**: la unicidad de `Brand.name` y `Category.name` y las búsquedas parciales
  por `name` y `sku` **no distinguen** mayúsculas ni acentos: `ACME`, `Acme` y `acmé` son el mismo
  nombre y colisionan. El `sku` se normaliza a mayúsculas antes de comparar, así que su unicidad es
  exacta sobre el valor ya normalizado.
- **Slug**: se deriva del `name` normalizado a kebab-case (minúsculas, acentos plegados, todo lo que
  no sea `[a-z0-9]` colapsado a un solo guión, sin guiones al principio ni al final).
- **Forma del cuerpo de error**: el servicio tiene **una** forma de error, la que impone el
  generador — `{timestamp, status, error, code, message, details}` más `correlationId`. Los
  escenarios solo fijan el `code` y el status HTTP; el resto de campos se verifica por presencia.
  El `message` es texto humano y **no** es contrato: ningún `Then` lo compara.
- **`BRAND_NOT_FOUND` y `CATEGORY_NOT_FOUND` llevan status distinto según la operación**, y es
  deliberado: `422` cuando son una referencia inválida dentro del alta o la edición de un producto
  (el recurso de la petición sí existe), y `404` cuando son el recurso de la propia petición
  (`updateBrand`, `updateCategory`). Cada escenario lo dice.
- **Los tres `DELETE` del servicio son idempotentes** y por eso ninguno declara un error de «no
  encontrado»: `deleteBrand`, `deleteCategory` y `removeProductImage` responden `204` sin cuerpo
  tanto si eliminaron algo como si el recurso ya no estaba. El evento de borrado se publica **solo**
  cuando hubo eliminación real, así que un `204` no implica evento. Las guardas que sí quedan
  (`BRAND_IN_USE`, `CATEGORY_IN_USE`, `PRODUCT_DISCONTINUED`, `LAST_IMAGE_OF_ACTIVE_PRODUCT`) se
  siguen evaluando y siguen fallando; la idempotencia cubre la ausencia del recurso, no las
  invariantes. `removeProductImage` es idempotente respecto a la **imagen**, no al producto de la
  ruta: un `productId` inexistente sigue siendo `404`.
- **Sobre de paginación**: `{ items, page, size, totalElements, totalPages }`, con `page` en base 0.
  Se pide con los query params `page` y `size`. `size` por defecto 20, tope 100; un `size` mayor
  **se recorta** a 100, no da error. Es contrato canónico del DSL, igual en todo generador.
- **Cabecera `Location`**: la emite toda creación `201` cuyo `output` traiga `id`
  (`createProduct`, `createBrand`, `createCategory`, `addProductImage`), con la URI de la petición
  más el id devuelto.
- **Idempotencia**: la clave viaja en la cabecera `Idempotency-Key`. La aplican `createProduct` y
  `addProductImage`, con ventana de 24 h. Repetir la clave **con el mismo contenido** devuelve el
  mismo status y el mismo cuerpo sin segundo efecto; repetirla **con contenido distinto** devuelve
  `IDEMPOTENCY_KEY_REUSED` (`409`).
- **Concurrencia**: `persistence.consistency.optimisticLocking: declared` y solo `Product` declara
  `lockVersion`. Dos ediciones concurrentes del mismo producto dan conflicto `409`
  (`PRODUCT_VERSION_CONFLICT`); dos ediciones concurrentes de la misma marca o categoría **no** dan
  conflicto y gana la última en confirmar, comprobado leyendo el estado final por la API.
- **Autorización**: los endpoints de `/management/...` exigen token de usuario con el permiso de la
  operación; los de `/services/...`, credencial de máquina del `serviceClient` con el scope
  `product:read`. Los escenarios hablan de "credencial de máquina del cliente `order-service`",
  nunca del proveedor de identidad.
- **Eventos**: todo evento viaja con la envoltura estándar (`metadata` + `data`). Los `Then` fijan
  el nombre del evento, su canal lógico (`productEvents` o `taxonomyEvents`) y los campos de `data`;
  `metadata` se verifica por presencia (`eventId`, `eventType`, `occurredAt`, `source`).
- **Archivos**: `ProductImage.file` se proyecta como la referencia pública al objeto del bucket
  `productImages`, que es de visibilidad `public`. Se verifica **por forma** (referencia resoluble)
  y porque una lectura directa de esa referencia devuelve el binario subido.

### Proyección de las respuestas

Derivada de los artefactos; **ninguna enumeración de este documento se aparta de esto**.

| Superficie | Campos del producto |
|---|---|
| Back-office (`getProduct`, `listProducts`, `createProduct`, `updateProduct`, transiciones, imágenes) | `id`, `sku`, `name`, `slug`, `description`, `price`, `status`, `lockVersion`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy`, `brand{}`, `category{}`, `images[]` |
| Tienda (`getPublicProduct`, `listPublicProducts`) | los mismos **menos** `lockVersion`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy` |
| M2M (`getProductForServices`, `listProductsBatchForServices`) | los mismos **menos** `lockVersion`, `createdAt`, `createdBy`, `updatedBy` — **conserva `updatedAt`** |

`brand{}` y `category{}` son objetos anidados (`embed`) con `id`, `name`, `slug`, `description`, y
**nada más**: la taxonomía no lleva campos de auditoría en ninguna superficie. `images[]` es una
lista de `{ id, file, altText, position, isPrimary }` ordenada por `position` ascendente.
`status` tiene `default: draft`, así que **viaja siempre**, incluso en la respuesta de un alta que
no lo mandó.

## Matriz de cobertura

| Operación | Flujos | Superficie |
|---|---|---|
| `createBrand` | FL-BRD-001 | usuarios |
| `updateBrand` | FL-BRD-001 | usuarios |
| `deleteBrand` | FL-BRD-010 | usuarios |
| `createCategory` | FL-CTG-001 | usuarios |
| `updateCategory` | FL-CTG-001 | usuarios |
| `deleteCategory` | FL-CTG-010 | usuarios |
| `createProduct` | FL-PRD-001 | usuarios |
| `updateProduct` | FL-PRD-010 | usuarios |
| `publishProduct` | FL-PRD-020 | usuarios |
| `unpublishProduct` | FL-PRD-020 | usuarios |
| `discontinueProduct` | FL-PRD-020 | usuarios |
| `reactivateProduct` | FL-PRD-020 | usuarios |
| `getProduct` | FL-PRD-030 | usuarios |
| `listProducts` | FL-PRD-030 | usuarios |
| `addProductImage` | FL-IMG-001 | usuarios |
| `setPrimaryProductImage` | FL-IMG-001 | usuarios |
| `reorderProductImages` | FL-IMG-001 | usuarios |
| `removeProductImage` | FL-IMG-001 | usuarios |
| `getPublicProduct` | FL-PUB-001, FL-CCH-001 | pública (sin credencial) |
| `listPublicProducts` | FL-PUB-010, FL-CCH-001, FL-CCH-020 | pública (sin credencial) |
| `listBrands` | FL-PUB-020 | pública (sin credencial) |
| `listCategories` | FL-PUB-020 | pública (sin credencial) |
| `getProductForServices` | FL-M2M-001 | **servidores (M2M)** |
| `listProductsBatchForServices` | FL-M2M-010 | **servidores (M2M)** |
| *(todas las protegidas)* | FL-SEC-001 | usuarios + **servidores** |

---

## Marcas

### FL-BRD-001: alta, consulta y edición de una marca

**Given**: no existe ninguna marca.

**When**: `createBrand` — `POST /api/v1/management/brands`, token de usuario con `taxonomy:write`
```json
{ "name": "Nike", "description": "Ropa y calzado deportivo." }
```

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta de la marca creada.
3. El cuerpo trae `id` (forma de uuid), `name: "Nike"`, `slug: "nike"` y
   `description: "Ropa y calzado deportivo."`, y ningún campo adicional — en particular **no** trae
   `createdAt`, `updatedAt`, `createdBy`, `updatedBy` ni `lockVersion`.
4. Se publica `BrandCreated` en el canal `taxonomyEvents` con `brandId` (el devuelto),
   `name: "Nike"`, `slug: "nike"` y `updatedAt` (forma de instante).
5. `listBrands` (`GET /api/v1/brands`, sin credencial) responde `200` con `totalElements: 1` y
   `items[0]` igual al cuerpo del punto 3.

**When**: `createBrand` con `{ "name": "nike", "description": null }` (mismo nombre en minúsculas)

**Then**:
6. Status `409`, `code: BRAND_NAME_ALREADY_EXISTS`: la unicidad no distingue mayúsculas.
7. `listBrands` sigue devolviendo `totalElements: 1`.

**When**: `updateBrand` — `PUT /api/v1/management/brands/{brandId}` sobre la marca creada
```json
{ "name": "Nike Sportswear", "description": null }
```

**Then**:
8. Status `200`.
9. El cuerpo trae el mismo `id`, `name: "Nike Sportswear"`, `slug: "nike-sportswear"` (recalculado)
   y `description: null` — presente y nulo, no ausente: la edición es un **reemplazo completo**.
10. Se publica `BrandUpdated` en `taxonomyEvents` con el `brandId`, `name: "Nike Sportswear"`,
    `slug: "nike-sportswear"` y `updatedAt`.

**Orden de evaluación** (`updateBrand`):
1. La marca existe → `BRAND_NOT_FOUND` (`404`).
2. Ninguna otra marca tiene ese `name` ignorando mayúsculas y acentos → `BRAND_NAME_ALREADY_EXISTS` (`409`).
3. El slug derivado no coincide con el de otra marca → `BRAND_SLUG_CONFLICT` (`409`).

**Casos borde**:
- `createBrand` con `name` ausente → `400`.
- `createBrand` con `name` de 81 caracteres → `400` (`maxLength: 80`).
- Existiendo `"Nike Sportswear"`, `createBrand` con `name: "Nike, Sportswear!"` → `409`,
  `code: BRAND_SLUG_CONFLICT`: los dos nombres son distintos (no colisiona la guarda 2) pero
  derivan el mismo slug `nike-sportswear`.
- `updateBrand` sobre un `brandId` inexistente **y** con un nombre ya usado por otra marca →
  `404`, `code: BRAND_NOT_FOUND`: la guarda 1 precede a la 2.
- Dos `updateBrand` concurrentes sobre la misma marca con nombres `A` y `B`: **ambas responden
  `200`** (la taxonomía no está sujeta a bloqueo optimista) y `listBrands` devuelve un `name` final
  que es `A` o `B`, con el `slug` coherente con ese ganador.

**Notas de determinación**: el `slug` se compara por valor exacto porque es contrato de URL.

### FL-BRD-010: borrado de una marca, en uso y libre

**Given**: existen la marca `b1` (`name: "Nike"`) y la categoría `c1` (`name: "Calzado"`), y el
producto `p1` (`sku: "SKU-001"`, marca `b1`, categoría `c1`), creado dentro de este flujo.

**When**: `deleteBrand` — `DELETE /api/v1/management/brands/{b1}`, token con `taxonomy:write`

**Then**:
1. Status `409`, `code: BRAND_IN_USE`.
2. No se publica ningún `BrandDeleted`.
3. `listBrands` sigue devolviendo la marca `b1`.

**When**: se crea la marca `b2` (`name: "Adidas"`), que ningún producto referencia, y se ejecuta
`deleteBrand` sobre `b2`

**Then**:
4. Status `204`, sin cuerpo.
5. Se publica `BrandDeleted` en `taxonomyEvents` con `brandId` (el de `b2`), `slug: "adidas"` y
   `deletedAt` (forma de instante).
6. `listBrands` devuelve `totalElements: 1` y ya no incluye `b2`.
7. `deleteBrand` sobre `b2` otra vez → `204`, sin cuerpo: el borrado es idempotente.
8. Esa repetición **no** publica un segundo `BrandDeleted`: el evento solo sale cuando algo se
   eliminó de verdad. `listBrands` sigue devolviendo `totalElements: 1`.

**Orden de evaluación** (`deleteBrand`):
1. Ningún producto la referencia → `BRAND_IN_USE` (`409`).
2. Si no existe una marca con ese id, no hay guarda que falle: `204` sin efecto.

**Casos borde**:
- `deleteBrand` sobre un id inexistente → `204`, sin cuerpo y sin evento, aunque otra marca sí
  estuviera en uso: la guarda de integridad solo mira la marca indicada.
- La referencia producto→marca es una **restricción de integridad**: si `createProduct` con la marca
  `b2` confirma a la vez que su borrado, una de las dos escrituras falla; el borrado que pierde
  responde `BRAND_IN_USE` (`409`) y el producto no queda nunca con una marca inexistente.
  Comprobado leyendo `listBrands` y `getProduct` después de la carrera.

---

## Categorías

### FL-CTG-001: alta, consulta y edición de una categoría

Simétrico a FL-BRD-001, con `createCategory` / `updateCategory`,
`POST|PUT /api/v1/management/categories[/{categoryId}]`, `listCategories`
(`GET /api/v1/categories`) y los eventos `CategoryCreated` / `CategoryUpdated` en `taxonomyEvents`
con `categoryId`, `name`, `slug` y `updatedAt`.

**Given**: no existe ninguna categoría.

**When**: `createCategory` con `{ "name": "Calzado", "description": "Zapatillas y botas." }`

**Then**:
1. Status `201`, cabecera `Location`.
2. El cuerpo trae `id`, `name: "Calzado"`, `slug: "calzado"`,
   `description: "Zapatillas y botas."` y ningún campo adicional.
3. Se publica `CategoryCreated` en `taxonomyEvents`.
4. `listCategories` responde `200` con `totalElements: 1`.

**When**: `createCategory` con `{ "name": "calzado" }` → **Then** `409`,
`code: CATEGORY_NAME_ALREADY_EXISTS`.

**When**: `updateCategory` con `{ "name": "Calzado deportivo", "description": null }`

**Then**:
5. Status `200`, `slug: "calzado-deportivo"`, `description: null`.
6. Se publica `CategoryUpdated` en `taxonomyEvents`.

**Orden de evaluación** (`updateCategory`):
1. La categoría existe → `CATEGORY_NOT_FOUND` (`404`).
2. Ninguna otra tiene ese `name` ignorando mayúsculas y acentos → `CATEGORY_NAME_ALREADY_EXISTS` (`409`).
3. El slug derivado no coincide con el de otra → `CATEGORY_SLUG_CONFLICT` (`409`).

**Casos borde**:
- `createCategory` con `name: "Calzado, deportivo"` existiendo `"Calzado deportivo"` → `409`,
  `code: CATEGORY_SLUG_CONFLICT`.
- `updateCategory` sobre id inexistente y nombre ya usado → `404`, `code: CATEGORY_NOT_FOUND`.

### FL-CTG-010: borrado de una categoría, en uso y libre

Simétrico a FL-BRD-010, con `deleteCategory` — `DELETE /api/v1/management/categories/{categoryId}`,
el código `CATEGORY_IN_USE` (`409`) y el evento `CategoryDeleted` en `taxonomyEvents` con
`categoryId`, `slug` y `deletedAt`. Mismo orden de evaluación, misma **idempotencia** (repetir el
borrado sobre una categoría ya eliminada, o sobre un id inexistente, responde `204` sin cuerpo y sin
publicar un segundo `CategoryDeleted`) y misma garantía de integridad referencial frente a la carrera
con `createProduct`.

---

## Productos: alta y ficha

### FL-PRD-001: alta de producto, evento e idempotencia

**Given**: existen la marca `b1` (`name: "Nike"`, `slug: "nike"`) y la categoría `c1`
(`name: "Calzado"`, `slug: "calzado"`), creadas dentro de este flujo. No existe ningún producto.

**When**: `createProduct` — `POST /api/v1/management/products`, token con `product:write`,
cabecera `Idempotency-Key: key-001`
```json
{
  "sku": "sku-001",
  "name": "Zapatilla Runner",
  "description": "Zapatilla de running para asfalto.",
  "price": 89.90,
  "brandId": "<b1>",
  "categoryId": "<c1>"
}
```

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta del producto creado.
3. El cuerpo trae exactamente: `id` (uuid), `sku: "SKU-001"` (**normalizado a mayúsculas**),
   `name: "Zapatilla Runner"`, `slug: "zapatilla-runner"`,
   `description: "Zapatilla de running para asfalto."`, `price: 89.90` (escala 2),
   `status: "draft"` (viaja aunque la petición no lo mandara), `lockVersion` (entero),
   `createdAt`, `updatedAt`, `createdBy`, `updatedBy`,
   `brand: { id: <b1>, name: "Nike", slug: "nike", description: … }`,
   `category: { id: <c1>, name: "Calzado", slug: "calzado", description: … }`
   e `images: []`.
4. `createdBy` y `updatedBy` traen la identidad del token que hizo la llamada, no `null`.
5. `brand` y `category` son **objetos anidados**, no `brandId` / `categoryId`, y ninguno de los dos
   trae campos de auditoría.
6. Se publica `ProductCreated` en el canal `productEvents` con `productId` (el devuelto),
   `sku: "SKU-001"`, `name`, `slug`, `description`, `price: 89.90`, `status: "draft"`,
   `brandId: <b1>`, `brandName: "Nike"`, `categoryId: <c1>`, `categoryName: "Calzado"`,
   `primaryImage` ausente (el producto aún no tiene imágenes) y `updatedAt`.
7. `getProduct` sobre el `id` devuelto responde `200` con el mismo cuerpo del punto 3.

**When**: se repite la **misma** petición con `Idempotency-Key: key-001`

**Then**:
8. Status `201` y **el mismo cuerpo** del punto 3, con el mismo `id`.
9. `listProducts` devuelve `totalElements: 1`: no hubo segundo alta.
10. No se publica un segundo `ProductCreated`.

**When**: se repite con `Idempotency-Key: key-001` pero `name: "Otra cosa"`

**Then**:
11. Status `409`, `code: IDEMPOTENCY_KEY_REUSED`.
12. `listProducts` sigue devolviendo `totalElements: 1`.

**Orden de evaluación** (`createProduct`):
1. Ningún producto tiene ese `sku` (normalizado) → `SKU_ALREADY_EXISTS` (`409`).
2. La marca existe → `BRAND_NOT_FOUND` (`422`).
3. La categoría existe → `CATEGORY_NOT_FOUND` (`422`).
4. La clave de idempotencia no se usó con otro contenido → `IDEMPOTENCY_KEY_REUSED` (`409`).
5. El slug derivado (con su sufijo) sigue libre en el instante de escribir → `PRODUCT_SLUG_CONFLICT`
   (`409`). Es la única guarda que se dispara por una carrera, no por el estado leído al validar las
   guardas 1-4.

**Casos borde**:
- `sku: "SKU-001"` de nuevo, sin cabecera de idempotencia → `409`, `code: SKU_ALREADY_EXISTS`.
- `sku: "ab"` → `400` (el patrón exige 3 caracteres como mínimo).
- `price: 89.999` → `400` (escala 2).
- `price: -1` → `400` (`min: 0`).
- `name` ausente → `400`.
- `brandId` de una marca inexistente → `422`, `code: BRAND_NOT_FOUND` (nótese: **422**, no 404,
  porque el recurso de la petición no es la marca).
- `brandId` inexistente **y** `sku` ya usado → `409`, `code: SKU_ALREADY_EXISTS`: la guarda 1
  precede a la 2.
- `brandId` inexistente **y** `categoryId` inexistente → `422`, `code: BRAND_NOT_FOUND`: la guarda 2
  precede a la 3.
- Dos altas con el mismo `sku` en paralelo: una responde `201` y la otra `409`
  (`SKU_ALREADY_EXISTS`); `listProducts` devuelve `totalElements: 1`.
- Dos altas en paralelo con el mismo `name` (y por tanto el mismo slug candidato): una responde
  `201` y la otra `409`, `code: PRODUCT_SLUG_CONFLICT` — la que pierde la carrera **no** reintenta
  en silencio con el siguiente sufijo; `listProducts` devuelve `totalElements: 1` y el slug del
  producto creado es el que ninguna de las dos había leído como ocupado. No es reproducible con dos
  peticiones secuenciales: en secuencial, la segunda siempre ve el slug de la primera ya escrito y
  deriva el sufijo libre sin error (validado por el `slug: "..."` recalculado en escenarios como
  FL-BRD-001).

**Notas de determinación**: `id`, `lockVersion`, `createdAt` y `updatedAt` se verifican por forma;
`sku` se compara por valor **ya normalizado a mayúsculas**.

### FL-PRD-010: edición de la ficha y conflicto de versión

**Given**: existen `b1`, `c1`, una segunda marca `b2` (`name: "Adidas"`) y el producto `p1`
(`sku: "SKU-001"`, `name: "Zapatilla Runner"`, `price: 89.90`, `status: "draft"`, marca `b1`),
creados dentro de este flujo. Se conoce su `lockVersion` actual, leído con `getProduct`.

**When**: `updateProduct` — `PUT /api/v1/management/products/{p1}`, token con `product:write`
```json
{
  "name": "Zapatilla Runner Pro",
  "description": null,
  "price": 99.00,
  "brandId": "<b2>",
  "categoryId": "<c1>",
  "lockVersion": <la leída>
}
```

**Then**:
1. Status `200`.
2. El cuerpo trae `sku: "SKU-001"` **sin cambios** (el sku es inmutable y no está en la entrada),
   `name: "Zapatilla Runner Pro"`, `slug: "zapatilla-runner-pro"` (recalculado: el producto sigue
   en `draft` y nunca se publicó), `description: null`, `price: 99.00`, `status: "draft"`,
   `brand.id: <b2>` y `brand.name: "Adidas"`.
3. `lockVersion` es **distinto** del enviado.
4. `updatedAt` es posterior a `createdAt`; `createdAt` y `createdBy` **no** cambiaron.
5. Se publica `ProductUpdated` en `productEvents` con la ficha completa ya actualizada
   (`price: 99.00`, `brandName: "Adidas"`).

**When**: se repite la misma petición con el `lockVersion` **antiguo**

**Then**:
6. Status `409`, `code: PRODUCT_VERSION_CONFLICT`.
7. `getProduct` devuelve la ficha del punto 2 sin cambios.
8. No se publica un segundo `ProductUpdated`.

**Orden de evaluación** (`updateProduct`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está descatalogado → `PRODUCT_DISCONTINUED` (`409`).
3. La marca existe → `BRAND_NOT_FOUND` (`422`).
4. La categoría existe → `CATEGORY_NOT_FOUND` (`422`).
5. Si el producto está `active`, el nuevo `price` es mayor que cero → `PRODUCT_NOT_PUBLISHABLE` (`422`).
6. El `lockVersion` coincide con el vigente → `PRODUCT_VERSION_CONFLICT` (`409`).
7. Si el `name` cambió y el producto sigue en `draft` sin publicar nunca (el slug se recalcula): el
   slug derivado sigue libre en el instante de escribir → `PRODUCT_SLUG_CONFLICT` (`409`). No aplica
   si el slug está congelado (producto ya publicado alguna vez): entonces no hay recálculo y esta
   guarda no se evalúa.

**Ramas condicionales**:
- El `slug` **solo** se recalcula si el producto sigue en `draft` y nunca se publicó. Un producto
  que ya pasó por `active` conserva su slug aunque cambie el `name` (validado en FL-PRD-020).
- `description` omitida o `null` deja el campo a `null`: la edición es reemplazo completo, nunca
  fusión.

**Casos borde**:
- `updateProduct` sobre un id inexistente → `404`, `code: PRODUCT_NOT_FOUND`.
- Id inexistente **y** `lockVersion` viejo → `404`, `code: PRODUCT_NOT_FOUND`: la guarda 1 precede
  a la 6.
- `categoryId` inexistente **y** `lockVersion` viejo → `422`, `code: CATEGORY_NOT_FOUND`: la guarda
  4 precede a la 6.
- `lockVersion` ausente → `400`.
- Dos ediciones en paralelo sobre dos productos `draft` distintos, ambos aún sin publicar, que
  cambian su `name` al mismo valor: una responde `200` con el slug recalculado y la otra `409`,
  `code: PRODUCT_SLUG_CONFLICT` — misma carrera que en el alta (FL-PRD-001), aplicada al recálculo.
  Un producto ya publicado alguna vez nunca entra en esta carrera: su slug está congelado.

---

## Productos: ciclo de vida

### FL-PRD-020: publicar, despublicar, descatalogar y reactivar

**Given**: existen `b1`, `c1` y el producto `p1` (`sku: "SKU-001"`, `name: "Zapatilla Runner"`,
`price: 89.90`, `status: "draft"`), **sin ninguna imagen**, creados dentro de este flujo.

**When**: `publishProduct` — `POST /api/v1/management/products/{p1}/publish`, token con
`product:publish`

**Then**:
1. Status `422`, `code: PRODUCT_NOT_PUBLISHABLE`: un producto `active` exige al menos una imagen.
2. `getProduct` devuelve `status: "draft"`.
3. No se publica ningún `ProductStatusChanged`.

**When**: se añade una imagen a `p1` con `addProductImage` y se repite `publishProduct`

**Then**:
4. Status `200`, cuerpo con `status: "active"` y el resto de la ficha sin cambios.
5. Se publica `ProductStatusChanged` en `productEvents` con `productId: <p1>`, `sku: "SKU-001"`,
   `slug: "zapatilla-runner"`, `previousStatus: "draft"`, `newStatus: "active"` y `updatedAt`.
6. `getPublicProduct` por `slug: "zapatilla-runner"` responde `200`: el producto ya es visible
   para la tienda.

**When**: `updateProduct` sobre `p1` con `name: "Zapatilla Runner Elite"`

**Then**:
7. Status `200` con `name: "Zapatilla Runner Elite"` y `slug: "zapatilla-runner"` **sin cambiar**:
   el slug queda congelado en cuanto el producto se publica por primera vez.
8. `getPublicProduct` por `slug: "zapatilla-runner-elite"` responde `404`,
   `code: PRODUCT_NOT_FOUND`; por `"zapatilla-runner"` sigue respondiendo `200`.

**When**: `publishProduct` sobre `p1`, que ya está `active`

**Then**:
9. Status `409`, `code: INVALID_STATUS_TRANSITION`.

**When**: `unpublishProduct` — `POST /api/v1/management/products/{p1}/unpublish`

**Then**:
10. Status `200`, `status: "draft"`.
11. Se publica `ProductStatusChanged` con `previousStatus: "active"`, `newStatus: "draft"`.
12. `getPublicProduct` por `"zapatilla-runner"` responde `404`, `code: PRODUCT_NOT_FOUND`: solo los
    `active` son visibles para la tienda.

**When**: `discontinueProduct` — `POST /api/v1/management/products/{p1}/discontinue` sobre un
producto en `draft`

**Then**:
13. Status `409`, `code: INVALID_STATUS_TRANSITION`: solo se descataloga desde `active`.

**When**: se vuelve a publicar `p1` y se ejecuta `discontinueProduct`

**Then**:
14. Status `200`, `status: "discontinued"`.
15. Se publica `ProductStatusChanged` con `previousStatus: "active"`, `newStatus: "discontinued"`.
16. `getPublicProduct` por su slug responde `404`.
17. `getProductForServices` sobre `{p1}` con credencial de máquina responde `200` con
    `status: "discontinued"`: el M2M sí ve los descatalogados.

**When**: `updateProduct` sobre `p1`, descatalogado, con `lockVersion` vigente

**Then**:
18. Status `409`, `code: PRODUCT_DISCONTINUED`.

**When**: `addProductImage` sobre `p1`, descatalogado

**Then**:
19. Status `409`, `code: PRODUCT_DISCONTINUED`.

**When**: `reactivateProduct` — `POST /api/v1/management/products/{p1}/reactivate`

**Then**:
20. Status `200`, `status: "active"`.
21. Se publica `ProductStatusChanged` con `previousStatus: "discontinued"`,
    `newStatus: "active"`.
22. `getPublicProduct` por `"zapatilla-runner"` vuelve a responder `200`.

**Orden de evaluación** (las cuatro transiciones):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El estado de origen admite la transición → `INVALID_STATUS_TRANSITION` (`409`).
3. Solo en `publishProduct` y `reactivateProduct`: el producto tiene al menos una imagen y
   `price > 0` → `PRODUCT_NOT_PUBLISHABLE` (`422`).

**Casos borde**:
- `unpublishProduct` sobre un producto `discontinued` → `409`, `code: INVALID_STATUS_TRANSITION`.
- `reactivateProduct` sobre un producto `draft` → `409`, `code: INVALID_STATUS_TRANSITION`.
- `publishProduct` sobre un id inexistente **y** sin imágenes → `404`,
  `code: PRODUCT_NOT_FOUND`: la guarda 1 precede a la 3.
- Los tres estados del ciclo de vida (`draft`, `active`, `discontinued`) quedan alcanzados por este
  flujo, y las cuatro transiciones declaradas quedan ejecutadas.

---

## Productos: consulta de back-office

### FL-PRD-030: consulta y listado con filtros, orden y paginación

**Given**: existen `b1` (`"Nike"`, slug `nike`), `b2` (`"Adidas"`, slug `adidas`), `c1`
(`"Calzado"`, slug `calzado`), `c2` (`"Camisetas"`, slug `camisetas`) y 25 productos creados dentro
del flujo: `p1`…`p20` con marca `b1` y categoría `c1`, y `p21`…`p25` con marca `b2` y categoría
`c2`. `p1` se llama `"Zapatilla Runner"` (`sku: "SKU-001"`, `price: 89.90`, publicado); el resto
quedan en `draft` con precios entre `10.00` y `250.00`.

**When**: `getProduct` — `GET /api/v1/management/products/{p1}`, token con `product:read`

**Then**:
1. Status `200` con la proyección completa de back-office (los 12 campos propios más `brand{}`,
   `category{}` e `images[]`), incluidos `createdBy` y `updatedBy`.

**When**: `listProducts` — `GET /api/v1/management/products`

**Then**:
2. Status `200` con el sobre `{ items, page: 0, size: 20, totalElements: 25, totalPages: 2 }`.
3. `items` trae 20 elementos, ordenados por `updatedAt` **descendente**, con el `id` como
   desempate; cada elemento con la proyección completa de back-office.
4. `GET /api/v1/management/products?page=1` devuelve `page: 1` y 5 elementos, sin repetir ninguno
   de la página 0.
5. `GET /api/v1/management/products?page=2` devuelve `items: []`, `page: 2`, `totalElements: 25`.
6. `GET /api/v1/management/products?size=500` devuelve `size: 100` (recortado al tope), no un error.

**When**: `GET /api/v1/management/products?status=draft&brandId=<b2>`

**Then**:
7. Status `200` con `totalElements: 5`: los filtros se combinan con conjunción.

**When**: `GET /api/v1/management/products?name=runner`

**Then**:
8. Status `200` e `items` incluye `p1` (`"Zapatilla Runner"`): la búsqueda es parcial y no
   distingue mayúsculas.

**When**: `GET /api/v1/management/products?name=RÚNNER`

**Then**:
9. Status `200` e `items` incluye `p1`: la búsqueda tampoco distingue acentos.

**When**: `GET /api/v1/management/products?sku=sku-001`

**Then**:
10. Status `200` con `totalElements: 1` (`p1`).

**Casos borde**:
- `getProduct` sobre un id inexistente → `404`, `code: PRODUCT_NOT_FOUND`.
- `listProducts` sin ningún resultado → `200` con `items: []` y `totalElements: 0`, nunca `404`.
- `listProducts` devuelve productos en **cualquier** estado, incluidos `discontinued`.

**Notas de determinación**: para que el orden por `updatedAt` sea distinguible de cualquier otro,
los productos se crean con una separación observable y el escenario comprueba que `p25` (el último
tocado) es `items[0]`.

---

## Imágenes del producto

### FL-IMG-001: galería completa, orden, imagen principal y borrado

**Given**: existen `b1`, `c1` y el producto `p1` (`status: "draft"`, sin imágenes), creados dentro
del flujo. El bucket `productImages` admite `image/jpeg`, `image/png` e `image/webp`, con un máximo
de 5 MB.

**When**: `addProductImage` — `POST /api/v1/management/products/{p1}/images`, token con
`product:write`, cabecera `Idempotency-Key: img-key-001`, cuerpo multiparte con un JPEG de 200 KB
y `altText: "Zapatilla vista lateral"`

**Then**:
1. Status `201`.
2. Cabecera `Location` con la ruta de la imagen creada.
3. El cuerpo trae `id` (uuid), `file` (referencia al objeto del bucket `productImages`),
   `altText: "Zapatilla vista lateral"`, `position: 0` e `isPrimary: true` — la primera imagen de
   un producto queda marcada como principal. No trae campos de auditoría.
4. Una lectura directa de la referencia de `file` devuelve el binario subido: el bucket es
   `public`, así que no hace falta URL firmada ni operación de descarga mediada.
5. `getProduct` sobre `{p1}` devuelve `images` con **un** elemento igual al del punto 3.
6. Se publica `ProductUpdated` en `productEvents`, con `primaryImage` apuntando al objeto subido.

**When**: se repite la misma subida con `Idempotency-Key: img-key-001`

**Then**:
7. Status `201` con el **mismo** `id` de imagen y `getProduct` sigue devolviendo **una** imagen:
   el reintento tras un timeout de subida no duplica la foto.

**When**: `addProductImage` dos veces más (imágenes `img2` e `img3`), con claves de idempotencia
distintas

**Then**:
8. `img2` sale con `position: 1` e `isPrimary: false`; `img3` con `position: 2` e
   `isPrimary: false`: cada imagen se añade al final.
9. `getProduct` devuelve `images` con 3 elementos ordenados por `position` ascendente
   (`img1`, `img2`, `img3`) y **exactamente uno** con `isPrimary: true`.

**When**: `setPrimaryProductImage` — `PUT /api/v1/management/products/{p1}/images/{img3}/primary`

**Then**:
10. Status `200` con la ficha del producto.
11. `getProduct` devuelve `img3` con `isPrimary: true` e `img1` con `isPrimary: false`: sigue
    habiendo exactamente una principal.
12. Las `position` no cambiaron.
13. Se publica `ProductUpdated`.

**When**: se repite `setPrimaryProductImage` sobre `img3`, que ya es la principal

**Then**:
14. Status `200` y estado idéntico: designar la que ya lo es se considera éxito.

**When**: `reorderProductImages` — `PUT /api/v1/management/products/{p1}/images/order`
```json
{ "imageIds": ["<img3>", "<img1>", "<img2>"] }
```

**Then**:
15. Status `200`.
16. `getProduct` devuelve `img3` con `position: 0`, `img1` con `1` e `img2` con `2`: la posición es
    el índice en la lista recibida, empezando en cero.
17. `img3` sigue siendo `isPrimary: true`: reordenar no cambia cuál es la principal.
18. Se publica `ProductUpdated`.

**When**: `removeProductImage` — `DELETE /api/v1/management/products/{p1}/images/{img3}`

**Then**:
19. Status `204`, sin cuerpo.
20. `getProduct` devuelve 2 imágenes con `position` `0` y `1`, sin huecos (se recompactan).
21. `img1`, la de menor `position` de las que quedan, pasa a `isPrimary: true`.
22. La referencia del `file` de `img3` deja de resolver: el archivo se elimina del bucket junto con
    la imagen.
23. Se publica `ProductUpdated`.

**When**: `removeProductImage` sobre `img3` **otra vez**, y después sobre un `imageId` que nunca
existió

**Then**:
24. Status `204` en ambas, sin cuerpo: el borrado es idempotente.
25. `getProduct` sigue devolviendo las mismas 2 imágenes, con `position` `0` y `1` y `img1` como
    principal: ninguna de las dos repeticiones altera la galería.
26. Ninguna de las dos publica `ProductUpdated`: el evento solo sale cuando se eliminó una imagen.

**When**: se publica `p1` (queda `active` con 2 imágenes), se borra una y se intenta borrar la
última

**Then**:
27. Status `422`, `code: LAST_IMAGE_OF_ACTIVE_PRODUCT`: un producto `active` no puede quedarse sin
    imágenes.
28. `getProduct` sigue devolviendo 1 imagen.

**Orden de evaluación** (`addProductImage`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está descatalogado → `PRODUCT_DISCONTINUED` (`409`).
3. El producto tiene menos de 10 imágenes → `TOO_MANY_PRODUCT_IMAGES` (`422`).
4. El content-type está entre los admitidos → `UNSUPPORTED_CONTENT_TYPE` (`415`).
5. El archivo no supera 5 MB → `FILE_TOO_LARGE` (`413`).
6. La clave de idempotencia no se usó con otro contenido → `IDEMPOTENCY_KEY_REUSED` (`409`).

**Orden de evaluación** (`setPrimaryProductImage`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está descatalogado → `PRODUCT_DISCONTINUED` (`409`).
3. La imagen pertenece a ese producto → `PRODUCT_IMAGE_NOT_FOUND` (`404`).

**Orden de evaluación** (`removeProductImage`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`). El producto de la ruta sí tiene que existir:
   la idempotencia alcanza a la imagen, no al contenedor.
2. El producto no está descatalogado → `PRODUCT_DISCONTINUED` (`409`).
3. Si `imageId` no está en la galería de ese producto, no hay guarda que falle: `204` sin efecto.
   `removeProductImage` **no** declara `PRODUCT_IMAGE_NOT_FOUND`, a diferencia de
   `setPrimaryProductImage`.
4. No es la última imagen de un producto `active` → `LAST_IMAGE_OF_ACTIVE_PRODUCT` (`422`). Solo
   se evalúa si la imagen existe: sobre una imagen ausente gana la guarda 3.

**Orden de evaluación** (`reorderProductImages`):
1. El producto existe → `PRODUCT_NOT_FOUND` (`404`).
2. El producto no está descatalogado → `PRODUCT_DISCONTINUED` (`409`).
3. La lista coincide exactamente con las imágenes del producto → `IMAGE_SET_MISMATCH` (`422`).

**Casos borde**:
- Subir un `application/pdf` → `415`, `code: UNSUPPORTED_CONTENT_TYPE`.
- Subir un JPEG de 6 MB → `413`, `code: FILE_TOO_LARGE`.
- Subir un PDF de 6 MB → `415`, `code: UNSUPPORTED_CONTENT_TYPE`: la guarda 4 precede a la 5.
- Subir una 11.ª imagen → `422`, `code: TOO_MANY_PRODUCT_IMAGES`.
- Un PDF sobre un producto inexistente → `404`, `code: PRODUCT_NOT_FOUND`: la guarda 1 precede a la 4.
- `altText` ausente → `400`; `altText` de 161 caracteres → `400`.
- `removeProductImage` con un `imageId` de **otro** producto → `204`, sin cuerpo y sin evento: no
  está en la galería del producto de la ruta, y la operación no distingue ese caso de una imagen ya
  borrada. La imagen ajena **no** se elimina: `getProduct` sobre su producto sigue devolviéndola.
- `setPrimaryProductImage` con un `imageId` de **otro** producto → `404`,
  `code: PRODUCT_IMAGE_NOT_FOUND`: esa operación no es idempotente y sí distingue el caso.
- `removeProductImage` sobre un producto **descatalogado**, con un `imageId` inexistente → `409`,
  `code: PRODUCT_DISCONTINUED`: la guarda 2 precede a la 3, y la idempotencia no la salta.
- `removeProductImage` sobre un `productId` inexistente → `404`, `code: PRODUCT_NOT_FOUND`, incluso
  con un `imageId` también inexistente: el contenedor de la ruta queda fuera de la idempotencia.
- `reorderProductImages` con la lista a la que le falta una imagen, o con una repetida, o con una
  ajena → `422`, `code: IMAGE_SET_MISMATCH` en los tres casos.
- `reorderProductImages` con `imageIds: []` → `400` (`required` sobre una lista significa presente
  y no vacía).

**Notas de determinación**: `file` se verifica por forma (referencia resoluble al bucket
`productImages`) y por el binario que devuelve, nunca por el valor literal de la clave, que es
decisión del generador.

---

## Tienda (superficie pública)

### FL-PUB-001: ficha pública por slug y campos que no salen

**Given**: existen `b1` (`"Nike"`), `c1` (`"Calzado"`), el producto `p1`
(`slug: "zapatilla-runner"`, `price: 89.90`, con una imagen, **publicado**) y el producto `p2`
(`slug: "camiseta-basica"`, en `draft`), creados dentro del flujo.

**When**: `getPublicProduct` — `GET /api/v1/products/zapatilla-runner`, **sin credencial**

**Then**:
1. Status `200` (no hace falta autenticarse).
2. El cuerpo trae exactamente `id`, `sku`, `name`, `slug`, `description`, `price: 89.90`,
   `status: "active"`, `brand{ id, name, slug, description }`,
   `category{ id, name, slug, description }` e `images[]`.
3. El cuerpo **no** trae `createdAt`, `updatedAt`, `createdBy`, `updatedBy` ni `lockVersion`,
   ni en el producto ni dentro de `brand` o `category`.
4. `images` trae un elemento `{ id, file, altText, position: 0, isPrimary: true }`.

**When**: `GET /api/v1/products/camiseta-basica` (producto en `draft`)

**Then**:
5. Status `404`, `code: PRODUCT_NOT_FOUND`: lo que no está `active` es inexistente para la tienda,
   y la respuesta no distingue "no existe" de "no publicado".

**When**: `GET /api/v1/products/no-existe`

**Then**:
6. Status `404`, `code: PRODUCT_NOT_FOUND`, idéntico al punto 5.

### FL-PUB-010: listado de tienda con los cuatro filtros

**Given**: existen `b1` (`"Nike"`, slug `nike`), `b2` (`"Adidas"`, slug `adidas`), `c1`
(`"Calzado"`, slug `calzado`), `c2` (`"Camisetas"`, slug `camisetas`) y 30 productos creados dentro
del flujo: 25 **publicados** (20 de `b1`/`c1`, 5 de `b2`/`c2`, con precios entre `10.00` y
`250.00`, uno de ellos llamado `"Zapatilla Runner"` a `89.90`) y 5 en `draft`.

**When**: `listPublicProducts` — `GET /api/v1/products`, **sin credencial**

**Then**:
1. Status `200` con `{ items, page: 0, size: 20, totalElements: 25, totalPages: 2 }`: los 5 en
   `draft` no aparecen.
2. `items` viene ordenado por `name` **ascendente**, con el `id` como desempate, y cada elemento
   trae la proyección pública (sin auditoría ni `lockVersion`).
3. `?page=1` devuelve 5 elementos sin repetir ninguno de la página 0; `?page=2` devuelve
   `items: []`; `?size=500` devuelve `size: 100`.

**When**: `GET /api/v1/products?categorySlug=calzado&brandSlug=nike`

**Then**:
4. Status `200` con `totalElements: 20`: los filtros se combinan con conjunción.

**When**: `GET /api/v1/products?minPrice=50.00&maxPrice=100.00`

**Then**:
5. Status `200` y todos los `items` tienen `price` entre `50.00` y `100.00`, **ambos incluidos**;
   un producto a exactamente `50.00` y otro a exactamente `100.00` aparecen en el resultado.

**When**: `GET /api/v1/products?name=runner`

**Then**:
6. Status `200` e `items` incluye `"Zapatilla Runner"`: coincidencia parcial, sin distinguir
   mayúsculas ni acentos.

**When**: `GET /api/v1/products?minPrice=100.00&maxPrice=50.00`

**Then**:
7. Status `422`, `code: INVALID_PRICE_RANGE`.

**When**: `GET /api/v1/products?brandSlug=marca-que-no-existe`

**Then**:
8. Status `200` con `items: []` y `totalElements: 0`: un slug desconocido devuelve página vacía,
   **no** un error.

**Casos borde**:
- `?minPrice=abc` → `400`.
- `?minPrice=50.00` sin `maxPrice` → `200`, filtra solo por el mínimo.
- Un producto que pasa a `discontinued` desaparece del listado en la siguiente lectura no cacheada.

### FL-PUB-020: taxonomía pública para los filtros de la tienda

**Given**: existen 3 marcas (`"Adidas"`, `"Nike"`, `"Puma"`) y 2 categorías (`"Calzado"`,
`"Camisetas"`), creadas dentro del flujo.

**When**: `listBrands` — `GET /api/v1/brands`, **sin credencial**

**Then**:
1. Status `200` con `{ items, page: 0, size: 20, totalElements: 3, totalPages: 1 }`.
2. `items` ordenado por `name` ascendente: `Adidas`, `Nike`, `Puma`.
3. Cada elemento trae exactamente `id`, `name`, `slug`, `description` y ningún campo más.

**When**: `listCategories` — `GET /api/v1/categories`

**Then**:
4. Status `200` con `totalElements: 2`, ordenado por `name` ascendente: `Calzado`, `Camisetas`.
5. Misma proyección de cuatro campos.

**Casos borde**: sin ninguna marca, `listBrands` devuelve `items: []` y `totalElements: 0`.

---

## Caché de la superficie pública

### FL-CCH-001: invalidación por cada una de las cinco vías declaradas

`getPublicProduct` y `listPublicProducts` declaran `cache: { ttlSeconds: 300 }` invalidada por
`ProductCreated`, `ProductUpdated`, `ProductStatusChanged`, `BrandUpdated` y `CategoryUpdated`.
Este flujo ejercita **un ciclo leer → mutar → releer por cada vía**; sin los cinco, una invalidación
incompleta pasaría inadvertida.

**Given**: existen `b1` (`"Nike"`), `c1` (`"Calzado"`) y el producto `p1`
(`slug: "zapatilla-runner"`, `price: 89.90`, con una imagen, publicado), creados dentro del flujo.

**Then**, para cada vía, y siempre **dentro** de los 300 s del TTL:

1. **`ProductUpdated`**: `getPublicProduct("zapatilla-runner")` → `price: 89.90`.
   `updateProduct` con `price: 79.90` → `getPublicProduct` devuelve `price: 79.90`, no `89.90`.
   `listPublicProducts` también refleja `79.90`.
2. **`BrandUpdated`**: `getPublicProduct` → `brand.name: "Nike"`. `updateBrand` con
   `name: "Nike Sportswear"` → `getPublicProduct` devuelve `brand.name: "Nike Sportswear"` y
   `brand.slug: "nike-sportswear"`. Es la vía que más se olvida: el dato que cambia está en el
   objeto **embebido**, no en el producto.
3. **`CategoryUpdated`**: ídem con `updateCategory` y `category.name`.
4. **`ProductStatusChanged`**: `discontinueProduct` sobre `p1` → `getPublicProduct` devuelve `404`
   y `listPublicProducts` deja de incluirlo, sin esperar al TTL.
5. **`ProductCreated`**: se lee `listPublicProducts` (`totalElements: N`), se crea un producto nuevo
   y se publica; la siguiente lectura refleja el nuevo total. (El alta por sí sola no cambia el
   resultado, porque nace en `draft`: quien lo hace visible es `ProductStatusChanged`, y por eso
   las dos vías se ejercitan por separado.)

**Notas de determinación**: el escenario de **retención** canónico —mutar por una vía que **no**
está en `invalidatedBy` y comprobar que se sigue sirviendo el valor viejo— **no es expresable en
este diseño**: `invalidatedBy` cubre todas las vías que pueden cambiar una respuesta cacheada, que
es justo la propiedad que se buscaba. La consecuencia es un límite conocido de esta cobertura: los
cinco ciclos de arriba los pasa igual una implementación que **no cachee nada**. FL-CCH-020 cubre
el fallo de cacheo que sí es observable.

### FL-CCH-020: la clave de caché incluye la paginación

`cache.keyFields` de `listPublicProducts` nombra los cinco filtros, pero **no** puede nombrar `page`
ni `size`, que no son campos del input sino parámetros del sobre de paginación. Una implementación
que construya la clave solo con los `keyFields` declarados serviría la página 0 para cualquier
página. Este escenario lo cierra: **la clave de caché incluye siempre `page` y `size` además de los
`keyFields`.**

**Given**: existen 25 productos publicados, creados dentro del flujo, con nombres distinguibles
(`"Producto 01"` … `"Producto 25"`).

**When**: `GET /api/v1/products?page=0&size=20`, y a continuación `GET /api/v1/products?page=1&size=20`

**Then**:
1. La primera responde `page: 0` con 20 elementos que empiezan por `"Producto 01"`.
2. La segunda responde `page: 1` con 5 elementos que empiezan por `"Producto 21"`, y **ninguno**
   coincide con los de la página 0.
3. `GET /api/v1/products?page=0&size=5` devuelve `size: 5` y 5 elementos, no los 20 de la lectura
   anterior.

---

## Superficie servidor a servidor

### FL-M2M-001: ficha de un producto para otro servidor

**Given**: existen `b1`, `c1`, el producto `p1` (`sku: "SKU-001"`, publicado, con una imagen) y el
producto `p2` (`sku: "SKU-002"`, **descatalogado**), creados dentro del flujo.

**When**: `getProductForServices` — `GET /api/v1/services/products/{p1}`, credencial de máquina del
cliente `order-service` con el scope `product:read`

**Then**:
1. Status `200`.
2. El cuerpo trae exactamente `id`, `sku`, `name`, `slug`, `description`, `price`, `status`,
   `updatedAt`, `brand{ id, name, slug, description }`,
   `category{ id, name, slug, description }` e `images[]`.
3. El cuerpo **no** trae `createdAt`, `createdBy`, `updatedBy` ni `lockVersion`.
4. Sí trae `updatedAt`: es lo que permite al consumidor ordenar lo recibido contra su propia copia
   al reconciliar tras perder un evento.

**When**: `GET /api/v1/services/products/{p2}` (producto descatalogado)

**Then**:
5. Status `200` con `status: "discontinued"`: a diferencia de la tienda, el M2M devuelve el producto
   sea cual sea su estado, porque el consumidor puede referenciar un producto que ya no se vende.

**When**: `GET /api/v1/services/products/{uuid inexistente}`

**Then**:
6. Status `404`, `code: PRODUCT_NOT_FOUND`.

**When**: la misma llamada con credencial de máquina **sin** el scope `product:read`

**Then**:
7. Status `403`.

**When**: la misma llamada con un token de máquina emitido para **otra** audiencia
(`serviceAuth.validateAudience: true`)

**Then**:
8. La llamada se rechaza sin ejecutar la operación: un token válido de otro servicio del ecosistema
   no abre el catálogo.

**When**: la misma llamada **sin** credencial

**Then**:
9. Status `401`.

### FL-M2M-010: resolución por lotes

**Given**: existen 3 productos `p1`, `p2`, `p3` (`sku` `SKU-001`, `SKU-002`, `SKU-003`; nombres
`"Alfa"`, `"Beta"`, `"Gamma"`), creados dentro del flujo.

**When**: `listProductsBatchForServices` — `POST /api/v1/services/products/batch`, credencial de
máquina del cliente `order-service`
```json
{ "ids": ["<p3>", "<p1>", "<uuid inexistente>", "<p1>"] }
```

**Then**:
1. Status `200`.
2. El cuerpo es una lista de **2** elementos: `p1` y `p3`.
3. Vienen ordenados por `name` ascendente (`"Alfa"`, `"Gamma"`), con el `id` como desempate — **no**
   en el orden de la petición.
4. El uuid inexistente **se omite sin error**, y `p1`, repetido en la petición, aparece **una sola
   vez**.
5. Cada elemento trae la misma proyección M2M de FL-M2M-001 (con `updatedAt`, sin `createdAt`,
   `createdBy`, `updatedBy` ni `lockVersion`), con `brand` y `category` anidados.

**When**: la misma llamada con 101 identificadores

**Then**:
6. Status `422`, `code: TOO_MANY_IDS`.

**When**: la misma llamada con 100 identificadores

**Then**:
7. Status `200`: 100 es la cota superior incluida.

**Casos borde**:
- `ids: []` → `400` (`required` sobre una lista significa presente y no vacía).
- `ids` ausente → `400`.
- Todos los ids inexistentes → `200` con lista vacía, nunca `404`.
- Con credencial de máquina sin `product:read` → `403`; sin credencial → `401`.

**Notas de determinación**: el orden de la respuesta es el declarado (`name` ascendente), **no** el
de la petición. Es la diferencia más fácil de resolver distinto entre dos generadores.

---

## Autorización

### FL-SEC-001: sin credencial y con credencial insuficiente

**Given**: existen `b1`, `c1` y el producto `p1` (publicado, con una imagen), creados dentro del
flujo con un token que sí tiene los permisos.

**Then**, para **cada** operación protegida:

1. **Sin credencial** → `401`, en las 20 operaciones de `/management/...` y en las 2 de
   `/services/...`.
2. **Con token de usuario válido pero sin el permiso exigido** → `403`:
   - sin `product:read`: `getProduct`, `listProducts`;
   - sin `product:write`: `createProduct`, `updateProduct`, `addProductImage`,
     `removeProductImage`, `setPrimaryProductImage`, `reorderProductImages`;
   - sin `product:publish`: `publishProduct`, `unpublishProduct`, `discontinueProduct`,
     `reactivateProduct`;
   - sin `taxonomy:write`: `createBrand`, `updateBrand`, `deleteBrand`, `createCategory`,
     `updateCategory`, `deleteCategory`.
3. **Con credencial de máquina sin `product:read`** → `403` en `getProductForServices` y
   `listProductsBatchForServices`.
4. **Las cuatro operaciones públicas responden `200` sin ninguna credencial**:
   `listPublicProducts`, `getPublicProduct`, `listBrands`, `listCategories`.
5. El rechazo por autorización ocurre **antes** que cualquier guarda de negocio: `updateProduct`
   sin `product:write` sobre un id inexistente responde `403`, no `404`.
6. Ninguna operación de mutación es accesible sin credencial.

**Notas de determinación**: el fallo de audiencia de un token de máquina y el fallo de scope los
resuelve el generador; el escenario fija que la llamada **no ejecuta la operación** y que el estado
no cambia, comprobado leyendo por la API con una credencial válida después.
