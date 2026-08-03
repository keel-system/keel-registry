---
service: catalog
version: 0.2.0
domain: commerce
basePath: /api/v1
m2mAuth:
  protocol: client-credentials
  audience: catalog
  validateAudience: true
endpoints:
  - name: getProductForServices
    method: GET
    path: /services/products/{productId}
    access: service product:read
  - name: listProductsBatchForServices
    method: POST
    path: /services/products/batch
    access: service product:read
events:
  envelope: keel
  published:
    - name: ProductCreated
      channel: productEvents
    - name: ProductUpdated
      channel: productEvents
    - name: ProductStatusChanged
      channel: productEvents
    - name: BrandCreated
      channel: taxonomyEvents
    - name: BrandUpdated
      channel: taxonomyEvents
    - name: BrandDeleted
      channel: taxonomyEvents
    - name: CategoryCreated
      channel: taxonomyEvents
    - name: CategoryUpdated
      channel: taxonomyEvents
    - name: CategoryDeleted
      channel: taxonomyEvents
errors:
  - code: PRODUCT_NOT_FOUND
    http: 404
  - code: TOO_MANY_IDS
    http: 422
---

# Integración con catalog

## Resumen

`catalog` es la fuente de verdad del catálogo comercial de una tienda: productos, marcas y
categorías, con su ficha, su precio de venta y su galería de imágenes. A otros servidores les ofrece
dos cosas: **resolución de fichas de producto** —una por id, o hasta cien de golpe— con un contrato
propio y estable, distinto del que sirve a la tienda; y **nueve eventos de dominio** con los que un
consumidor puede mantener su propia copia del catálogo sin llamar de vuelta. Los endpoints de esta
sección devuelven el producto **en cualquier estado**, incluidos los descatalogados, porque un
consumidor puede referenciar un producto que ya no se vende.

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los `scopes` que tu
   cliente tiene concedidos (ver tabla). Fija la audiencia `aud: catalog`, **que se valida**: un
   token emitido para otro servicio del ecosistema no abre este.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=product:read&audience=catalog
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| `order-service` | `product:read` | Resuelve las fichas de los productos de un carrito o de un pedido, casi siempre por lotes. |
| `inventory-service` | `product:read` | Mantiene el stock por producto y necesita saber qué productos existen y cuáles salen del catálogo. |
| `search-service` | `product:read` | Mantiene su índice de búsqueda con los eventos del catálogo y reconcilia por lotes. |

**Forma del producto en M2M.** Las dos operaciones devuelven la misma proyección:

| Campo | Tipo | Notas |
|---|---|---|
| `id` | uuid | requerido |
| `sku` | string | requerido; código comercial, siempre en mayúsculas |
| `name` | string | requerido; máx. 140 |
| `slug` | string | requerido; identificador de URL, estable desde la primera publicación |
| `description` | text \| null | máx. 4000 |
| `price` | decimal | requerido; escala 2, divisa única de la tienda |
| `status` | enum | requerido; `draft` \| `active` \| `discontinued` |
| `updatedAt` | timestamp | requerido; ISO-8601 UTC. **Úsalo para ordenar lo recibido contra tu copia** al reconciliar |
| `brand` | objeto | requerido; `{ id, name, slug, description }` |
| `category` | objeto | requerido; `{ id, name, slug, description }` |
| `images` | lista | `[{ id, file, altText, position, isPrimary }]`, ordenada por `position` ascendente; `[]` si no tiene |

No viajan `createdAt`, `createdBy`, `updatedBy` ni `lockVersion`: son auditoría interna del
back-office. `brand` y `category` viajan **resueltos como objeto**, no como identificador, para que
no necesites una segunda llamada.

### getProductForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/products/{productId}` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |
| Caché | ninguna: la respuesta es siempre el estado vigente |

**Request** — path `productId: uuid` (requerido).

**Response** — la forma del producto en M2M descrita arriba.

```json
{
  "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-001",
  "name": "Zapatilla Runner",
  "slug": "zapatilla-runner",
  "description": "Zapatilla de running para asfalto.",
  "price": 89.90,
  "status": "active",
  "updatedAt": "2026-08-02T09:21:07.482Z",
  "brand": {
    "id": "7c1f9a20-1b33-4d55-8e77-2a9b4c6d8e10",
    "name": "Nike",
    "slug": "nike",
    "description": "Ropa y calzado deportivo."
  },
  "category": {
    "id": "b45e0c11-9d22-4f66-a1b3-5e7f9a0c2d44",
    "name": "Calzado",
    "slug": "calzado",
    "description": "Zapatillas y botas."
  },
  "images": [
    {
      "id": "e91a7b04-6c58-4a12-9f30-1d5b8c2e7a66",
      "file": "https://<storage-host>/productImages/e91a7b04-6c58-4a12-9f30-1d5b8c2e7a66.jpg",
      "altText": "Zapatilla vista lateral",
      "position": 0,
      "isPrimary": true
    }
  ]
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `PRODUCT_NOT_FOUND` | 404 | No existe un producto con ese id. | No reintentar; el recurso no existe. Si tu copia lo referenciaba, purgarla. |

### listProductsBatchForServices

| | |
|---|---|
| Endpoint | `POST /api/v1/services/products/batch` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |
| Método | `POST` sobre una lectura, a propósito: cien uuid no caben en una URL |
| Cota | **100 identificadores** por llamada; trocea si necesitas más |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| `ids` | lista de uuid | requerido y no vacío; máximo 100 |

```json
{ "ids": ["3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "a17c5d88-4e91-4b02-9c6f-38d0a1e7b455"] }
```

**Response** — lista de productos con la forma M2M descrita arriba, **ordenada por `name`
ascendente** (con el `id` como desempate), **no** en el orden de la petición.

Tres comportamientos que conviene tener presentes al consumirlo:

- Los identificadores que **no existen se omiten** del resultado, sin error. Compara lo pedido con
  lo recibido si necesitas detectar bajas.
- Los identificadores **repetidos** se resuelven una sola vez.
- Devuelve productos en **cualquier estado**, `discontinued` incluido.

```json
[
  {
    "id": "a17c5d88-4e91-4b02-9c6f-38d0a1e7b455",
    "sku": "SKU-002",
    "name": "Camiseta Básica",
    "slug": "camiseta-basica",
    "description": null,
    "price": 19.90,
    "status": "discontinued",
    "updatedAt": "2026-07-28T14:03:55.120Z",
    "brand": { "id": "7c1f9a20-1b33-4d55-8e77-2a9b4c6d8e10", "name": "Nike", "slug": "nike", "description": "Ropa y calzado deportivo." },
    "category": { "id": "c02d7e33-8a14-4b90-b7e2-6f1a3c5d9e77", "name": "Camisetas", "slug": "camisetas", "description": null },
    "images": []
  },
  {
    "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
    "sku": "SKU-001",
    "name": "Zapatilla Runner",
    "slug": "zapatilla-runner",
    "description": "Zapatilla de running para asfalto.",
    "price": 89.90,
    "status": "active",
    "updatedAt": "2026-08-02T09:21:07.482Z",
    "brand": { "id": "7c1f9a20-1b33-4d55-8e77-2a9b4c6d8e10", "name": "Nike", "slug": "nike", "description": "Ropa y calzado deportivo." },
    "category": { "id": "b45e0c11-9d22-4f66-a1b3-5e7f9a0c2d44", "name": "Calzado", "slug": "calzado", "description": "Zapatillas y botas." },
    "images": [ { "id": "e91a7b04-6c58-4a12-9f30-1d5b8c2e7a66", "file": "https://<storage-host>/productImages/e91a7b04-...jpg", "altText": "Zapatilla vista lateral", "position": 0, "isPrimary": true } ]
  }
]
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `TOO_MANY_IDS` | 422 | La petición trae más de 100 identificadores. | No reintentar igual; trocea la lista en lotes de 100 y repite. |

## Eventos

El contrato formal de estos eventos está en [`asyncapi.yaml`](asyncapi.yaml); aquí va lo que hace
falta para consumirlos.

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El
payload del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProductCreated",
    "eventVersion": 1,
    "occurredAt": "2026-08-02T09:21:07.482Z",
    "source": "catalog",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": {
    "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
    "sku": "SKU-001"
  }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `metadata.eventId` | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| `metadata.eventType` | string | Nombre del evento (`ProductCreated`). Discriminador si el canal transporta varios tipos. |
| `metadata.eventVersion` | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| `metadata.occurredAt` | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| `metadata.source` | string | Servicio emisor: `catalog`. |
| `metadata.correlationId` | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| `data` | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

> **Garantía de entrega: `best-effort`.** La publicación **no** comparte transacción con el cambio
> de estado. Si el broker está caído en el instante en que la escritura confirma, **ese evento se
> pierde sin traza**: no hay error, no hay reintento y nada aguas arriba lo delata. Es una decisión
> deliberada del diseño, y la contrapartida es que **la reconciliación es responsabilidad del
> consumidor**: usa `listProductsBatchForServices` para refrescar por lotes los productos que
> conozcas, y compara el `updatedAt` recibido con el de tu copia. Si tu caso de uso no tolera perder
> un evento, no construyas sobre la suscripción sola.

Dos canales lógicos (el topic o cola real es decisión de despliegue, no se documenta aquí):

- **`productEvents`** — altas, cambios de ficha y cambios de estado comercial de los productos.
- **`taxonomyEvents`** — altas, cambios y bajas de marcas y categorías.

### ProductCreated

Se dio de alta un producto. **Nace en `draft`, así que todavía no está a la venta**: no lo publiques
en una vitrina por recibir este evento.

- **Canal**: `productEvents`
- **Emitido por**: `createProduct`

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-001",
  "name": "Zapatilla Runner",
  "slug": "zapatilla-runner",
  "description": "Zapatilla de running para asfalto.",
  "price": 89.90,
  "status": "draft",
  "brandId": "7c1f9a20-1b33-4d55-8e77-2a9b4c6d8e10",
  "brandName": "Nike",
  "categoryId": "b45e0c11-9d22-4f66-a1b3-5e7f9a0c2d44",
  "categoryName": "Calzado",
  "primaryImage": null,
  "updatedAt": "2026-08-02T09:21:07.482Z"
}
```

`primaryImage` viene ausente mientras el producto no tenga ninguna imagen. `updatedAt` es el momento
del cambio en el catálogo: **úsalo para descartar eventos que lleguen desordenados**.

### ProductUpdated

Cambió la ficha comercial de un producto, su galería de imágenes o su imagen principal. Trae la
ficha completa ya actualizada, así que no necesitas llamar de vuelta.

- **Canal**: `productEvents`
- **Emitido por**: `updateProduct`, `addProductImage`, `removeProductImage`,
  `setPrimaryProductImage`, `reorderProductImages`
- **Desde `removeProductImage`, solo cuando se eliminó una imagen**: ese borrado es idempotente y
  repetirlo responde `204` sin publicar este evento.

Mismo payload que `ProductCreated`.

### ProductStatusChanged

Un producto entró o salió de la venta. **Es el evento que basta seguir** si lo único que te importa
es si el producto sigue disponible: no tienes que procesar cada cambio de descripción.

- **Canal**: `productEvents`
- **Emitido por**: `publishProduct`, `unpublishProduct`, `discontinueProduct`, `reactivateProduct`

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-001",
  "slug": "zapatilla-runner",
  "previousStatus": "draft",
  "newStatus": "active",
  "updatedAt": "2026-08-02T09:24:11.905Z"
}
```

Transiciones posibles: `draft → active`, `active → draft`, `active → discontinued`,
`discontinued → active`. Un producto **nunca se borra**: `discontinued` es la baja comercial y su
ficha sigue siendo consultable por M2M.

### BrandCreated

Se dio de alta una marca en el catálogo.

- **Canal**: `taxonomyEvents`
- **Emitido por**: `createBrand`

```json
{
  "brandId": "7c1f9a20-1b33-4d55-8e77-2a9b4c6d8e10",
  "name": "Nike",
  "slug": "nike",
  "updatedAt": "2026-08-01T11:02:33.007Z"
}
```

### BrandUpdated

Cambió el nombre o la reseña de una marca; **las fichas de producto que la referencian quedan
desactualizadas** en tu copia. Si guardas `brandName`, actualízalo con este evento.

- **Canal**: `taxonomyEvents`
- **Emitido por**: `updateBrand`

Mismo payload que `BrandCreated`.

### BrandDeleted

Se eliminó una marca. Quien mantenga una copia de la taxonomía **debe purgarla**. El catálogo solo
permite borrar una marca que ningún producto referencia, así que este evento nunca deja fichas
huérfanas.

- **Canal**: `taxonomyEvents`
- **Emitido por**: `deleteBrand`
- **Solo cuando hubo borrado real**: `deleteBrand` es idempotente y responde `204` también sobre una
  marca que ya no existe; en ese caso **no** publica este evento. No esperes un evento por cada
  llamada de borrado que veas tener éxito.

```json
{
  "brandId": "7c1f9a20-1b33-4d55-8e77-2a9b4c6d8e10",
  "slug": "nike",
  "deletedAt": "2026-08-02T16:40:19.331Z"
}
```

### CategoryCreated

Se dio de alta una categoría en el catálogo.

- **Canal**: `taxonomyEvents`
- **Emitido por**: `createCategory`

```json
{
  "categoryId": "b45e0c11-9d22-4f66-a1b3-5e7f9a0c2d44",
  "name": "Calzado",
  "slug": "calzado",
  "updatedAt": "2026-08-01T11:05:47.612Z"
}
```

### CategoryUpdated

Cambió el nombre o el texto de una categoría; las fichas que la referencian quedan desactualizadas.

- **Canal**: `taxonomyEvents`
- **Emitido por**: `updateCategory`

Mismo payload que `CategoryCreated`.

### CategoryDeleted

Se eliminó una categoría. Quien mantenga una copia de la taxonomía debe purgarla. Igual que con las
marcas, solo se borra una categoría que ningún producto referencia.

- **Canal**: `taxonomyEvents`
- **Emitido por**: `deleteCategory`
- **Solo cuando hubo borrado real**: igual que `BrandDeleted`, un `deleteCategory` repetido responde
  `204` sin publicar un segundo evento.

```json
{
  "categoryId": "b45e0c11-9d22-4f66-a1b3-5e7f9a0c2d44",
  "slug": "calzado",
  "deletedAt": "2026-08-02T16:41:02.884Z"
}
```

### Suscripciones

`catalog` **no consume ningún evento**: no hay nada que publicar para activar una de sus
operaciones. Toda interacción entrante es HTTP, por los endpoints de la sección anterior.
