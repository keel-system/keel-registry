---
service: catalog
version: 0.3.0
domain: commerce
basePath: /api/v1
m2mAuth:
  protocol: client-credentials
  audience: catalog
  validateAudience: true
endpoints:
  - name: getProductForServices
    method: GET
    path: /internal/products/{productId}
    access: service product:read
  - name: getProductsByIds
    method: POST
    path: /internal/products/batch
    access: service product:read
events:
  envelope: keel
  published:
    - name: ProductCreated
      channel: productEvents
    - name: ProductUpdated
      channel: productEvents
    - name: ProductRetired
      channel: productEvents
  consumed: []
errors:
  - { code: PRODUCT_NOT_FOUND, http: 404 }
---

# Integración con catalog

## Resumen

`catalog` mantiene el catálogo comercial de productos —con su marca y su categoría— como fuente de
verdad para el resto del sistema (dominio commerce). A otros servidores les ofrece dos vías de
integración: una superficie de lectura servidor-a-servidor para resolver productos por id o por lote,
con su marca y su categoría incorporadas, y los eventos de alta, cambio y baja de producto para que
cada consumidor mantenga su propia copia local al día.

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los `scopes` que tu
   cliente tiene concedidos (ver tabla). Fija la audiencia `aud: catalog` (se valida).

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=product:read&audience=catalog
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| order-service | product:read | Mantiene una copia local de la ficha de producto para componer pedidos, alimentada por los eventos del catálogo y reconciliada con los endpoints por lote. |

### getProductForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/internal/products/{productId}` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |

**Request** — path `productId: uuid` (requerido).

**Response** — el producto en **cualquier estado** (`draft`, `active` o `retired`); este endpoint no
filtra por estado, el consumidor decide qué hacer con él. Cacheada 300 s por `productId`, invalidada
por `ProductUpdated` y `ProductRetired`.

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | requerido |
| sku | SKU | requerido; string, patrón `^[A-Z0-9][A-Z0-9-]{3,19}$` |
| name | string | requerido |
| description | string | opcional |
| price.amount | decimal | requerido; escala 2, mínimo 0 |
| price.currency | string | requerido; ISO-4217 en mayúsculas |
| status | ProductStatus | requerido; `draft` \| `active` \| `retired` |
| tags | string[] | opcional; hasta 10 |
| brand | objeto | requerido; `{ id, name, slug }`, sin relaciones propias (profundidad 1) |
| category | objeto | requerido; `{ id, name, slug }`, sin relaciones propias (profundidad 1) |
| images | objeto[] | galería ordenada por `position`; cada elemento `{ id, position, altText, file, createdAt }` |
| images[].file | string (uri) | URL absoluta de lectura del objeto en el bucket público `productImages` |
| createdAt | timestamp | requerido |
| updatedAt | timestamp | requerido |

```json
{
  "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "ACME-RUN-01",
  "name": "Zapatilla Run",
  "description": "Zapatilla de running neutra",
  "price": { "amount": 89.90, "currency": "EUR" },
  "status": "active",
  "tags": ["running", "neutra"],
  "brand": { "id": "b1", "name": "Acme", "slug": "acme" },
  "category": { "id": "c1", "name": "Calzado", "slug": "calzado" },
  "images": [
    { "id": "i1", "position": 1, "altText": "Vista lateral", "file": "https://cdn.example.com/product-images/i1.jpg", "createdAt": "2026-03-14T09:21:07.482Z" }
  ],
  "createdAt": "2026-03-14T09:20:00.000Z",
  "updatedAt": "2026-03-14T09:21:07.482Z"
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PRODUCT_NOT_FOUND | 404 | No existe un producto con ese `productId`. | No reintentar; el recurso no existe. |

### getProductsByIds

| | |
|---|---|
| Endpoint | `POST /api/v1/internal/products/batch` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| productIds | uuid[] | requerido; entre 1 y 100 elementos |

```json
{ "productIds": ["p3", "p1", "p2"] }
```

**Response** — una **lista** (no un sobre de paginación) con el mismo contrato de producto que
`getProductForServices`, en el orden pedido. Los identificadores inexistentes se omiten sin error y sin
hueco en la lista; un identificador repetido en la petición se resuelve una sola vez.

```json
[
  { "id": "p3", "sku": "ACME-RET-01", "name": "...", "status": "retired", "brand": { "id": "b1", "name": "Acme", "slug": "acme" }, "category": { "id": "c1", "name": "Calzado", "slug": "calzado" }, "images": [], "createdAt": "...", "updatedAt": "..." },
  { "id": "p1", "sku": "ACME-RUN-01", "name": "Zapatilla Run", "status": "active", "brand": { "id": "b1", "name": "Acme", "slug": "acme" }, "category": { "id": "c1", "name": "Calzado", "slug": "calzado" }, "images": [], "createdAt": "...", "updatedAt": "..." }
]
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| (validación) 400 | 400 | `productIds` ausente, vacío (`minItems: 1`), con más de 100 elementos (`maxItems: 100`), o con un elemento que no es uuid. | Corregir input. |

## Eventos

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProductCreated",
    "eventVersion": 1,
    "occurredAt": "2026-03-14T09:21:07.482Z",
    "source": "catalog",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1"
  },
  "data": { "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "sku": "ACME-RUN-01" }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`ProductCreated`). Discriminador si el canal transporta varios tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| metadata.source | string | Servicio emisor: `catalog`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Canal**: los tres eventos viajan por el canal lógico `productEvents` (el topic/cola físico que lo
respalda es parámetro de despliegue, no de diseño).

**Garantía de entrega**: `best-effort`. Es una decisión de diseño explícita — se acepta perder un
evento si el broker está caído en el instante del commit; sin outbox no hay reconciliación automática.
`getProductsByIds` es la vía por la que un consumidor resincroniza su copia cuando lo necesita.

> Los eventos llevan `brandId` y `categoryId`, nunca los nombres de marca y categoría: un renombrado
> dejaría rancio el nombre en cada copia local sin forma de detectarlo. Resuelve el nombre por
> `getProductForServices` / `getProductsByIds`.

### ProductCreated

Se dio de alta un producto, siempre en estado `draft`.

- **Canal**: `productEvents`.
- **Emitido por**: `createProduct`.

**Payload de ejemplo** (contenido de `data`):

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "ACME-RUN-01",
  "name": "Zapatilla Run",
  "description": "Zapatilla de running neutra",
  "price": { "amount": 89.90, "currency": "EUR" },
  "status": "draft",
  "brandId": "b1",
  "categoryId": "c1",
  "tags": ["running", "neutra"],
  "createdAt": "2026-03-14T09:20:00.000Z"
}
```

| Campo | Tipo | Notas |
|---|---|---|
| productId | uuid | requerido |
| sku | SKU | requerido |
| name | string | requerido |
| description | string | opcional |
| price | Money | requerido; `{ amount: decimal, currency: string }` |
| status | ProductStatus | requerido; siempre `draft` en este evento |
| brandId | uuid | requerido |
| categoryId | uuid | requerido |
| tags | string[] | opcional; hasta 10 |
| createdAt | timestamp | requerido |

### ProductUpdated

Cambió la ficha de un producto. Cubre también la publicación, que llega como `status: "active"`.

- **Canal**: `productEvents`.
- **Emitido por**: `updateProduct`, `publishProduct`, `addProductImage`, `removeProductImage`,
  `reorderProductImages`.

**Payload de ejemplo** (contenido de `data`):

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "ACME-RUN-01",
  "name": "Zapatilla Run II",
  "description": "Zapatilla de running neutra",
  "price": { "amount": 99.00, "currency": "EUR" },
  "status": "active",
  "brandId": "b2",
  "categoryId": "c1",
  "tags": ["running", "neutra"],
  "imageUrls": ["https://cdn.example.com/product-images/i1.jpg"],
  "updatedAt": "2026-03-14T09:25:00.000Z"
}
```

| Campo | Tipo | Notas |
|---|---|---|
| productId | uuid | requerido |
| sku | SKU | requerido |
| name | string | requerido |
| description | string | opcional |
| price | Money | requerido |
| status | ProductStatus | requerido |
| brandId | uuid | requerido |
| categoryId | uuid | requerido |
| tags | string[] | opcional; hasta 10 |
| imageUrls | string[] (uri) | opcional; hasta 8, ordenadas por posición, la primera es la principal. Ausente si el producto no tiene imágenes. |
| updatedAt | timestamp | requerido |

### ProductRetired

Un producto se retiró del catálogo de forma definitiva. Es la vía de baja de toda copia local.

- **Canal**: `productEvents`.
- **Emitido por**: `retireProduct`.

**Payload de ejemplo** (contenido de `data`):

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "ACME-RUN-01",
  "retiredAt": "2026-03-20T11:00:00.000Z"
}
```

| Campo | Tipo | Notas |
|---|---|---|
| productId | uuid | requerido |
| sku | SKU | requerido |
| retiredAt | timestamp | requerido |

### Suscripciones

`catalog` no consume eventos de ningún otro servidor: no hay suscripciones declaradas en su capa
`messaging`.
