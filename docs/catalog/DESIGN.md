# catalog — Documento de diseño

> specs/catalog v0.1.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`catalog` es la **fuente de verdad del catálogo comercial** de una tienda: productos con sus imágenes, las marcas bajo las que se venden y las categorías por las que se navegan. Sirve a tres públicos con tres superficies separadas:

- **El consumidor final** ve el escaparate por endpoints públicos sin autenticación. Solo ve productos publicados y puede filtrarlos por categoría, marca, nombre y rango de precio.
- **El equipo de la tienda** gestiona el catálogo por endpoints protegidos por rol, con rastro de quién creó y quién modificó cada producto.
- **Otros servidores** (`orders`, `cart`) resuelven productos por id o por lote con credencial de máquina, y se mantienen al día con los eventos que el catálogo publica en cada alta y cada cambio.

**Fuera de alcance, a propósito**: stock e inventario, variantes (talla, color), descuentos y promociones, multi-divisa y multi-tienda. El catálogo no consume eventos de nadie ni llama a ningún servidor: es una fuente pura.

## 2. Modelo de dominio

### Value types

| Tipo | Forma | Significado |
|---|---|---|
| `SKU` | string `^[A-Z0-9][A-Z0-9_-]{1,31}$` | Referencia comercial única del producto, la que comparten el catálogo y los servidores que lo consumen. La entrada la admite en minúsculas y se normaliza. |
| `Money` | `{ amount: decimal ≥ 0, escala 2, scalePolicy: reject; currency: ISO 4217 }` | Importe con su moneda. Un importe con más de 2 decimales se **rechaza**, nunca se redondea. |
| `ProductStatus` | enum `draft`, `active`, `retired` | Situación comercial del producto. |
| `ProductImageRef` | `{ imageId, file, altText, position, main }` | Una imagen tal como viaja en los eventos. |

### Entidades y agregados

| Agregado | Raíz | Internas | Por qué es su propia frontera |
|---|---|---|---|
| `Product` | `Product` | `ProductImage` | La imagen no existe fuera de su producto y ninguna otra entidad la referencia. «Una sola principal», «máximo 10» y «posiciones sin huecos» son invariantes del conjunto. |
| `Brand` | `Brand` | — | El producto la referencia por id; renombrar una marca no bloquea a sus miles de productos. |
| `Category` | `Category` | — | Ídem. |

- **Product**: `id` (*generated*), `sku` (único, exacto tras normalizar), `name` (≤160, se compara sin mayúsculas ni acentos), `description` (≤4000), `price: Money`, `status` (default `draft`), `brand` y `category` (obligatorias, many-to-one), `images` (one-to-many, ordenadas por `position`), y los cuatro campos de auditoría `createdAt`, `updatedAt`, `createdBy`, `updatedBy` (*generated*).
- **ProductImage**: `file` (bucket `productImages`), `altText` (≤160), `position` (0–9), `main`.
- **Brand** / **Category**: `name` (≤80, único sin mayúsculas ni acentos: `Acmé` = `ACME`), `description` (≤2000), `active` (default `true`). **No** llevan campos de auditoría en el contrato (ver §6).
- Ningún campo es `sensitive` ni `computed`.

### Ciclo de vida de `Product`

```
draft ──publishProduct──▶ active
  ▲                          │
  └─────unpublishProduct─────┘
draft ─┐
       ├──retireProduct──▶ retired   (terminal)
active ┘
```

Un producto nace en `draft`, invisible al público. Se publica cuando está listo, se puede devolver a `draft` para corregirlo y se retira desde cualquiera de los dos. `retired` es terminal: el producto conserva todos sus datos y sigue siendo resoluble por los servidores que lo consumen, pero no vuelve a venderse. Cualquier otra transición responde `409 INVALID_STATE_TRANSITION`.

## 3. Invariantes y reglas clave

**Del dominio (`Product`)**
- Un producto `active` tiene al menos una imagen y su `price.amount` es mayor que cero.
- Un producto con imágenes tiene **exactamente una** marcada como `main`; como máximo 10; dos imágenes no comparten `position`. Las dos unicidades las respalda además el almacén (índices únicos, uno condicionado a `main = true`).
- El `price.currency` es la **moneda del catálogo**, un parámetro de despliegue único para todo el servicio.

**De los casos de uso**
- El `sku` no se modifica nunca: es la referencia estable con la que otros servidores conocen al producto.
- **PATCH**: un campo ausente conserva su valor; un opcional enviado a `null` se vacía; un obligatorio a `null` es `400`.
- **Sin cambio, sin evento**: un PATCH, `activate*` o `deactivate*` que no cambia nada responde `200`, no publica y no toca `updatedAt`/`updatedBy`.
- Asignar una marca o categoría **inactiva** a un producto falla (`422 BRAND_INACTIVE` / `CATEGORY_INACTIVE`), pero solo cuando la petición la **cambia**: reenviar la que ya tenía no falla.
- Desactivar una marca o categoría **no** oculta sus productos: los quita del menú de filtros y bloquea asignaciones nuevas, nada más.
- Borrar una marca o categoría solo se puede sin productos asociados. Ante una carrera con un alta, gana una de las dos operaciones y la otra falla con su error; nunca queda un producto huérfano.
- Imágenes: la primera es siempre principal; al marcar otra como principal, la anterior deja de serlo; reordenar desplaza las demás sin dejar huecos; borrar la principal promueve a la que queda en `position 0`; no se puede dejar sin imágenes a un producto `active`.
- Binarios: se suben **antes** de confirmar la imagen y se borran **después** de confirmar su baja. Un fallo a mitad deja como mucho un archivo huérfano, nunca una ficha que apunte a un archivo inexistente.
- El escaparate responde `404` a un producto que no está `active`, igual que a uno que no existe. Un filtro por una categoría o marca inexistente devuelve una página vacía, no un error.

## 4. Qué hace

API REST bajo `/api/v1`. Paginación offset: 24 por página, tope 100.

**Gestión — productos** (protegida)
| Operación | Endpoint | Notas |
|---|---|---|
| `createProduct` | `POST /products` → 201 | Idempotency-Key; emite `ProductCreated` |
| `updateProduct` | `PATCH /products/{id}` | Idempotency-Key; emite `ProductUpdated` |
| `publishProduct` / `unpublishProduct` | `POST /products/{id}/publish` · `/unpublish` | transición; emite `ProductStatusChanged` |
| `retireProduct` | `POST /products/{id}/retire` | solo `catalog-admin`; `reason` opcional viaja en el evento |
| `getProduct`, `listProducts` | `GET /products/{id}` · `GET /products` | cualquier estado; filtros de escaparate + `status`; orden `updatedAt desc` |
| `addProductImage` | `POST /products/{productId}/images` → 200 | multipart; devuelve la ficha del producto |
| `updateProductImage` | `PATCH /products/{productId}/images/{imageId}` | texto alternativo, posición, principal |
| `removeProductImage` | `DELETE /products/{productId}/images/{imageId}` → 200 | devuelve la ficha: la baja recoloca posiciones y puede cambiar la principal |

**Gestión — marcas y categorías** (protegida): `create*` (201), `update*` (PATCH), `activate*` / `deactivate*`, `delete*` (204, solo `catalog-admin`), `get*`, `list*` (filtros `active` y `name`, orden `name asc`). Todas las mutaciones publican su evento en `taxonomyEvents`.

**Escaparate** (público, sin autenticación): `listPublicProducts` (`GET /public/products`, filtros `categoryId`, `brandId`, `name` por contenido sin mayúsculas ni acentos, `minPrice`/`maxPrice` inclusivos, orden `name asc`), `getPublicProduct`, `listPublicCategories` y `listPublicBrands` (solo activas, sin paginar: son los menús de filtro).

Las 13 mutaciones con efecto hacia fuera exigen `Idempotency-Key` (24 h). Ninguna query lleva caché.

### Superficie servidor-a-servidor

| Operación | Endpoint | Consumidores |
|---|---|---|
| `getProductForServices` | `GET /internal/products/{id}` | `orders`, `cart` (scope `product:read`) |
| `listProductsBatchForServices` | `POST /internal/products/batch` (1–100 ids) | ídem |

Devuelven el producto **en cualquier estado, incluido `retired`**: un pedido de hace meses tiene que poder resolver su línea. El `status` viaja en la respuesta y el consumidor decide. En el lote, los ids inexistentes se omiten sin error y los repetidos aparecen una vez. Van por `POST` porque 100 uuid no caben con seguridad en una URL. Ninguna de las dos devuelve campos de auditoría. El contrato completo, para quien integra, lo produce `/keel-integrate` en `INTEGRATION.md`.

## 5. Fronteras e integraciones

- **Eventos** (`messaging`, fiabilidad **outbox**): canal `productEvents` con `ProductCreated` y `ProductUpdated` (la **ficha completa**, con marca y categoría resueltas por id y nombre, y las imágenes con la clave del objeto) y `ProductStatusChanged` (`previousStatus`, `status`, `reason`). Canal `taxonomyEvents` con `Brand*` y `Category*` `Created`/`Updated`/`Deleted`. Todo en la envoltura Keel (`metadata.eventId` para deduplicar, `occurredAt` para ordenar).
- **Persistencia**: relacional, frontera **por agregado**, bloqueo optimista en las tres raíces (`409 CONCURRENT_MODIFICATION`), auditoría `declared` (tiempos y autoría) solo en `Product`. Claves naturales: `sku`, `Brand.name`, `Category.name`.
- **Almacenamiento**: bucket `productImages`, **público**, JPEG/PNG/WebP, 5 MB. En las respuestas HTTP la imagen viaja como URL absoluta; en los eventos, como clave.
- **Seguridad**: OIDC; roles `catalog-manager` (gestión diaria) y `catalog-admin` (además, retirar y borrar). Permisos `product:read|write|publish|retire` y `taxonomy:read|write|delete`. M2M por `client-credentials` con validación de audiencia (`catalog`). CORS declarado para la tienda y el back-office (orígenes por despliegue).

## 6. Decisiones de diseño (qué / por qué)

| Decisión | Qué se eligió | Por qué | Alternativa descartada |
|---|---|---|---|
| Imágenes | Entidad hija `ProductImage` dentro del agregado `Product` | La portada es una decisión, no la primera de una lista; hay que borrar, reordenar y marcar una imagen concreta | Lista de archivos sin identidad (cualquier cambio reescribe la lista) |
| Categorías | Planas | Filtro público por igualdad; subcategorías son una evolución barata | Árbol (ciclos, profundidad, filtro recursivo) |
| Lifecycle | `draft ↔ active`, ambos → `retired`, terminal | Despublicar para corregir; retirar sin borrar protege el histórico de los consumidores | Solo hacia delante; o `retired` reversible (resucitar lo que un consumidor dio por muerto) |
| Borrado de producto | No existe: solo retirada | Los pedidos históricos y las réplicas siguen resolviendo el id | `deleteProduct` para errores de carga |
| Identidad | `SKU` único, aceptado en minúsculas y normalizado | Es la referencia que comparten el catálogo, el ERP y los consumidores; el duplicado da un error claro | Solo uuid; o SKU + slug público |
| Marcas y categorías | Campo `active`; borrado solo sin productos | Desactivar es la salida real sin perder histórico; borrar con productos dejaría huérfanos | Cascada (arrastra el catálogo); referencia opcional |
| Desactivar | No oculta los productos | Es una decisión de escaparate; la visibilidad depende solo del `status` del producto | Ocultar en cascada (desaparecen productos sin que ningún evento lo explique) |
| Moneda | `Money` con moneda + moneda única de catálogo, `422 CURRENCY_NOT_SUPPORTED` | El importe nunca viaja desnudo, y el filtro de precio es correcto por construcción | Importe sin moneda; multi-moneda en el filtro |
| Mín. 1 imagen / máx. 10 | Invariantes de publicación y de tamaño | El escaparate nunca muestra un hueco; la respuesta del lote M2M tiene techo | Imagen opcional; sin límite |
| §3.4 Superficie M2M | Operaciones propias (`…ForServices`), cualquier estado, lote ≤ 100 | Los contratos de máquina y de pantalla evolucionan distinto; el consumidor necesita resolver productos retirados | Reutilizar los endpoints públicos (`audience: both`); solo `active` |
| §3.3 Caché | Ninguna, en ninguna query | El diseñador prefiere frescura inmediata en precios y publicación; si hace falta, CDN por delante sin tocar el contrato | Caché de 5 min en la ficha (precio rancio); 60 s en menús |
| Ficha pública | Marca y categoría embebidas | La tienda pinta la ficha en una llamada | Solo ids (segunda llamada o copia propia) |
| §3.2 Idempotencia | `Idempotency-Key` **obligatoria**, 24 h, en las 13 mutaciones con evento | Un reintento no publica un cambio fantasma ni duplica una foto; opcional sería una garantía que depende de que cada cliente se acuerde | Solo en altas; opcional; ninguna |
| §3.8 Paginación | 24 / 100; menús públicos sin paginar | Rejilla de tienda; los menús tienen unas decenas de entradas | 20/100; 50/200 |
| §3.9 Concurrencia | Bloqueo optimista en las tres raíces | Dos gestores sobre la misma ficha es rutina; «último gana» borra en silencio una corrección de precio | Último gana; solo en `Product` |
| §3.9b Auditoría | `declared` (tiempos y autoría), **solo en `Product`**, visible solo en gestión | El DSL no puede recortar campos dentro de una marca embebida: auditar marcas en el contrato expondría ids de empleados en la ficha pública | Auditoría en las tres (fuga al público); solo en BD; ids en vez de embebidos |
| Eventos | `Created` + `Updated` + `StatusChanged`, ficha completa, más eventos de marca y categoría | El consumidor distingue un cambio de precio de una baja sin comparar campos, actualiza su copia sin llamar de vuelta, y corrige un renombrado sin reindexar | Solo Created/Updated; aviso con id; reemitir todos los productos al renombrar |
| §3.1 Fiabilidad | `outbox` | Un despliegue del broker no puede dejar a un replicante con precios viejos para siempre | `best-effort` |
| §3.7 Frontera | `per-aggregate` | Ninguna operación cruza agregados, así que no se acepta inconsistencia nueva; el evento confirma con su agregado | `per-operation` |
| §3.10 Bucket | `public`, JPEG/PNG/WebP, 5 MB | Fotos de escaparate, cacheables en CDN; 5 MB corta originales de cámara sin procesar | `private` con URL firmada (10 firmas por visita) |
| Roles | `catalog-manager` / `catalog-admin` | El corte está en lo irreversible: retirar y borrar | Un solo rol; tres con lectura |
| M2M | Consumidores `orders` y `cart`, identidad del llamante no participa | El catálogo no tiene datos por consumidor | Acotar por consumidor (multi-inquilino) |
| Carreras de borrado | Nunca un producto huérfano | La referencia es obligatoria y el almacén la hace cumplir | Aceptar la ventana |
| Nulos en contrato | `include` | El consumidor ve la forma completa del recurso | `omit` |
| Importe en JSON | Número | Natural para un front; escala 2 exacta | Cadena decimal |

Los códigos canónicos `IDEMPOTENCY_KEY_IN_PROGRESS`, `IDEMPOTENCY_KEY_REUSED` y `CONCURRENT_MODIFICATION` se aceptaron como contrato público (en `decisions.yaml`), por homogeneidad con cualquier otro servicio Keel.

## 7. Ficha de reutilización: adoptar, derivar o evolucionar

### Contrato estable vs adaptable

**Estable** (cambiarlo rompe a alguien): los 25 códigos de error y sus status; los nueve eventos, sus canales y sus payloads; las rutas y los status de los 30 endpoints, en especial los dos de `/internal`; los roles, los permisos y el scope `product:read`; la forma de `Product` en las respuestas (con marca y categoría embebidas y sin auditoría fuera de gestión); la convención `nulls: include`.

**Adaptable** sin romper a nadie: los límites del bucket (formatos, tamaño), el máximo de imágenes, el patrón del `SKU` (si se relaja), el `ttlSeconds` de la idempotencia, los tamaños de página, los índices y las reglas internas de reordenación. El diseño versiona con semver del contrato (`docs/methodology.md`): una ampliación compatible es minor; retirar o cambiar cualquier cosa de la lista estable es major.

### Puntos de extensión típicos

- **Categorías jerárquicas**: una relación `parent` sobre `Category` y un filtro «con descendientes».
- **Variantes**: una entidad hija `ProductVariant` con su propio SKU dentro del agregado `Product`.
- **Estados nuevos** del lifecycle (`discontinued`, `preorder`) entre `active` y `retired`.
- **Caché** en las queries públicas, que ahora exigiría listar `Product*`, `Brand*` y `Category*` en `invalidatedBy` (los eventos ya existen).
- **Logo de marca**: un campo `file` en `Brand` y un bucket nuevo.
- Piezas reutilizables en otros servicios: `Money` con moneda única de catálogo; el patrón de hija ordenada con «una principal» (índices únicos condicionados); la pareja de operaciones M2M por id y por lote.

### Supuestos y limitaciones

- **Volumen**: pensado para catálogos de hasta unos **50.000 productos**. Por eso el filtro por nombre es «contiene» sin índice de texto, los menús de filtro no paginan y no hay caché. Por encima de ese volumen, la búsqueda debería ser un servicio aparte alimentado por `productEvents`.
- **Una tienda por despliegue**: sin inquilinos, roles globales y una sola moneda. Multi-tienda es una derivación: un campo de tienda en cada entidad y `authentication.scoping`.
- **Moneda única**: la fija el despliegue; el servicio no convierte ni mantiene precios por divisa.
- **Auditoría parcial**: marcas y categorías no registran quién las tocó (ver §6).
- **Binarios huérfanos**: un fallo a mitad de subida o de borrado puede dejar archivos sin referencia; no hay barrido que los limpie.
- **Autorización M2M**: el `403` por scope insuficiente no tiene escenario, porque todos los consumidores declarados tienen el único scope que existe.
- Fuera de alcance: stock, variantes, promociones, búsqueda avanzada (facetas, relevancia).

### Cómo reutilizarlo

`keel describe catalog` da el resumen mecánico (identidad, estado, capas, contenido).

- **Adoptarlo tal cual** —lo esperado si tu tienda encaja en los supuestos de arriba—: `keel registry get catalog`. Llega con sus derivados al día y se va directo a generar (`keel-<tech> build specs/catalog`).
- **Derivarlo** si necesitas tocar algo de la lista estable o salir de los supuestos (variantes, multi-tienda, jerarquía): `keel new <nuevo> --from registry:catalog`. `/keel-design` arranca en modo derivación y entrevista solo lo que cambia. No derives para acabar usándolo sin cambios: tendrías que regenerar a mano todos los derivados que la adopción ya trae.
