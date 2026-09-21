---
service: catalog
version: 0.1.0
domain: commerce
basePath: /api/v1
m2mAuth:
  protocol: client-credentials
  audience: catalog
  validateAudience: true
endpoints:
  - name: getProductForServices
    method: GET
    path: /internal/products/{id}
    access: service product:read
  - name: listProductsBatchForServices
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
---

# Integración con catalog

## Resumen

`catalog` es la fuente de verdad del catálogo comercial de una tienda: productos con sus imágenes, marcas y categorías. A otros servidores les ofrece dos cosas. La primera son **dos endpoints de lectura**, por id y por lote de hasta 100 ids, que devuelven el producto **en cualquier estado, incluido `retired`**, para que un pedido o una cesta resuelvan referencias históricas. La segunda son **eventos** con la ficha completa en cada alta y cada cambio, más los cambios de estado y la taxonomía (marcas y categorías), para quien quiera mantener una copia local sin llamar de vuelta. El servicio no consume eventos de nadie: no tiene suscripciones. Contratos formales: [`openapi.yaml`](openapi.yaml) (HTTP) y [`asyncapi.yaml`](asyncapi.yaml) (eventos).

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client credentials), no con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los `scopes` que tu cliente tiene concedidos (ver tabla). Fija la audiencia `aud: catalog`: **se valida**, y un token emitido para otro servicio se rechaza.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=product:read&audience=catalog
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| orders | product:read | Resuelve productos por id y por lote al confirmar y facturar pedidos; consume los eventos del catálogo. |
| cart | product:read | Resuelve la cesta entera en una llamada por lote y refresca precios y disponibilidad comercial. |

Errores comunes a los dos endpoints, además de los propios de cada uno:

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| — | 401 | Sin credencial, o credencial inválida o caducada. | No reintentar con el mismo token; pide uno nuevo. |
| — | 403 | El token no tiene el scope `product:read`, o se emitió para otra audiencia. | No reintentar; revisa scopes y audiencia con el dueño del servicio. |
| — | 400 | La petición incumple la forma del contrato (id que no es uuid, lote vacío o de más de 100). | Corregir input. |
| — | 5xx / timeout | Fallo del servicio. | Reintentable, con backoff. Las dos operaciones son lecturas: repetirlas no tiene efecto. |

Los errores sin `code` de negocio llevan el cuerpo de error del servicio sin un código que distinga el caso: se distinguen por el status.

**Convenciones del payload.** Un campo sin valor viaja como `null`, nunca se omite. `price.amount` es un número con escala 2 y `price.currency` es un código ISO 4217; el catálogo opera en una sola moneda. `images[].file` es la **URL absoluta** de la imagen, de lectura anónima. `images` viaja siempre ordenada por `position` ascendente, y exactamente una imagen lleva `main: true` cuando hay alguna.

### getProductForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/internal/products/{id}` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |

**Request** — path `id: uuid` (requerido). Sin cuerpo.

**Response** — `200`

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | requerido |
| sku | string | requerido; `^[A-Z0-9][A-Z0-9_-]{1,31}$`, siempre en mayúsculas |
| name | string | requerido; ≤ 160 |
| description | string \| null | ≤ 4000 |
| price.amount | decimal | requerido; ≥ 0, escala 2 |
| price.currency | string | requerido; ISO 4217 |
| status | `draft` \| `active` \| `retired` | requerido. **Decide tú** qué hacer con un producto que ya no se vende. |
| brand | objeto | requerido; `{ id: uuid, name: string, description: string \| null, active: boolean }` |
| category | objeto | requerido; `{ id: uuid, name: string, description: string \| null, active: boolean }` |
| images | lista (≤ 10) | requerido; cada una `{ id: uuid, file: URL, altText: string \| null, position: int 0–9, main: boolean }` |

No devuelve campos de auditoría (`createdAt`, `updatedAt`, `createdBy`, `updatedBy`).

```json
{
  "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "TS-RED-M",
  "name": "Camiseta Básica Roja",
  "description": "Algodón orgánico.",
  "price": { "amount": 19.90, "currency": "EUR" },
  "status": "active",
  "brand": { "id": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c", "name": "Acme", "description": null, "active": true },
  "category": { "id": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f", "name": "Camisetas", "description": null, "active": true },
  "images": [
    { "id": "e8f9a0b1-2c3d-4e5f-9a6b-7c8d9e0f1a2b", "file": "https://images.example.invalid/productImages/3d2e1f00/front.jpg", "altText": "Frontal", "position": 0, "main": true }
  ]
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PRODUCT_NOT_FOUND | 404 | No existe un producto con ese id. | No reintentar; el recurso no existe. Un producto retirado **no** da 404: se devuelve con `status: retired`. |

### listProductsBatchForServices

| | |
|---|---|
| Endpoint | `POST /api/v1/internal/products/batch` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query; va por `POST` porque 100 uuid no caben con seguridad en una URL) |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| ids | lista de uuid | requerido; entre 1 y 100 elementos |

```json
{ "ids": ["3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "5b6c7d8e-9f0a-4b1c-8d2e-3f4a5b6c7d8e", "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9"] }
```

**Response** — `200`: una **lista** (sin sobre de paginación) de productos con la misma forma que `getProductForServices`, ordenada por `name` ascendente (con desempate por `id`).

- Se devuelven los productos que existen, en **cualquier estado**.
- Un id que no existe **se omite**, sin error. El resultado puede tener menos elementos que ids pediste: detectar cuáles faltan es trabajo tuyo, cruzando por `id`.
- Un id repetido aparece una sola vez.

```json
[
  {
    "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
    "sku": "TS-RED-M",
    "name": "Camiseta Básica Roja",
    "description": "Algodón orgánico.",
    "price": { "amount": 19.90, "currency": "EUR" },
    "status": "active",
    "brand": { "id": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c", "name": "Acme", "description": null, "active": true },
    "category": { "id": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f", "name": "Camisetas", "description": null, "active": true },
    "images": [
      { "id": "e8f9a0b1-2c3d-4e5f-9a6b-7c8d9e0f1a2b", "file": "https://images.example.invalid/productImages/3d2e1f00/front.jpg", "altText": "Frontal", "position": 0, "main": true }
    ]
  }
]
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| — | 400 | `ids` vacío, con más de 100 elementos o con algún valor que no es uuid. | Corregir input; parte el lote en trozos de 100. |

## Eventos

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProductCreated",
    "eventVersion": 1,
    "occurredAt": "2026-03-14T09:21:07.482Z",
    "source": "catalog",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": { "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "sku": "TS-RED-M" }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`ProductCreated`). Discriminador: cada canal de este servicio transporta varios tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. Úsalo para descartar un evento más viejo que tu copia. |
| metadata.source | string | Servicio emisor: `catalog`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| metadata.traceparent | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Garantía de entrega de todos los eventos de esta sección: outbox.** El evento se escribe en la misma transacción que el cambio: ningún evento se pierde si la transacción confirma, aunque el broker esté caído en ese instante. La entrega es **at-least-once** y el orden entre eventos no está garantizado, así que deduplica por `metadata.eventId` y ordena por `metadata.occurredAt`.

**Imágenes en los eventos.** En `data.images[].file` viaja la **clave del objeto** en el bucket, no la URL (una URL en un evento caduca y ata el mensaje al almacenamiento). Para mostrar la imagen, resuelve el producto por `getProductForServices`, que la devuelve como URL absoluta.

**Sin cambio, sin evento.** Una edición que no cambia ningún valor no publica nada.

Canales lógicos (el topic o cola físico que los respalda es parámetro de despliegue):

| Canal | Qué transporta |
|---|---|
| productEvents | Altas, cambios y cambios de estado de los productos. |
| taxonomyEvents | Altas, cambios y bajas de marcas y categorías. |

### ProductCreated

| | |
|---|---|
| Canal | `productEvents` |
| Garantía | outbox |
| Emitido por | `createProduct` |

Se dio de alta un producto. Nace en `draft`, así que **todavía no está a la venta**. `data` es la ficha completa:

| Campo | Tipo | Notas |
|---|---|---|
| productId | uuid | requerido |
| sku | string | requerido; en mayúsculas |
| name | string | requerido |
| description | string \| null | |
| price | `{ amount: decimal, currency: string }` | requerido |
| status | `draft` \| `active` \| `retired` | requerido |
| brandId | uuid | requerido |
| brandName | string | requerido |
| categoryId | uuid | requerido |
| categoryName | string | requerido |
| images | lista (≤ 10) | cada una `{ imageId: uuid, file: clave, altText: string \| null, position: int, main: boolean }` |

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "TS-RED-M",
  "name": "Camiseta Básica Roja",
  "description": "Algodón orgánico.",
  "price": { "amount": 19.90, "currency": "EUR" },
  "status": "draft",
  "brandId": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c",
  "brandName": "Acme",
  "categoryId": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
  "categoryName": "Camisetas",
  "images": []
}
```

### ProductUpdated

| | |
|---|---|
| Canal | `productEvents` |
| Garantía | outbox |
| Emitido por | `updateProduct`, `addProductImage`, `updateProductImage`, `removeProductImage` |

Cambió algo de la ficha: datos comerciales o imágenes. `data` es la **ficha completa tras el cambio**, no el delta, con la misma forma que `ProductCreated`: sustituye tu copia entera. El paso de estado **no** viaja aquí: tiene su propio evento, `ProductStatusChanged`.

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "TS-RED-M",
  "name": "Camiseta Básica Roja",
  "description": "Algodón orgánico.",
  "price": { "amount": 24.50, "currency": "EUR" },
  "status": "active",
  "brandId": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c",
  "brandName": "Acme",
  "categoryId": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f",
  "categoryName": "Camisetas",
  "images": [
    { "imageId": "e8f9a0b1-2c3d-4e5f-9a6b-7c8d9e0f1a2b", "file": "productImages/3d2e1f00/front.jpg", "altText": "Frontal", "position": 0, "main": true }
  ]
}
```

### ProductStatusChanged

| | |
|---|---|
| Canal | `productEvents` |
| Garantía | outbox |
| Emitido por | `publishProduct`, `unpublishProduct`, `retireProduct` |

Un producto se publicó (`draft → active`), se despublicó (`active → draft`) o se retiró (`draft|active → retired`). Es el evento que debes leer como **alta o baja comercial**. `retired` es terminal: un producto retirado no vuelve.

| Campo | Tipo | Notas |
|---|---|---|
| productId | uuid | requerido |
| sku | string | requerido |
| previousStatus | `draft` \| `active` \| `retired` | requerido |
| status | `draft` \| `active` \| `retired` | requerido |
| reason | string \| null | ≤ 200; solo lo lleva una retirada con motivo |

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "TS-RED-M",
  "previousStatus": "active",
  "status": "retired",
  "reason": "Fin de temporada"
}
```

### BrandCreated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Garantía | outbox |
| Emitido por | `createBrand` |

Se dio de alta una marca.

| Campo | Tipo | Notas |
|---|---|---|
| brandId | uuid | requerido |
| name | string | requerido |
| description | string \| null | |
| active | boolean | requerido |

```json
{ "brandId": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c", "name": "Acme", "description": null, "active": true }
```

### BrandUpdated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Garantía | outbox |
| Emitido por | `updateBrand`, `activateBrand`, `deactivateBrand` |

Cambió el nombre, la descripción o la actividad de una marca. **Es lo que te permite corregir `brandName` en tu copia de productos sin reindexarlos**: renombrar una marca no publica ningún `ProductUpdated`. Desactivar una marca no retira sus productos. Misma forma que `BrandCreated`.

```json
{ "brandId": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c", "name": "Acme Outdoor", "description": null, "active": true }
```

### BrandDeleted

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Garantía | outbox |
| Emitido por | `deleteBrand` |

Se borró una marca. Solo se pueden borrar marcas sin productos, así que ningún producto de tu copia apunta a ella.

| Campo | Tipo | Notas |
|---|---|---|
| brandId | uuid | requerido |
| name | string | requerido |

```json
{ "brandId": "7a1c0b22-5f3e-4d8a-9c10-2b3e4f5a6b7c", "name": "Acme" }
```

### CategoryCreated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Garantía | outbox |
| Emitido por | `createCategory` |

Se dio de alta una categoría.

| Campo | Tipo | Notas |
|---|---|---|
| categoryId | uuid | requerido |
| name | string | requerido |
| description | string \| null | |
| active | boolean | requerido |

```json
{ "categoryId": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f", "name": "Camisetas", "description": null, "active": true }
```

### CategoryUpdated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Garantía | outbox |
| Emitido por | `updateCategory`, `activateCategory`, `deactivateCategory` |

Cambió el nombre, la descripción o la actividad de una categoría. Úsalo para corregir `categoryName` en tu copia; desactivar una categoría no retira sus productos. Misma forma que `CategoryCreated`.

```json
{ "categoryId": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f", "name": "Camisetas y polos", "description": null, "active": true }
```

### CategoryDeleted

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Garantía | outbox |
| Emitido por | `deleteCategory` |

Se borró una categoría sin productos.

| Campo | Tipo | Notas |
|---|---|---|
| categoryId | uuid | requerido |
| name | string | requerido |

```json
{ "categoryId": "c4d5e6f7-1a2b-4c3d-8e9f-0a1b2c3d4e5f", "name": "Camisetas" }
```

### Suscripciones

Este servicio no se suscribe a ningún evento: no tiene puertas de entrada por mensajería, y nadie puede encargarle trabajo publicando en un canal. Toda escritura entra por su API de gestión, que no es superficie servidor-a-servidor.
