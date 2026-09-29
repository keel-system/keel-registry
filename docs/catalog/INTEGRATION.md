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
    access: service product:read-internal
  - name: getProductsBatch
    method: POST
    path: /internal/products/batch
    access: service product:read-internal
events:
  envelope: keel
  published:
    - name: ProductCreated
      channel: productEvents
    - name: ProductUpdated
      channel: productEvents
  consumed: []
errors:
  - code: PRODUCT_NOT_FOUND
    http: 404
---

# Integración con catalog

## Resumen

`catalog` (dominio `commerce`) es la fuente de verdad del catálogo comercial: productos con sus
imágenes, marcas y categorías. A otros servidores les ofrece dos cosas. La primera es **resolver
productos** en cualquier estado (`draft`, `active` o `retired`), por id o por lote de hasta 100 ids,
con marca y categoría anidadas. Sirve, por ejemplo, para que un servidor de pedidos resuelva un
producto ya retirado de un pedido antiguo. La segunda es **mantener una copia local** a partir de los
eventos `ProductCreated` y `ProductUpdated`, que llevan la foto completa del producto tras cada cambio
y una `version` monotónica para descartar los que lleguen desordenados. Todo precio va en la moneda
única del catálogo, un parámetro de despliegue (ISO 4217 de 2 decimales). El servicio no convierte
divisas.

Contratos formales: [`openapi.yaml`](openapi.yaml) (HTTP) y [`asyncapi.yaml`](asyncapi.yaml) (eventos).

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), no con un token de usuario. Para obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y el scope concedido
   (`product:read-internal`). Fija la audiencia `aud: catalog`, porque se valida: un token emitido
   para otro servicio responde `403 ACCESS_DENIED`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=product:read-internal&audience=catalog
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| orders | product:read-internal | Servidor de pedidos: resuelve los productos de un pedido, también los ya retirados. |
| cart | product:read-internal | Servidor de carrito: resuelve los productos de un carrito para pintarlo y validarlo. |

Un consumidor nuevo necesita que el dueño del servicio lo dé de alta como cliente máquina. Es una
evolución del diseño (`security.serviceClients`), no una configuración.

**Forma del producto M2M.** Los dos endpoints devuelven el producto con la misma forma:

| Campo | Tipo | Notas |
|---|---|---|
| id | uuid | requerido |
| sku | string | requerido; `^[A-Z0-9][A-Z0-9_-]{1,31}$`, siempre en mayúsculas. Solo cambia mientras el producto no se ha publicado nunca (`firstPublishedAt: null`). |
| name | string | requerido; 1..160 |
| description | text \| null | ≤ 4000 |
| price | objeto | requerido; `{ amount: decimal (escala 2), currency: string ISO 4217 }` |
| status | enum | requerido; `draft` \| `active` \| `retired` |
| firstPublishedAt | timestamp \| null | ISO-8601 UTC de la primera publicación; `null` si nunca se publicó |
| lockVersion | int | requerido; versión del producto. Es la misma que viaja como `version` en los eventos |
| brand | objeto | requerido; `{ id: uuid, name: string, description: text \| null, active: boolean }` |
| category | objeto | requerido; `{ id: uuid, name: string, description: text \| null, active: boolean }` |
| images | array | requerido (`[]` si no tiene); hasta 10, ordenadas por `position` ascendente |
| images[].id | uuid | requerido |
| images[].file | string (URI) | requerido; URL absoluta del objeto en el bucket público `productImages` |
| images[].altText | string \| null | ≤ 160 |
| images[].position | int | requerido; 0..9, consecutivas desde 0 |
| images[].main | boolean | requerido; exactamente una `true` si hay imágenes |
| images[].productId | uuid | requerido; el id del producto |

No viajan `createdAt`, `updatedAt`, `createdBy` ni `updatedBy`, ni la versión o la auditoría de la
marca y la categoría. Un campo sin valor viaja como `null`, y una colección vacía como `[]`.

### getProductForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/internal/products/{id}` |
| Acceso | `service` — scopes `product:read-internal` |
| Idempotencia | no aplica (query) |
| Caché | ninguna en el servicio: la respuesta siempre es fresca |

**Request**: `id: uuid` en el path (requerido). Sin cuerpo.

**Response**: `200` con el producto M2M (forma arriba), **en cualquier estado**.

```json
{
  "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "AB-100",
  "name": "Zapatilla Runner",
  "description": "Zapatilla ligera de running.",
  "price": { "amount": 59.90, "currency": "EUR" },
  "status": "active",
  "firstPublishedAt": "2026-09-29T10:15:00Z",
  "lockVersion": 4,
  "brand": { "id": "6b1c5e2a-0d3f-4c8e-9a21-3f7d9b0c1e44", "name": "Acme", "description": null, "active": true },
  "category": { "id": "a7e4d3c2-1b0f-4e9d-8c7b-6a5f4e3d2c1b", "name": "Calzado", "description": null, "active": true },
  "images": [
    {
      "id": "c0ffee00-1234-4abc-9def-0123456789ab",
      "file": "https://images.example.invalid/productImages/c0ffee00-1234-4abc-9def-0123456789ab.jpg",
      "altText": "Vista lateral",
      "position": 0,
      "main": true,
      "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9"
    }
  ]
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| PRODUCT_NOT_FOUND | 404 | No existe un producto con ese id. | No reintentar; el producto no existe (un producto retirado sí existe y responde `200`). |
| VALIDATION_ERROR | 400 | El `id` no es un uuid. | Corregir el input. |
| UNAUTHENTICATED | 401 | Sin token o con un token inválido o caducado. | Pedir un token nuevo y reintentar una vez. |
| ACCESS_DENIED | 403 | El token no tiene el scope `product:read-internal`, es un token de usuario o está emitido para otra audiencia. | No reintentar; revisar la credencial. |

### getProductsBatch

| | |
|---|---|
| Endpoint | `POST /api/v1/internal/products/batch` |
| Acceso | `service` — scopes `product:read-internal` |
| Idempotencia | no aplica (query; se expone por `POST` porque 100 ids no caben con holgura en una URL) |
| Caché | ninguna |

**Request**

| Campo | Tipo | Notas |
|---|---|---|
| ids | uuid[] | requerido; entre 1 y 100 elementos |

```json
{ "ids": ["3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "00000000-0000-0000-0000-000000000000", "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9"] }
```

**Response**: `200` con una lista de productos M2M (forma arriba):

- en cualquier estado;
- **ordenados por `sku` ascendente**, no por el orden de la petición: casa cada producto por su `id`;
- los ids que no existen se omiten sin error, y un id repetido aparece una sola vez;
- una petición con solo ids inexistentes devuelve `[]`.

```json
[
  {
    "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
    "sku": "AB-100",
    "name": "Zapatilla Runner",
    "description": "Zapatilla ligera de running.",
    "price": { "amount": 59.90, "currency": "EUR" },
    "status": "active",
    "firstPublishedAt": "2026-09-29T10:15:00Z",
    "lockVersion": 4,
    "brand": { "id": "6b1c5e2a-0d3f-4c8e-9a21-3f7d9b0c1e44", "name": "Acme", "description": null, "active": true },
    "category": { "id": "a7e4d3c2-1b0f-4e9d-8c7b-6a5f4e3d2c1b", "name": "Calzado", "description": null, "active": true },
    "images": []
  }
]
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| VALIDATION_ERROR | 400 | `ids` falta, está vacío, tiene más de 100 elementos o contiene algo que no es un uuid. | Corregir el input; partir los lotes grandes en trozos de 100. |
| UNAUTHENTICATED | 401 | Sin token o con un token inválido o caducado. | Pedir un token nuevo y reintentar una vez. |
| ACCESS_DENIED | 403 | Falta el scope, es un token de usuario o es de otra audiencia. | No reintentar; revisar la credencial. |

Un `5xx` o un timeout en cualquiera de los dos endpoints es **reintentable**: son lecturas y no tienen
efecto que duplicar.

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
    "occurredAt": "2026-09-29T09:21:07.482Z",
    "source": "catalog",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": { "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "sku": "AB-100", "…": "…" }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| metadata.eventId | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| metadata.eventType | string | Nombre del evento (`ProductCreated` o `ProductUpdated`). Discriminador: el canal transporta los dos tipos. |
| metadata.eventVersion | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| metadata.occurredAt | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. |
| metadata.source | string | Servicio emisor: `catalog`. |
| metadata.correlationId | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| metadata.traceparent | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| data | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Los dos eventos llevan el mismo `data`**, la foto completa del producto tras el cambio:

| Campo | Tipo | Notas |
|---|---|---|
| productId | uuid | requerido |
| sku | string | requerido; en mayúsculas |
| name | string | requerido |
| description | text \| null | |
| price | objeto | requerido; `{ amount: decimal, currency: string }` |
| status | enum | requerido; `draft` \| `active` \| `retired` |
| brand | objeto | requerido; `{ id: uuid, name: string }` con el nombre **en el instante del hecho** |
| category | objeto | requerido; `{ id: uuid, name: string }` con el nombre en el instante del hecho |
| images | array | ≤ 10, ordenadas por `position`; `[]` si no hay |
| images[].imageId | uuid | requerido |
| images[].file | string | requerido; **key** del objeto en el bucket `productImages`, no una URL |
| images[].altText | string \| null | |
| images[].position | int | requerido |
| images[].main | boolean | requerido |
| version | int | requerido; la `lockVersion` del producto tras el cambio. Estrictamente creciente por producto |

Reglas para mantener la copia:

- **Upsert por `productId`**, descartando el evento si su `version` es **menor o igual** que la que ya
  aplicaste. Así, un evento que llega desordenado no pisa uno más nuevo.
- **Deduplica por `metadata.eventId`**, o simplemente por `version`, que tiene el mismo efecto.
- **El producto nunca desaparece**: una retirada llega como `ProductUpdated` con `status: retired`.
  No hay evento de borrado.
- **Los nombres de marca y categoría son una foto del instante del hecho.** Renombrar o desactivar una
  marca o categoría **no publica nada**, así que tu copia conserva el nombre viejo hasta el siguiente
  cambio de cada producto. Si necesitas el nombre vigente, resuélvelo con `getProductForServices`.
- **URL de una imagen**: el bucket es público. La URL es la base pública del bucket `productImages`
  (un dato de despliegue que te comunica el dueño del servicio) más la `file` del evento. Si prefieres
  no conocer esa base, resuelve el producto con `getProductForServices`, que devuelve URLs absolutas.

### ProductCreated

| | |
|---|---|
| Canal lógico | `productEvents`: altas y cambios de producto del catálogo. El topic o la cola física es decisión de despliegue |
| Garantía de entrega | outbox: ningún evento se pierde si la transacción confirma. La entrega es at-least-once |
| Emitido por | `createProduct` |

Se dio de alta un producto. Siempre nace en `draft` y sin imágenes. Los valores de `version` de los ejemplos son
ilustrativos: el contrato es que crece estrictamente con cada cambio, no su valor inicial.

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "AB-100",
  "name": "Zapatilla Runner",
  "description": null,
  "price": { "amount": 59.90, "currency": "EUR" },
  "status": "draft",
  "brand": { "id": "6b1c5e2a-0d3f-4c8e-9a21-3f7d9b0c1e44", "name": "Acme" },
  "category": { "id": "a7e4d3c2-1b0f-4e9d-8c7b-6a5f4e3d2c1b", "name": "Calzado" },
  "images": [],
  "version": 1
}
```

### ProductUpdated

| | |
|---|---|
| Canal lógico | `productEvents` |
| Garantía de entrega | outbox: ningún evento se pierde si la transacción confirma. La entrega es at-least-once |
| Emitido por | `updateProduct`, `publishProduct`, `unpublishProduct`, `retireProduct`, `addProductImage`, `updateProductImage`, `removeProductImage` |

Cambió algo del producto: datos comerciales, estado (publicación, despublicación o retirada) o
imágenes. Un cambio nulo, en el que nada cambia, no publica.

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "AB-100",
  "name": "Zapatilla Runner",
  "description": "Zapatilla ligera de running.",
  "price": { "amount": 59.90, "currency": "EUR" },
  "status": "active",
  "brand": { "id": "6b1c5e2a-0d3f-4c8e-9a21-3f7d9b0c1e44", "name": "Acme" },
  "category": { "id": "a7e4d3c2-1b0f-4e9d-8c7b-6a5f4e3d2c1b", "name": "Calzado" },
  "images": [
    { "imageId": "c0ffee00-1234-4abc-9def-0123456789ab", "file": "productImages/c0ffee00-1234-4abc-9def-0123456789ab.jpg", "altText": "Vista lateral", "position": 0, "main": true }
  ],
  "version": 4
}
```

### Suscripciones

`catalog` no se suscribe a ningún evento: no tiene puertas de entrada por mensaje (`nature: request`)
ni reacciona a hechos ajenos (`nature: fact`). Solo se le encarga trabajo por su API.
