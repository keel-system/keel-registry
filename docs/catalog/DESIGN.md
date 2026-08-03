# catalog — Documento de diseño

> specs/catalog v0.3.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`catalog` es la **fuente de verdad del catálogo comercial de una tienda**: los productos, sus marcas
y sus categorías, con la ficha, el precio de venta y la galería de imágenes. Sirve a tres públicos
con contratos distintos y separados a propósito: la **tienda** (lectura pública sin credencial, con
búsqueda y filtros), el **back-office** (gestión autenticada del catálogo) y **otros servidores**
(resolución de fichas por id o por lotes, más eventos de dominio).

Queda fuera todo lo que no es la ficha: no lleva stock ni disponibilidad, no modela variantes de un
mismo artículo, no tiene promociones ni precios de campaña, y no es multi-idioma ni multi-divisa.
El detalle de esos límites está en § 7.

## 2. Modelo de dominio

**Value types.** Cuatro, todos con significado de negocio en vez de constraints repetidas inline:
`SKU` (código comercial, patrón `^[A-Z0-9][A-Z0-9-]{2,31}$`), `Slug` (identificador de URL en
kebab-case), `Price` (decimal ≥ 0 con escala 2) y `ProductStatus` (enum `draft` / `active` /
`discontinued`).

**Entidades.**

| Entidad | Campos | Notas |
|---|---|---|
| `Product` | `id`, `sku` (único), `name`, `slug` (único), `description`, `price`, `status`, `lockVersion`, `createdAt`, `updatedAt`, `createdBy`, `updatedBy` | `slug` es `computed`; `id`, `lockVersion` y los cuatro de auditoría son `generated`. Relaciones: `brand` y `category` (many-to-one, obligatorias) e `images` (one-to-many) |
| `ProductImage` | `id`, `file` (bucket `productImages`), `altText`, `position`, `isPrimary` | Entidad hija: tiene identidad propia porque se referencia, se reordena y se borra una a una |
| `Brand` | `id`, `name` (único), `slug` (único, `computed`), `description` | Sin campos de auditoría, a propósito (ver § 6) |
| `Category` | `id`, `name` (único), `slug` (único, `computed`), `description` | Ídem |

No hay ningún campo `sensitive`: el catálogo no guarda secretos. La protección de los datos internos
—quién editó cada ficha— se hace recortando la proyección por superficie, no marcando campos.

**Agregados.** Tres: `Product` (raíz, con `ProductImage` dentro), `Brand` y `Category`. El producto
y sus imágenes cambian siempre en la misma transacción; la marca y la categoría son fronteras
propias que el producto referencia por id.

**Ciclo de vida de `Product`** (campo `status`, arranca en `draft`):

```
draft ──publish──► active ──discontinue──► discontinued
  ▲                  │                          │
  └───unpublish──────┘                          │
  ▲                                             │
  └──────────────reactivate─────────────────────┘   (a active)
```

Las cuatro transiciones son las únicas válidas y cada una tiene su operación con nombre propio.
No hay estados terminales: un producto descatalogado se recupera.

## 3. Invariantes y reglas clave

Del dominio:

- El `sku` es **inmutable** una vez creado el producto.
- Un producto `active` tiene `price` mayor que cero **y al menos una imagen**.
- Solo los productos `active` son visibles en la superficie pública.
- El `slug` de un producto que ya se publicó alguna vez **no vuelve a cambiar**, aunque cambie su
  nombre.
- Un producto `discontinued` no admite cambios en su ficha ni en su galería.
- Un producto con imágenes tiene **exactamente una** `isPrimary`, y no hay dos imágenes con la
  misma `position`.
- Una marca o una categoría no se pueden eliminar mientras algún producto las referencie.
- Dos marcas (o dos categorías) no pueden tener el mismo nombre **ignorando mayúsculas y acentos**.

De los casos de uso:

- El `sku` se normaliza a mayúsculas antes de validar unicidad.
- Las tres operaciones de edición son **reemplazo completo**: lo que no viene en la petición queda a
  nulo. No hay semántica de fusión.
- La referencia producto→marca y producto→categoría es una **restricción de integridad**, no una
  comprobación previa: es lo que cierra la carrera entre crear un producto y borrar su marca.
- Los tres borrados son **idempotentes**: repetirlos sobre un recurso que ya no existe responde
  `204` y no cambia nada. Un `204` no implica, por tanto, que se haya publicado evento de borrado.
- Los filtros de texto (`name`, `sku`) buscan coincidencia parcial sin distinguir mayúsculas ni
  acentos.
- La derivación del sufijo del slug de `Product` (lectura del slug libre + escritura) no es atómica:
  dos altas o ediciones simultáneas del mismo `name` pueden calcular el mismo sufijo y solo una
  llega a escribir. Esa carrera se resuelve con `PRODUCT_SLUG_CONFLICT`, nunca reintentando en
  silencio.

## 4. Qué hace

**Gestión de productos** (back-office, token de usuario). `createProduct` da de alta en `draft` y es
**idempotente por clave de cliente** (ventana 24 h). `updateProduct` cambia la ficha y exige la
`lockVersion` leída. Cuatro operaciones con nombre de intención cubren el ciclo de vida:
`publishProduct`, `unpublishProduct`, `discontinueProduct`, `reactivateProduct`. `getProduct` y
`listProducts` consultan en cualquier estado, con filtros de gestión (nombre, sku, estado, marca,
categoría) y orden por `updatedAt` descendente.

**Galería** (back-office). `addProductImage` (idempotente por clave de cliente, devuelve la imagen
creada con su id), `setPrimaryProductImage`, `reorderProductImages` y `removeProductImage`. Las
cuatro operan sobre el agregado a través de la raíz. `removeProductImage` es un `DELETE`
**idempotente**: responde `204` sin cuerpo tanto si eliminó la imagen como si ya no estaba, y solo
publica `ProductUpdated` cuando hubo eliminación real. El producto de la ruta sí tiene que existir.

**Taxonomía** (back-office). `createBrand` / `updateBrand` / `deleteBrand` y sus tres equivalentes
de categoría. El borrado se rechaza si hay productos que la referencian, y —como el de imágenes— es
idempotente: sobre una marca o categoría que ya no existe responde `204` sin publicar evento.

**Tienda** (público, sin credencial). `listPublicProducts` con los cuatro filtros pedidos —categoría,
marca, nombre y rango de precio— paginado y ordenado por nombre, y `getPublicProduct` por slug. Las
dos **cachean 300 s**, invalidadas por los cinco eventos que pueden cambiar su respuesta.
`listBrands` y `listCategories` pueblan los menús de filtrado.

### Superficie servidor-a-servidor

Contrato propio, separado del de usuarios desde el primer día y con otro equipo acoplado detrás:

| Operación | Endpoint | Qué promete |
|---|---|---|
| `getProductForServices` | `GET /api/v1/services/products/{productId}` | La ficha de un producto **en cualquier estado**, incluidos los descatalogados que un pedido histórico referencia |
| `listProductsBatchForServices` | `POST /api/v1/services/products/batch` | Hasta **100** ids en una llamada; los inexistentes se omiten sin error y los repetidos se resuelven una vez |

Las dos exigen credencial de máquina con el scope `product:read` y validan la audiencia del token.
Su proyección conserva `updatedAt` —necesario para reconciliar— y excluye el resto de la auditoría.
`POST` en la operación de lotes es deliberado: 100 uuid no caben en una URL.

**Eventos publicados**: `ProductCreated`, `ProductUpdated` y `ProductStatusChanged` en el canal
`productEvents`; `BrandCreated` / `BrandUpdated` / `BrandDeleted` y `CategoryCreated` /
`CategoryUpdated` / `CategoryDeleted` en `taxonomyEvents`.

## 5. Fronteras e integraciones

`catalog` **no depende de ningún otro servidor**: no hay capa `dependencies` ni `http-clients`. Todo
lo que expone es suyo. Las fronteras son de salida.

- **Mensajería.** Dos canales lógicos. Los eventos de producto llevan la **ficha completa** (con
  marca y categoría resueltas por nombre y la imagen principal), para que un consumidor mantenga su
  copia sin llamar de vuelta. Publicación **best-effort**: ver § 6.
- **Persistencia.** Modelo relacional, frontera transaccional **por agregado**, bloqueo optimista
  declarado solo en `Product`. Auditoría `declared` en producto (los cuatro campos son contrato).
  Los índices se eligieron contra las queries reales, no contra la forma de las entidades: `slug` en
  marcas y categorías porque el filtro público llega por slug, y `(status, name)` porque es lo que
  sostiene el listado paginado de la tienda.
- **Almacenamiento.** Un bucket, `productImages`: `public`, JPEG/PNG/WebP, 5 MB. El archivo se borra
  junto con la imagen.
- **Seguridad.** OIDC para usuarios, `client-credentials` con validación de audiencia para máquinas.
  Un rol (`catalog-admin`) y cuatro permisos (`product:read`, `product:write`, `product:publish`,
  `taxonomy:write`). Tres `serviceClients` —`order-service`, `inventory-service`, `search-service`—
  con `product:read` y nada más. CORS declarado: la tienda y el back-office son SPA.

## 6. Decisiones de diseño (qué / por qué)

| Decisión | Qué se decidió | Por qué | Alternativa descartada |
|---|---|---|---|
| **Fiabilidad de publicación** | `best-effort` | El diseñador acepta perder un evento si el broker está caído al confirmar, porque existe vía de reconciliación: `listProductsBatchForServices` permite a un consumidor reconstruir su copia | `outbox` (recomendado por el agente): ningún evento se pierde, a cambio de una tabla y un relay que operar |
| **Frontera de agregado** | `Product` + `ProductImage` juntos; `Brand` y `Category` aparte | No se acepta que una escritura a medias deje imágenes huérfanas o dos imágenes principales. Marca y categoría cambian a otro ritmo y con otro actor | Entidades independientes |
| **Frontera transaccional** | `per-aggregate` | Ninguna operación escribe dos agregados a la vez, así que la transacción por agregado basta y contiende menos | `per-operation` |
| **Concurrencia** | Bloqueo optimista solo en `Product`, con `409` | Un operador no debe ver desaparecer su cambio de precio sin aviso. La taxonomía la edita un admin de tanto en tanto: ahí gana la última escritura, dicho a sabiendas | Último gana en todo |
| **Idempotencia** | `client-key` en `createProduct` y `addProductImage`, 24 h | El reintento tras un timeout no debe producir un alta duplicada ni una foto repetida, ni un `409` confuso | `payload-hash` (dos altas legítimas idénticas colisionarían) y sin idempotencia |
| **`DELETE` idempotente** | Los tres borrados (`deleteBrand`, `deleteCategory`, `removeProductImage`) responden `204` también cuando el recurso ya no está, y por eso no declaran un error de «no encontrado». El evento de borrado sale solo si hubo eliminación real | Un reintento de red tras un `204` perdido no debe ensuciar el cliente con un `404` que describe un éxito. La idempotencia cubre la ausencia del recurso, nunca las invariantes: `BRAND_IN_USE`, `CATEGORY_IN_USE`, `PRODUCT_DISCONTINUED` y `LAST_IMAGE_OF_ACTIVE_PRODUCT` se siguen evaluando | `404` al repetir (`BRAND_NOT_FOUND`, `CATEGORY_NOT_FOUND`, `PRODUCT_IMAGE_NOT_FOUND` en esas operaciones): delata el borrado doble como bug del cliente, a cambio de que un reintento legítimo sea indistinguible de un error. También se descartó extender la idempotencia al `productId` de `removeProductImage`, que dejaría pasar un typo en la ruta como éxito |
| **Caché** | 300 s en las dos lecturas públicas, con cinco vías de invalidación | La tienda es la carga dominante. Como la respuesta embebe marca y categoría, la elección **obligó** a publicar `BrandUpdated` y `CategoryUpdated`: sin ellos, renombrar una marca tardaría 5 min en verse y nada lo delataría | Cachear solo la ficha; no cachear |
| **Superficie M2M** | Operaciones propias con `audience: services` | Los dos contratos ya divergen: el M2M devuelve productos en cualquier estado, cosa que la tienda nunca debe ver. Compartir endpoint sería compartir output, errores y scopes | `audience: both` |
| **Auditoría** | `declared` en `Product`; **ninguna** en `Brand` y `Category` | Los cuatro campos son contrato porque el back-office muestra quién tocó cada ficha. Se quitaron de la taxonomía porque el DSL **no permite recortar un objeto `embed`**: con ellos, la marca anidada en cada ficha pública habría filtrado el nombre del operador que la creó | Mantener la auditoría en la taxonomía y aceptar la fuga; quitar el `embed` de las superficies públicas |
| **Visibilidad del bucket** | `public` | Material de catálogo pensado para verse, servible y cacheable en el borde. Riesgo asumido: la foto de un producto en `draft` es visible para quien acierte con su URL | `private` con URL firmada: impide cachear en el borde y añade una operación de lectura |
| **Slug congelado al publicar** | Se recalcula solo mientras el producto está en `draft` | Una corrección ortográfica del título no debe romper los enlaces compartidos ni el posicionamiento de una ficha ya publicada | Recalcular siempre; mantener un historial de slugs con redirección |
| **Producto descatalogado inmutable** | `PRODUCT_DISCONTINUED` (409) en las cinco operaciones de edición | La ficha que un pedido histórico referencia no debe cambiar de precio ni de fotos después de vendido. Para tocarla hay que reactivar el producto | Editable como cualquiera |
| **Sin borrado de productos** | Solo `discontinueProduct` | Un borrado real dejaría referencias rotas en todo consumidor con copia, sin vía de reparación | Borrado real; borrado solo en `draft` |
| **Borrado de taxonomía bloqueado** | `BRAND_IN_USE` / `CATEGORY_IN_USE` (409) | Ningún producto queda huérfano y ningún consumidor ve una referencia rota. La garantía es de integridad referencial, no un `SELECT` previo, para cerrar la carrera | Cascada (una acción de mantenimiento sacaría cientos de productos de la tienda); baja lógica |
| **Rechazo de colisión de slug en taxonomía** | `BRAND_SLUG_CONFLICT` / `CATEGORY_SLUG_CONFLICT` (409) | La taxonomía es corta y la maneja un humano: mejor que corrija el nombre a tener `/marcas/nike-2` en el menú de la tienda. En productos, en cambio, el volumen obliga al sufijo automático | Sufijo numérico también en taxonomía |
| **Carrera del sufijo de slug en `Product`** | `PRODUCT_SLUG_CONFLICT` (409) al perdedor de la carrera, sin reintento en servidor | Reintentar en servidor añade un bucle con su propio límite y ventana de bloqueo, y esconde al cliente que perdió una carrera de nombres; el 409 es más simple de implementar y deja la decisión de reintentar (con qué backoff) en manos del cliente, igual que ya ocurre con `SKU_ALREADY_EXISTS` | Reintento automático en servidor con el siguiente sufijo libre |
| **Un solo rol** | `catalog-admin` | Decisión del diseñador contra la recomendación del agente (admin + editor). Queda anotado que cualquier operador puede borrar taxonomía compartida | `catalog-admin` + `catalog-editor` sin permiso sobre marcas y categorías |
| **Reemplazo total en las ediciones** | Omitir un campo lo pone a nulo | Sin ambigüedad y coherente con el `PUT` elegido; el formulario del back-office manda siempre la ficha entera | Fusión (`PATCH`): exige distinguir "ausente" de "nulo", que el DSL no expresa hoy |
| **Al menos una imagen para publicar** | `PRODUCT_NOT_PUBLISHABLE` (422) | La tienda nunca pinta un hueco, y no tiene que decidir por su cuenta qué mostrar cuando falta la foto | Imagen opcional |
| **Cota del lote M2M** | 100 ids, como `precondition` y no como `constraints` | Cabe el carrito más grande y acota el coste de una petición. Va en prosa para que el exceso falle con el código estable `TOO_MANY_IDS` en vez de con un error genérico de forma | 50; 500; declararlo en `constraints` |
| **Paginación** | Offset, 20 por defecto, 100 máximo, en las cuatro colecciones | Una rejilla de tienda pinta 20 y el back-office puede pedir 100; el tope protege la consulta con filtro de precio | 24/60; 50/200 |
| **Precio sin moneda** | `Price` es un decimal, la divisa es implícita | La tienda opera en una sola divisa | `Money` (importe + moneda ISO-4217): abrir un segundo mercado será un cambio incompatible del contrato |

## 7. Ficha de reutilización: adoptar, derivar o evolucionar

### Contrato estable vs adaptable

**Estable** —cambiarlo rompe a alguien y exige versión mayor—: los 24 códigos de error en
`SCREAMING_SNAKE_CASE` con su status HTTP; los nueve nombres de evento y la forma de su payload; los
dos endpoints `audience: services` y sus scopes; los cuatro endpoints públicos y la forma de sus
filtros; el prefijo `/api/v1` (una ruptura abre `/api/v2`, que convive con la anterior mientras los
tres consumidores migran); los nombres de rol y permiso; el nombre lógico de los canales y del
bucket; y la **semántica idempotente de los tres `DELETE`** —un cliente que trate el `204` repetido
como éxito se rompe si algún derivado vuelve a introducir el `404`.

**Adaptable** sin romper a nadie: las `rules` y `preconditions` de los casos de uso; los TTL y las
vías de invalidación de la caché; la ventana de idempotencia; los límites del bucket
(`maxSizeMb`, `allowedContentTypes`); los índices de `persistence`; el máximo de 10 imágenes por
producto; los tamaños de página.

Versionado del spec: **patch** para prosa y reglas internas; **minor** para operaciones, campos
opcionales o eventos nuevos; **major** para cualquier cambio en lo estable de arriba.

### Puntos de extensión típicos

- **`lifecycle` de `Product`**: hay sitio evidente para un estado de revisión editorial
  (`draft → pending_review → active`) sin tocar los estados existentes.
- **`ProductStatus`** es un enum nominal ampliable; los códigos de error no.
- **Capas ausentes** que un derivado puede añadir sin rediseñar: `dependencies` y `http-clients` (si
  el precio o el stock pasan a venir de fuera), y una operación con `schedule` si el negocio quiere
  un barrido de reconciliación propio en vez de dejárselo a los consumidores.
- **Piezas reutilizables en otro servicio**: los value types `Slug` y `Price`, el patrón de galería
  (entidad hija con `position` + `isPrimary` y sus cuatro operaciones), y el patrón de superficie
  triple (pública / gestión / M2M) con proyecciones distintas por `exclude`.

### Supuestos y limitaciones

**Asume**: un solo tenant —el catálogo es de **una tienda**, no de un marketplace, y no hay noción
de "mis productos" ni autorización a nivel de dato—; una sola divisa; un solo idioma; y un
**catálogo de tamaño medio**, de hasta decenas de miles de productos y unos cientos de marcas y
categorías, que es lo que sostienen los índices elegidos y el listado paginado con filtro de precio.

**No cubre, a propósito**:

- **Stock e inventario.** La disponibilidad vive en otro servicio; el catálogo no sabe si hay
  unidades.
- **Variantes.** Cada combinación vendible (talla, color) es un producto con su propio SKU; no hay
  modelo padre-variante.
- **Multi-idioma y multi-divisa.** Nombres, descripciones y precios en un solo idioma y una sola
  divisa.
- **Promociones y descuentos.** El `price` es el de venta; no hay precios tachados, campañas ni
  reglas de descuento.

**Limitaciones conocidas del diseño tal cual**:

- Con `best-effort`, un consumidor puede perder un evento sin traza. La reconciliación es
  responsabilidad suya, vía `listProductsBatchForServices`. Un derivado que no pueda asumirlo debe
  cambiar `messaging.publishing.reliability` a `outbox`.
- Un catálogo de cientos de miles de productos deja corto el filtro por nombre de
  `listPublicProducts`: eso ya es un servicio de búsqueda, no una query de este servicio.
- La cobertura de caché tiene un límite conocido y documentado en `validation-scenarios.md`
  (FL-CCH-001): como `invalidatedBy` es exhaustivo, no existe una vía de mutación con la que probar
  la **retención**, así que los escenarios de invalidación los pasaría también una implementación
  que no cacheara nada.

### Cómo reutilizarlo

Empieza por el resumen mecánico: `keel describe catalog` (identidad, estado, capas, contenido y
frescura de los derivados).

- **Si te sirve tal cual** —tienda de una sola divisa, sin variantes ni stock aquí—, **adóptalo**:
  `keel registry get catalog`. Llega con sus derivados al día y vas **directo a generar**, sin fase
  de diseño.
- **Si tienes que cambiarlo**, **derívalo**: `keel new <nuevo> --from registry:catalog`. Clona solo
  el spec con el linaje en `basedOn` y hace que `/keel-design` entreviste **solo lo que cambia**.

Dado lo que dice «Contrato estable vs adaptable», **lo esperable en la mayoría de casos es
adoptarlo**: las piezas que un negocio suele querer distinta —tamaños de página, TTL de caché,
límites del bucket, ventana de idempotencia— son todas adaptables sin tocar el contrato, y varias se
deciden al generar. Derivar solo compensa si hay que cambiar algo de la lista estable: añadir
variantes, pasar a multi-divisa, convertirlo en marketplace multi-vendedor o cambiar la fiabilidad
de publicación a `outbox`. Derivar para acabar usándolo sin cambios obliga a regenerar a mano todo
lo que ya estaba hecho.
