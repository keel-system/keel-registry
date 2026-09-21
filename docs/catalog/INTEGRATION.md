---
service: catalog
version: 0.1.2
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
    method: GET
    path: /services/products
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
    - name: ProductDeleted
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
  - code: EMPTY_ID_LIST
    http: 422
  - code: TOO_MANY_IDS
    http: 422
---

# Integración con catalog

## Resumen

`catalog` es la fuente de verdad del catálogo comercial de una tienda: los productos, sus marcas y sus
categorías. A otros servidores les ofrece dos cosas y solo dos: **resolver productos** —uno por su id o
varios por lote, con la marca y la categoría ya resueltas— y **enterarse de los cambios** por eventos,
para que un consumidor mantenga su propia copia o su propio índice sin preguntar. No lleva existencias,
no conoce variantes de un artículo y no convierte divisas: si necesitas stock, no es este servicio.

El catálogo **no consume eventos de nadie ni llama a ningún otro servidor**, así que la integración es
en un solo sentido: tú le lees y le escuchas, él no te necesita para funcionar.

## Endpoints expuestos a otros servidores

Los endpoints de esta sección se consumen con un **token de cliente máquina** (OAuth2 client
credentials), nunca con token de usuario. Cómo obtenerlo:

1. Pide al dueño del servicio tus credenciales de cliente (`clientId` + `clientSecret`) y la URL del
   endpoint de token del proveedor de identidad (`tokenUrl`), que varía por entorno. El diseño declara
   el protocolo, no el producto: ni el `tokenUrl` ni los secretos están aquí.
2. Solicita un token con `grant_type=client_credentials`, tus credenciales y los scopes que tu cliente
   tiene concedidos (ver tabla). Fija la audiencia `catalog`: **se valida**, así que un token emitido
   para otro servicio del mismo proveedor de identidad se rechaza con `403`.

   ```
   POST {tokenUrl}
   Content-Type: application/x-www-form-urlencoded

   grant_type=client_credentials&client_id=...&client_secret=...&scope=product:read&audience=catalog
   ```

3. Envía el `access_token` recibido en cada llamada como `Authorization: Bearer <access_token>`.

| Cliente | Scopes concedidos | Propósito |
|---|---|---|
| `orders-service` | `product:read` | Resuelve la ficha y el precio de los artículos de un pedido, por id y por lote. |
| `cart-service` | `product:read` | Resuelve por lote los artículos que el comprador tiene en el carrito. |
| `search-service` | `product:read` | Mantiene su índice de búsqueda con los eventos del catálogo y usa el lote para reconstruirlo. |

Si tu servicio no está en esa tabla, no hay credencial que aprovisionar: pídele al dueño del catálogo
que te declare como cliente máquina antes de integrarte.

**Lo que estos endpoints no devuelven, a propósito**: la autoría de la ficha (`createdBy`, `updatedBy`)
ni el control de versión (`lockVersion`). Quién gestiona la tienda no sale de la tienda, y el candado de
concurrencia es de quien escribe, no de quien lee.

**Qué productos ve un servidor.** Se resuelven los productos `active` y los `discontinued` —un pedido
histórico referencia artículos que ya no se venden—, pero **un producto en `draft` responde como si no
existiera**: un borrador no es todavía un artículo del catálogo.

### getProductForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/products/{productId}` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |
| Caché | la respuesta se cachea 60 s en el servidor, invalidada por los cambios del producto y de su marca o categoría |

**Request** — path `productId: uuid` (requerido). Sin cuerpo.

**Response**

| Campo | Tipo | Notas |
|---|---|---|
| `id` | uuid | requerido |
| `sku` | string | requerido; código comercial, único e inmutable (patrón `^[A-Z0-9][A-Z0-9-]{2,31}$`) |
| `name` | string | requerido; hasta 140 caracteres |
| `slug` | string | requerido; identificador de la URL pública. Se congela en la primera publicación: es estable |
| `description` | string | opcional; **no viaja** si está vacía |
| `price` | decimal | requerido; escala 2, en la divisa única de la tienda |
| `status` | enum | requerido; `draft` \| `active` \| `discontinued` — nunca verás `draft` por esta puerta |
| `createdAt` | timestamp | requerido; UTC ISO-8601 |
| `updatedAt` | timestamp | requerido; UTC ISO-8601. **De aquí sabes de cuándo es la ficha que recibes** |
| `brand` | objeto | requerido; `{ id, name, slug, description? }` — resuelto, no un id |
| `category` | objeto | requerido; `{ id, name, slug, description? }` — resuelto, no un id |
| `images` | array | galería ordenada por `position`; cada elemento `{ id, image, position, primary, altText? }`. `image` es la URL pública del objeto, de lectura anónima. No lleva `productId`: la imagen ya viene dentro de su producto |

```json
{
  "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-M01",
  "name": "Laptop Pro 14",
  "slug": "laptop-pro-14",
  "description": "Portátil de 14 pulgadas, 16 GB de RAM.",
  "price": 1299.00,
  "status": "active",
  "createdAt": "2026-05-14T09:21:07.482Z",
  "updatedAt": "2026-05-20T11:04:55.120Z",
  "brand": { "id": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001", "name": "Acme", "slug": "acme" },
  "category": { "id": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002", "name": "Laptops", "slug": "laptops" },
  "images": [
    {
      "id": "e1a0b2c3-3333-4c33-9e03-cccc0000d003",
      "image": "https://cdn.example.com/productImages/3d2e1f00/front.jpg",
      "position": 0,
      "primary": true,
      "altText": "Vista frontal del producto"
    }
  ]
}
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `PRODUCT_NOT_FOUND` | 404 | No existe un producto `active` o `discontinued` con ese id. También es la respuesta de un producto en `draft`. | No reintentar: el artículo no está publicado o no existe. Si tu copia lo tenía, dálo de baja. |
| `VALIDATION_ERROR` | 400 | El `productId` no es un uuid. | Corregir la entrada; reintentar igual no cambia nada. |
| `UNAUTHORIZED` | 401 | Falta el token o no es válido. | Renovar el token y reintentar una vez. |
| `FORBIDDEN` | 403 | El token es válido pero no trae el scope `product:read`, o se emitió para otra audiencia. | No reintentar: es un problema de aprovisionamiento de tu cliente. |
| — | 5xx / timeout | Fallo del servidor o de la red. | Reintentable con espera creciente; la operación es una lectura, así que repetirla no tiene efectos. |

### listProductsBatchForServices

| | |
|---|---|
| Endpoint | `GET /api/v1/services/products` |
| Acceso | `service` — scopes `product:read` |
| Idempotencia | no aplica (query) |
| Cota del lote | **100 ids** por petición |
| Caché | no se cachea: solo las fichas individuales lo están |

**Request** — parámetro de consulta `productIds`, lista de uuid. Entre **1 y 100** elementos.

```
GET /api/v1/services/products?productIds=3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9&productIds=7a4b9c10-5555-4e55-9f05-eeee0000f005
```

**Response** — una **lista** (no un sobre paginado) de la misma forma que devuelve
[`getProductForServices`](#getproductforservices), con estas tres reglas que hay que tener en cuenta al
consumirla:

| Regla | Qué significa para ti |
|---|---|
| Los ids inexistentes, los repetidos y los de productos en `draft` **se omiten** sin fallar | La respuesta puede traer **menos** elementos que la petición, y eso no es un error. Compara por `id` lo que pediste con lo que llegó; lo que falta, no está publicado |
| El orden es por `sku` ascendente, con el `id` como desempate | **No** es el orden de tu petición. Indexa el resultado por `id` en vez de emparejar por posición |
| Un lote de 100 no cuesta más por elemento que uno de 2 | La resolución es por lote: úsalo en lugar de N llamadas a `getProductForServices` |

```json
[
  {
    "id": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
    "sku": "SKU-B01",
    "name": "Laptop Pro 14",
    "slug": "laptop-pro-14",
    "price": 1299.00,
    "status": "active",
    "createdAt": "2026-05-14T09:21:07.482Z",
    "updatedAt": "2026-05-20T11:04:55.120Z",
    "brand": { "id": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001", "name": "Acme", "slug": "acme" },
    "category": { "id": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002", "name": "Laptops", "slug": "laptops" },
    "images": []
  },
  {
    "id": "7a4b9c10-5555-4e55-9f05-eeee0000f005",
    "sku": "SKU-B02",
    "name": "Teclado K1 Pro",
    "slug": "teclado-k1-pro",
    "price": 89.90,
    "status": "discontinued",
    "createdAt": "2026-04-02T08:10:00.000Z",
    "updatedAt": "2026-05-18T16:42:31.900Z",
    "brand": { "id": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001", "name": "Acme", "slug": "acme" },
    "category": { "id": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002", "name": "Laptops", "slug": "laptops" },
    "images": []
  }
]
```

| Código | HTTP | Cuándo | Acción recomendada |
|---|---|---|---|
| `EMPTY_ID_LIST` | 422 | La lista de ids llega vacía o ausente. | Corregir la entrada: no pidas un lote vacío. |
| `TOO_MANY_IDS` | 422 | La lista trae más de 100 ids. | Corregir la entrada: **parte el lote** en trozos de 100 y encadena las peticiones. |
| `VALIDATION_ERROR` | 400 | Algún elemento no es un uuid. | Corregir la entrada. |
| `UNAUTHORIZED` | 401 | Falta el token o no es válido. | Renovar el token y reintentar una vez. |
| `FORBIDDEN` | 403 | El token no trae el scope `product:read`, o se emitió para otra audiencia. | No reintentar: es aprovisionamiento de tu cliente. |
| — | 5xx / timeout | Fallo del servidor o de la red. | Reintentable con espera creciente; es una lectura. |

El contrato formal de las dos operaciones, con todos sus schemas, está en
[`openapi.yaml`](openapi.yaml) (visor: [`openapi.html`](openapi.html)).

## Eventos

El contrato formal de esta sección —canales, mensajes y schemas— está en
[`asyncapi.yaml`](asyncapi.yaml) (visor: [`asyncapi.html`](asyncapi.html)). Aquí va lo que necesitas
saber para consumirlo.

### Publicados

**Forma del mensaje.** Todo evento de esta sección viaja en la envoltura estándar de Keel. El payload
del evento es el contenido de `data`; `metadata` es la misma para todos.

```json
{
  "metadata": {
    "eventId": "9f1c3b6e-2d4a-4a91-b0f2-5c7d8e0a1b23",
    "eventType": "ProductCreated",
    "eventVersion": 1,
    "occurredAt": "2026-05-14T09:21:07.482Z",
    "source": "catalog",
    "correlationId": "1f7b0a52-33c9-4a1e-9a44-6c0f2b8d55e1",
    "traceparent": "00-4bf92f3577b34da6a3ce929d0e0e4736-00f067aa0ba902b7-01"
  },
  "data": { "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9", "sku": "SKU-M01" }
}
```

| Campo | Tipo | Descripción |
|---|---|---|
| `metadata.eventId` | uuid | Id único de esta ocurrencia. **Úsalo como clave de deduplicación**: la entrega es at-least-once y una reentrega repite el mismo `eventId`. |
| `metadata.eventType` | string | Nombre del evento (`ProductCreated`). Discriminador si el canal transporta varios tipos — y los dos canales de este servicio transportan varios. |
| `metadata.eventVersion` | int | Versión del contrato de `data`. Sube solo al romper compatibilidad. |
| `metadata.occurredAt` | timestamp | ISO-8601 UTC del instante en que ocurrió el hecho, no el del envío. Con outbox los dos pueden distar. |
| `metadata.source` | string | Servicio emisor: `catalog`. |
| `metadata.correlationId` | string \| null | Correlación de la petición que originó el hecho; propágala para conservar la traza end-to-end. `null` si no hubo contexto de petición. |
| `metadata.traceparent` | string \| null | Contexto de traza W3C del hecho. Si tu servicio tiene trazas distribuidas, continúalo al consumir para que la traza no se corte en el broker. `null` si el emisor no tiene telemetría. |
| `data` | objeto | Payload del evento; su forma depende del `eventType` (ver cada evento abajo). |

**Garantía de entrega.** La publicación es `outbox`: el evento se escribe en la misma transacción que
el cambio de estado y un relay lo publica después, así que **un cambio confirmado no se queda sin
anunciar por una caída del canal**. Lo que no garantiza es *cuándo*: puede llegar con retraso si el
broker estuvo caído. Y hay un desenlace que sí pierde el evento y conviene conocer: el relay tiene un
presupuesto de reintentos (lo fija el stack del proveedor, no el diseño), y el evento que lo agota se
da por **abandonado** — no se publica ya, y el servidor lo cuenta y lo señala para que alguien lo
reponga. Un consumidor que no tolere ese hueco debe reconciliar por el endpoint de lote
(`listProductsBatchForServices`), no asumir que el canal lo trae todo.

**Dos canales, a propósito.** `productEvents` lleva el ciclo de vida de los productos;
`taxonomyEvents`, el de marcas y categorías. Están separados para que quien solo sigue productos no
tenga que tragar el ruido de la taxonomía, que cambia mucho menos.

> **Si copias `brandName` o `categoryName` dentro de tu propia ficha de producto, tienes que escuchar
> `taxonomyEvents`.** Cuando se renombra una marca, el catálogo **no** re-emite los eventos de sus
> productos —serían miles de golpe—, así que la corrección te llega solo por `BrandUpdated` /
> `CategoryUpdated` y actualizas tus copias por `brandId` / `categoryId`. Sin esa suscripción te quedas
> con el nombre viejo indefinidamente.

**Los eventos de producto llevan la ficha completa** para que puedas actualizar tu copia sin llamar de
vuelta. Los campos de `data` son los mismos en `ProductCreated`, `ProductUpdated` y
`ProductStatusChanged` (este último añade `previousStatus`):

| Campo de `data` | Tipo | Notas |
|---|---|---|
| `productId` | uuid | requerido |
| `sku` | string | requerido |
| `name` | string | requerido |
| `slug` | string | requerido |
| `description` | string | solo en `ProductCreated` y `ProductUpdated`; **no viaja** si está vacía |
| `price` | decimal | requerido; escala 2 |
| `status` | enum | requerido; `draft` \| `active` \| `discontinued` |
| `previousStatus` | enum | **solo** en `ProductStatusChanged`: el estado del que viene |
| `brandId` / `brandName` | uuid / string | requeridos |
| `categoryId` / `categoryName` | uuid / string | requeridos |
| `primaryImageUrl` | string | URL pública de la imagen principal; **no viaja** si el producto no tiene imágenes |

### ProductCreated

| | |
|---|---|
| Canal | `productEvents` |
| Emitido por | `createProduct` |
| Garantía | outbox |

Un producto se dio de alta. **Nace en `draft`**, así que todavía no se vende: si mantienes una copia de
lo que está a la venta, este evento te dice que existe, no que se pueda comprar.

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-M01",
  "name": "Laptop Pro 14",
  "slug": "laptop-pro-14",
  "description": "Portátil de 14 pulgadas, 16 GB de RAM.",
  "price": 1299.00,
  "status": "draft",
  "brandId": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001",
  "brandName": "Acme",
  "categoryId": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002",
  "categoryName": "Laptops"
}
```

### ProductUpdated

| | |
|---|---|
| Canal | `productEvents` |
| Emitido por | `updateProduct`, `addProductImage`, `removeProductImage`, `setPrimaryProductImage`, `reorderProductImages` |
| Garantía | outbox |

Cambió la ficha comercial de un producto: su nombre, descripción, precio, marca, categoría **o su
galería de imágenes**. El payload es el **estado resultante**, no el diff: aplícalo entero sobre tu
copia en vez de intentar deducir qué cambió.

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-M01",
  "name": "Laptop Pro 14",
  "slug": "laptop-pro-14",
  "price": 1199.00,
  "status": "active",
  "brandId": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001",
  "brandName": "Acme",
  "categoryId": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002",
  "categoryName": "Laptops",
  "primaryImageUrl": "https://cdn.example.com/productImages/3d2e1f00/front.jpg"
}
```

### ProductStatusChanged

| | |
|---|---|
| Canal | `productEvents` |
| Emitido por | `publishProduct`, `unpublishProduct`, `discontinueProduct`, `reactivateProduct` |
| Garantía | outbox |

Un producto **entró o salió de la venta**. Lleva `previousStatus` para que sepas qué transición fue sin
comparar con tu copia. Es el evento que te dice si debes mostrarlo o dejar de mostrarlo:
`status: "active"` es a la venta; `draft` y `discontinued`, no.

```json
{
  "productId": "3d2e1f00-8a44-4c9b-9f01-77b6c2d4e5a9",
  "sku": "SKU-M01",
  "name": "Laptop Pro 14",
  "slug": "laptop-pro-14",
  "price": 1199.00,
  "previousStatus": "active",
  "status": "discontinued",
  "brandId": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001",
  "brandName": "Acme",
  "categoryId": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002",
  "categoryName": "Laptops",
  "primaryImageUrl": "https://cdn.example.com/productImages/3d2e1f00/front.jpg"
}
```

### ProductDeleted

| | |
|---|---|
| Canal | `productEvents` |
| Emitido por | `deleteProduct` |
| Garantía | outbox |

Se eliminó definitivamente un producto **que nunca llegó a publicarse**. Es tu **vía de baja**: sin
procesarlo, una copia local se queda rancia para siempre. No repite la ficha porque el producto ya no
existe: lleva lo justo para localizar tu copia y borrarla. Un producto que sí estuvo publicado nunca se
borra —se descontinúa—, así que este evento solo alcanza a borradores.

```json
{
  "productId": "5c9d8e70-4444-4d44-9a04-dddd0000e004",
  "sku": "SKU-030",
  "name": "Monitor de prueba"
}
```

### BrandCreated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Emitido por | `createBrand` |
| Garantía | outbox |

```json
{
  "brandId": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001",
  "name": "Acme",
  "slug": "acme",
  "description": "Fabricante de periféricos."
}
```

### BrandUpdated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Emitido por | `updateBrand` |
| Garantía | outbox |

Cambió el nombre o la reseña de una marca; **con el nombre cambia también su slug público**. Si copias
`brandName`, este es el evento que te pone al día: actualiza por `brandId` todas tus copias de productos
de esa marca.

```json
{
  "brandId": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001",
  "name": "Acme Corp",
  "slug": "acme-corp"
}
```

### BrandDeleted

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Emitido por | `deleteBrand` |
| Garantía | outbox |

Se eliminó una marca **que ningún producto referenciaba**, así que no puede dejar productos tuyos
huérfanos.

```json
{ "brandId": "b1f0c2a4-1111-4a11-9c01-aaaa0000b001", "name": "Acme Corp" }
```

### CategoryCreated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Emitido por | `createCategory` |
| Garantía | outbox |

```json
{
  "categoryId": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002",
  "name": "Laptops",
  "slug": "laptops",
  "description": "Portátiles de todas las gamas."
}
```

### CategoryUpdated

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Emitido por | `updateCategory` |
| Garantía | outbox |

Cambió el nombre o el texto de cabecera de una categoría; con el nombre cambia también su slug público.
Mismo trato que `BrandUpdated` si copias `categoryName`.

```json
{
  "categoryId": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002",
  "name": "Portátiles",
  "slug": "portatiles"
}
```

### CategoryDeleted

| | |
|---|---|
| Canal | `taxonomyEvents` |
| Emitido por | `deleteCategory` |
| Garantía | outbox |

Se eliminó una categoría que ningún producto referenciaba.

```json
{ "categoryId": "c2e1a3b5-2222-4b22-9d02-bbbb0000c002", "name": "Portátiles" }
```

### Suscripciones

**Este servicio no se suscribe a ningún evento.** No hay nada que puedas publicar para encargarle
trabajo: el catálogo solo se modifica por su API de gestión, con token de usuario y los permisos del
back-office. Si necesitas que el catálogo haga algo —crear un producto, cambiar un precio—, eso no es
una integración servidor-a-servidor: es una acción de gestión, y la hace una persona o un sistema con
credencial de usuario contra `/api/v1/management/**`, fuera del alcance de este documento.

Que no consuma nada tiene una consecuencia útil para ti: `catalog` **se despliega y se prueba solo**, no
arrastra a nadie en su arranque, y puede estar disponible antes que cualquier servicio que lo consuma.
