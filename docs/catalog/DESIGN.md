# catalog — Documento de diseño

> specs/catalog v0.1.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`catalog` gestiona el catálogo comercial de una tienda: **productos** con sus **imágenes**, **marcas** y
**categorías**. Sirve a tres públicos con superficies separadas:

- **Gestores de la tienda** (panel de gestión, protegido por rol): dan de alta y mantienen productos,
  imágenes, marcas y categorías, y deciden qué se publica. Toda escritura deja rastro de **quién** y
  **cuándo** (`createdBy`/`updatedBy`, `createdAt`/`updatedAt`) en el contrato de gestión.
- **Consumidores de la tienda** (anónimos): ven solo lo publicado y filtran por categoría, marca,
  nombre y rango de precio.
- **Otros servidores** (pedidos, carrito): resuelven productos por id o por lote con credencial de
  máquina, y mantienen su propia copia a partir de los eventos `ProductCreated` / `ProductUpdated`.

Fuera de alcance, a propósito: stock, ofertas y reglas de precio (otros servicios), multi-tienda,
traducciones de contenido y búsqueda de texto completo.

## 2. Modelo de dominio

**Value types**

| Tipo | Forma | Significado |
|---|---|---|
| `SKU` | string `^[A-Z0-9][A-Z0-9_-]{1,31}$` | Referencia comercial única compartida con otros servidores; se normaliza a mayúsculas |
| `Money` | `{ amount: decimal(escala 2, min 0, se rechaza escala mayor), currency: ISO 4217 }` | Importe en la moneda del catálogo; sin conversión de divisas |
| `ProductStatus` | `draft \| active \| retired` | Situación comercial |
| `NamedRef` | `{ id, name }` | Marca o categoría tal como viaja en un evento (foto del instante) |
| `ProductImageRef` | `{ imageId, file, altText, position, main }` | Imagen tal como viaja en un evento |

**Entidades y agregados**

| Agregado | Raíz | Internas | Campos clave |
|---|---|---|---|
| `Product` | `Product` | `ProductImage` | `sku` (único), `name`, `description`, `price: Money`, `status`, `firstPublishedAt`, relaciones `brand` y `category` (por id, many-to-one, obligatorias), `images` (1..N, hasta 10) |
| `Brand` | `Brand` | — | `name` (único sin distinguir mayúsculas ni acentos), `description`, `active` |
| `Category` | `Category` | — | ídem que `Brand`; categorías **planas**, sin jerarquía |

`ProductImage`: `file` (bucket `productImages`), `altText`, `position` (0..9), `main`.

Campos que nunca envía el cliente: `id`, `lockVersion`, `createdAt`, `updatedAt`, `createdBy`,
`updatedBy` (`generated`, los pone la infraestructura) y `firstPublishedAt` (lo escribe
`publishProduct` en la primera publicación). No hay campos `sensitive` ni `computed`.

**Ciclo de vida de `Product`**

```
draft ──publish──▶ active ──retire──▶ retired (terminal)
  ▲                  │
  └────unpublish─────┘
draft ──retire──▶ retired
```

Marcas y categorías no tienen máquina de estados: se **desactivan** y **reactivan** (`active`), y
nunca se borran.

## 3. Invariantes y reglas clave

- Un producto `active` tiene al menos una imagen y `price.amount > 0`.
- Con imágenes, exactamente una es la principal (`main`); como máximo 10; sus posiciones son
  exactamente `0..n-1`, sin huecos ni repeticiones. La primera imagen subida es principal siempre;
  al quitar la principal, pasa a serlo la que queda en posición 0.
- Un producto `retired` es inmutable (datos e imágenes); sigue siendo resoluble por los servidores.
- El `sku` solo se corrige mientras el producto **nunca** se publicó (`firstPublishedAt` null);
  después es inmutable.
- `price.currency` es siempre el parámetro de despliegue `currency` (moneda de 2 decimales, fijada en
  el primer despliegue y que no cambia nunca).
- Una marca o categoría inactiva no admite productos **nuevos** ni reasignaciones hacia ella, ni sale
  como filtro público; lo que ya está asignado sigue su vida (incluso se puede publicar).
- La `lockVersion` del producto sube con cualquier cambio del agregado, también de sus imágenes; un
  cambio nulo no cambia nada, no sube la versión ni publica evento.

## 4. Qué hace

**Gestión (usuarios con rol, bajo `/api/v1/admin`)**

| Operación | Qué hace | Notas |
|---|---|---|
| `createBrand` / `createCategory` | Alta, activa desde el inicio | `Idempotency-Key` obligatoria (24 h) |
| `updateBrand` / `updateCategory` | Edita nombre y descripción | `PATCH` con `version` obligatoria |
| `deactivateBrand` / `reactivateBrand`, `deactivateCategory` / `reactivateCategory` | Baja y alta lógicas | Idempotentes por naturaleza |
| `getBrand`, `listBrands`, `getCategory`, `listCategories` | Lectura de gestión | Listados paginados, orden `name` + `id` |
| `createProduct` | Alta en `draft`, sin imágenes | `Idempotency-Key` obligatoria; emite `ProductCreated` |
| `updateProduct` | Datos comerciales (y sku si nunca se publicó) | `PATCH` con `version`; emite `ProductUpdated` |
| `publishProduct` / `unpublishProduct` / `retireProduct` | Transiciones del ciclo de vida | Transiciones irrepetibles; emiten `ProductUpdated` |
| `getProduct`, `searchProducts` | Lectura de gestión en cualquier estado | Filtros de la tienda + `status` y prefijo de `sku` |
| `addProductImage` | Sube una imagen (multipart) | `Idempotency-Key` obligatoria; tamaño y firma se comprueban primero |
| `updateProductImage` | Texto alternativo, posición, portada | `PATCH` con `version` |
| `removeProductImage` | Quita una imagen y renumera | No deja sin imagen a un producto publicado |

**Tienda (pública, sin credencial)**

| Operación | Qué hace |
|---|---|
| `searchPublishedProducts` — `GET /products` | Solo `active`; filtros acumulativos `categoryId`, `brandId`, `name` (contiene, sin mayúsculas ni acentos), `minPrice`/`maxPrice` (inclusivos); paginado, orden `name` + `id` |
| `getPublishedProduct` — `GET /products/{id}` | Ficha; `draft` y `retired` responden como inexistentes |
| `listPublicBrands`, `listPublicCategories` | Las activas, sin paginar, para pintar los filtros |

Las respuestas de la tienda traen marca y categoría **anidadas** y **no** exponen `lockVersion` ni la
auditoría.

**Superficie servidor-a-servidor (M2M, `audience: services`)**

| Operación | Contrato |
|---|---|
| `getProductForServices` — `GET /internal/products/{id}` | El producto en **cualquier** estado, con marca y categoría anidadas y `lockVersion`; sin auditoría. `404 PRODUCT_NOT_FOUND` |
| `getProductsBatch` — `POST /internal/products/batch` `{ ids }` | 1..100 ids; devuelve los que existen, en cualquier estado, ordenados por `sku`; omite los inexistentes y deduplica |

Autenticación por client-credentials con audiencia `catalog` y scope `product:read-internal`; clientes
declarados: `orders` y `cart`. Ninguna query tiene caché.

## 5. Fronteras e integraciones

- **Eventos** (canal lógico `productEvents`, envoltura Keel, **outbox**): `ProductCreated` y
  `ProductUpdated`, ambos con la **foto completa** del producto (`productId, sku, name, description,
  price, status, brand{id,name}, category{id,name}, images[], version`). Todo cambio —datos, estado,
  imágenes— es un `ProductUpdated`; la retirada llega como `status: retired`, así que la copia del
  consumidor nunca pierde un producto. `version` es la `lockVersion`: el consumidor descarta lo que le
  llegue con versión menor o igual a la que ya aplicó. En los eventos `file` viaja como **key**; la URL
  es la base pública del bucket más la key (o se resuelve por el endpoint M2M).
- **Persistencia**: modelo relacional, frontera transaccional **por agregado** (cada operación
  escribe un solo agregado más su fila de outbox), bloqueo optimista con `lockVersion` declarada en
  las tres raíces, auditoría de tiempos y autoría **declarada** (parte del contrato de gestión).
- **Archivos**: bucket lógico `productImages`, **público**, jpeg/png/webp, 5 MB por imagen,
  comprobación del formato por la firma del contenido.
- **Seguridad**: OIDC. Roles `catalog-editor` (lee todo; crea, edita, publica y despublica productos
  e imágenes) y `catalog-admin` (además retira productos y gestiona marcas y categorías). CORS
  declarado para la tienda y el panel en navegador (cabeceras `Authorization`, `Content-Type`,
  `Idempotency-Key`; expone `X-Correlation-Id` y `Location`).
- **Parámetro de despliegue**: `currency` (ISO 4217, obligatorio en producción, `EUR` en pruebas).

No consume eventos ni llama a otros servidores.

## 6. Decisiones de diseño (qué / por qué)

**Dominio**

- **Imagen como entidad hija del agregado `Product`**, no como lista de valores: cada imagen tiene
  identidad (se reordena, se marca como portada, se quita por id) y cambia siempre junto a su
  producto. Marca y categoría son agregados propios porque viven independientes del producto y se
  referencian por id.
- **`active → draft` existe** para corregir una ficha sacándola un momento de la tienda sin perder el
  producto: retirarlo es terminal y quemaría el sku y las referencias de pedidos.
- **Marcas y categorías se desactivan, nunca se borran**: tienen productos (y pedidos ajenos)
  apuntándolas. Desactivar solo corta lo nuevo.
- **Categorías planas**: es lo que pide el negocio; una jerarquía añadiría reglas de ciclos y de
  herencia en los filtros que nadie necesita todavía.
- **Moneda única del catálogo** como parámetro inmutable de 2 decimales: el filtro por rango de precio
  compara importes homogéneos, y una moneda de otra escala (JPY, KWD) no encaja con `Money`.
- **Sku corregible solo antes de la primera publicación** (`firstPublishedAt`): una errata no queda
  ocupada para siempre, pero un sku que ya vieron otros servidores no cambia.
- **Publicar exige al menos una imagen y precio > 0**: la tienda nunca muestra fichas sin foto ni
  gratis por error. Un mismo código, `PRODUCT_PRICE_NOT_POSITIVE` (409), protege la invariante al
  publicar y al editar un producto publicado.

**Estructurales** (de `specs/catalog/decisions.yaml`)

| § | Decisión | Elegido | Descartado | Por qué |
|---|---|---|---|---|
| 3.1 | Fiabilidad de publicación | `outbox` | best-effort | Un evento perdido deja la copia del consumidor con precio o estado viejo sin que nadie lo vea |
| 3.2 | Idempotencia de las cuatro altas | `Idempotency-Key` obligatoria, 24 h | sin clave (apoyarse en la unicidad) | El reintento del panel tras un timeout recibe la respuesta original, no un 409 engañoso ni una imagen duplicada ni un segundo `ProductCreated` |
| 3.2 | Resto de commands | sin clave | clave en todos | Ninguno duplica efecto (transiciones irrepetibles, PATCH con `version`, (des)activar absoluto); se acepta un 409/404 engañoso en el reintento de algo ya aplicado |
| 3.3 | Caché | ninguna | caché de 60 s en la ficha pública, 5 min en las listas de filtros | Un cambio de precio o una retirada se ven al instante; marcas y categorías no publican eventos que invaliden. Una CDN delante no toca el contrato |
| 3.4 | Superficie M2M | operaciones propias, `audience: services` | `audience: both` sobre la ficha pública; solo eventos | La ficha pública oculta `draft`/`retired`, y pedidos necesita resolver productos retirados; los dos contratos evolucionan por separado |
| 3.7 | Frontera transaccional | `per-aggregate` | per-operation | Ninguna operación escribe dos agregados. Aceptado: un alta concurrente con la desactivación de su marca puede dejar el producto creado |
| 3.8 | Paginación | offset, 20 por defecto, tope 100 | sin paginar | Colecciones sin cota; las listas públicas de filtros no se paginan (decenas o pocos cientos) |
| 3.9 | Concurrencia | bloqueo optimista `declared`, `version` obligatoria en los PATCH | último gana | Nadie pisa en silencio un precio ajeno, tampoco desde una pantalla vieja; `lockVersion` viaja como `version` en los eventos |
| 3.9b | Auditoría | tiempos y autoría `declared` | solo en la base; ninguna | Requisito explícito: el panel muestra quién y cuándo. No sale a la tienda, al contrato M2M ni a los eventos |
| 3.10 | Visibilidad de `productImages` | `public` | privada con URL firmada | Material de catálogo pensado para verse y cachear; se acepta que la imagen de un borrador sea accesible por su URL (claves no secuenciales) |

**Otras decisiones con rationale**

- **Evento con foto completa** (`ProductCreated` + `ProductUpdated`) en vez de un evento por intención:
  el consumidor solo hace upsert y no puede quedarse desalineado por no suscribirse a uno de ellos.
- **Marca y categoría con id y nombre en el evento**: el consumidor pinta sin llamar a nadie. Se
  acepta que un renombrado no publique nada y que las copias guarden el nombre viejo hasta el
  siguiente cambio de cada producto; el dato fresco está en el endpoint M2M.
- **Lote por `POST`**: 100 uuids son unos 3,7 KB de URL, y hay proxies que cortan antes. Orden por
  `sku` porque «el orden de la petición» no es expresable; el consumidor casa por id.
- **Sin limpieza de binarios huérfanos**: la baja ya intenta borrar el binario y el contenido no es
  personal; se acepta el coste de almacenamiento.
- **Sin índice único `[productId, position]`**: la renumeración desplaza posiciones dentro de una
  transacción; la invariante la sostiene el bloqueo optimista del agregado.
- **Roles**: `catalog-editor` publica y despublica (es reversible y es su día a día); solo
  `catalog-admin` retira (terminal) y gestiona marcas y categorías (afectan a todo el catálogo).
- **Errores canónicos aceptados** como contrato: `IDEMPOTENCY_KEY_IN_PROGRESS`,
  `IDEMPOTENCY_KEY_REUSED`, `CONCURRENT_MODIFICATION`, `INVALID_STATE_TRANSITION`.
- **Identidad del llamante M2M no participa**: sin inquilinos, `orders` y `cart` ven lo mismo; la
  credencial decide si se puede preguntar, no qué se ve.

## 7. Ficha de reutilización

### Contrato estable vs adaptable

**Estable** (cambiarlo rompe a alguien): los códigos de error (`PRODUCT_NOT_FOUND`,
`PRODUCT_SKU_ALREADY_EXISTS`, `PRODUCT_SKU_IMMUTABLE`, `PRODUCT_RETIRED`, `PRODUCT_HAS_NO_IMAGES`,
`PRODUCT_PRICE_NOT_POSITIVE`, `IMAGE_LIMIT_REACHED`, `IMAGE_NOT_FOUND`, `MAIN_IMAGE_REQUIRED`,
`LAST_IMAGE_OF_PUBLISHED_PRODUCT`, `BRAND_*`, `CATEGORY_*`, `CURRENCY_NOT_SUPPORTED`,
`INVALID_PRICE_RANGE`, `IDEMPOTENCY_KEY_REQUIRED` y los canónicos), los eventos `ProductCreated` /
`ProductUpdated` y su payload, las rutas publicadas —sobre todo las M2M—, los roles, los permisos y
los scopes.

**Adaptable** sin romper a nadie: los límites del bucket (tamaño, formatos), el tope de 10 imágenes,
el tamaño de página, las reglas internas de reordenación y los índices. El spec versiona por semver
del contrato (`docs/methodology.md`): patch si no cambia nada observable, minor si añade, major si
rompe.

### Puntos de extensión típicos

- Estados nuevos en `ProductStatus` (p. ej. `scheduled`) insertando aristas en el lifecycle.
- Jerarquía de categorías como evolución de `Category` (relación a sí misma).
- Eventos de marca y categoría (`BrandUpdated`, `CategoryUpdated`) si los consumidores necesitan
  nombres frescos, y con ellos una caché invalidable.
- Nuevos `serviceClients` para más consumidores M2M (evolución menor).
- Reutilizables en otros servicios: `Money` con moneda de despliegue, el patrón «foto completa +
  versión» en eventos, y el par operación pública / operación M2M separadas.

### Supuestos y limitaciones

- **Una sola tienda**: un catálogo por despliegue, sin multi-tenant.
- **Volumen de decenas de miles de productos**: la búsqueda por nombre (`contains`) sobre el almacén
  basta; no hay motor de texto completo.
- **Un solo idioma**: nombres y descripciones sin traducciones.
- **Moneda única** de 2 decimales, inmutable tras el primer despliegue.
- **Stock y precios promocionales fuera**: los gestionan otros servicios.
- **Consistencia eventual** en las copias de los consumidores (outbox + versión), y nombres de marca o
  categoría en las copias que pueden quedar viejos tras un renombrado.

### Cómo reutilizarlo

`keel describe catalog` da el resumen mecánico. Si el diseño sirve **tal cual** (una tienda, una
moneda, este modelo de roles), se **adopta** con `keel registry get catalog`: llega con sus derivados
al día y se va directo a generar (`keel-<tech> build specs/catalog`). Si hay que cambiar algo del
contrato estable —multi-tienda, jerarquía de categorías, otra política de eventos—, se **deriva** con
`keel new <nuevo> --from registry:catalog` y `/keel-design` entrevista solo lo que cambia. Lo esperado
para una tienda con un único catálogo es **adoptarlo**.

## Cobertura de comportamiento

`specs/catalog/validation-scenarios.md` cubre las 28 operaciones y todos los códigos de error en 32
flujos: marcas (`FL-BRD`), categorías (`FL-CAT`), productos y ciclo de vida (`FL-PRD`), imágenes
(`FL-IMG`), tienda (`FL-SHP`), servidor-a-servidor (`FL-M2M`), outbox (`FL-EVT`) y CORS (`FL-SEC`),
incluidas las carreras de idempotencia, la concurrencia sobre el agregado y la supervivencia del
evento con el canal caído.
