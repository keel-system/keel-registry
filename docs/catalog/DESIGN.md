# catalog — Documento de diseño

> specs/catalog v0.1.2. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`catalog` es la **fuente de verdad del catálogo comercial de una tienda**: los productos, las marcas
bajo las que se venden y las categorías por las que se navegan. Sirve a tres públicos con tres
contratos distintos y deliberadamente separados: la **tienda pública**, donde un consumidor anónimo
busca y filtra artículos a la venta; el **back-office**, donde el personal de la tienda carga y
mantiene fichas con rastro de autoría; y **otros servidores**, que resuelven productos por id o por
lote y se mantienen al día con los eventos que el catálogo publica.

Lo que queda fuera es tan importante como lo que entra: el catálogo **no** lleva existencias, no
conoce variantes de un artículo, no traduce ni convierte divisas, y no indexa para búsqueda con
relevancia. Es la verdad de **qué se vende y a qué precio**, no de cuánto hay ni de cómo se encuentra.

## 2. Modelo de dominio

**Value types.** Cuatro, cada uno con significado de negocio en lugar de repetir restricciones:

- `SKU` — el código con el que el negocio conoce el artículo (almacén, proveedor, facturas). Único e
  inmutable: identifica al mismo artículo físico durante toda su vida.
- `Slug` — identificador legible para las URLs públicas, derivado del nombre: minúsculas,
  diacríticos transliterados a su letra base (`"Portátiles"` → `portatiles`), separadores a guiones
  y el resto de caracteres descartados.
- `Price` — importe de venta en la divisa única de la tienda. La escala de dos decimales se
  **valida, no se ajusta**: un importe con más decimales se rechaza en vez de redondearse, porque el
  servicio no hace aritmética sobre el precio; lo guarda y lo devuelve tal cual llegó
  (`scalePolicy: reject`: `19.999` es un `400`, también en los filtros de rango de precio).
- `ProductStatus` — `draft`, `active`, `discontinued`.

**Entidades.**

| Entidad | Campos relevantes |
|---|---|
| `Brand` | `id`, `name` (único), `slug` (*computed*), `description` |
| `Category` | `id`, `name` (único), `slug` (*computed*), `description` — planas, sin jerarquía |
| `Product` | `id`, `sku` (único), `name`, `slug` (*computed*), `slugFrozen` (*generated*), `description`, `price`, `status`, `lockVersion` (*generated*, avanza exactamente una vez por escritura confirmada), `createdAt` / `updatedAt` / `createdBy` / `updatedBy` (*generated*) |
| `ProductImage` | `id`, `image` (archivo en el bucket `productImages`), `altText`, `position`, `primary` |

`Product` referencia una `Brand` y una `Category` (obligatorias, por id) y posee una galería de
`ProductImage`. Ningún campo es `sensitive`: un catálogo comercial no guarda secretos, y lo más
cercano —quién editó la ficha— se controla por proyección (sale en el back-office, no en la tienda).

**Agregados.** `Product` es raíz y sus imágenes son internas; `Brand` y `Category` son agregados
propios que los productos referencian por id. Se entra siempre por el producto.

**Lifecycle de `Product`** (sobre `status`, y solo estas transiciones son válidas):

```
draft ──publishProduct──▶ active ──discontinueProduct──▶ discontinued
  ▲                         │                                  │
  └───unpublishProduct──────┘◀──────reactivateProduct──────────┘
```

No hay estado terminal: un artículo descontinuado se puede volver a poner a la venta.

## 3. Invariantes y reglas clave

- Un producto `active` siempre tiene un `price` mayor que cero.
- Como máximo **una** imagen de un producto es `primary`; si está `active` y tiene imágenes, tiene
  exactamente una. Dos imágenes del mismo producto nunca comparten `position`.
- Dos marcas (o dos categorías) no pueden tener el mismo `name` **ignorando mayúsculas y acentos**
  (`compare: ignore-case-accents` en el campo: `ACME`, `acme` y `Acmé` son el mismo nombre);
  tampoco el mismo `slug`, que es un fallo distinto y tiene su propio código (`Acme!` frente a `Acme`).
- El `sku` es inmutable: no forma parte de la entrada de `updateProduct`.
- La primera imagen de una galería vacía recibe `position` 0; cada siguiente, la posición siguiente a
  la mayor. Y la primera imagen de un producto es su `primary` aunque no se pida.
- En las cuatro operaciones con clave de idempotencia, la clave se resuelve **antes** que cualquier
  guarda de negocio: un reintento tiene que reproducir la respuesta original sin volver a chocar con
  la unicidad del `sku` o del nombre. El orden de la lista `errors` de cada operación es el orden en
  que se evalúan sus guardas.
- El control de versión de `Product` protege frente a escrituras que **se solapan**: la versión leída
  no viaja en la petición, así que dos ediciones sucesivas se aplican en orden y prevalece la última.
- El `slug` de un producto sigue a su nombre **hasta la primera publicación**; ahí se congela
  (`slugFrozen`) y no vuelve a moverse, ni siquiera al despublicar.
- Una marca o categoría no se elimina mientras algún producto la referencie.
- Solo los productos `active` salen en la tienda pública. Un borrador o un descontinuado responden
  `404` en el escaparate: indistinguibles de lo que no existe.
- La entrada de las actualizaciones es la **representación completa** del recurso: un campo opcional
  que no viene se vacía, no se conserva.

## 4. Qué hace

**Gestión de productos (back-office).** `createProduct` (nace en `draft`), `updateProduct`,
`publishProduct`, `unpublishProduct`, `discontinueProduct`, `reactivateProduct`, `deleteProduct`
(solo un borrador que nunca se publicó), `getProduct` y `listProducts` (paginado, filtros por estado,
marca, categoría, nombre y sku, combinados con AND; el de nombre es coincidencia parcial ignorando
mayúsculas y acentos). Las cuatro transiciones de estado son
operaciones con nombre de intención, no un `updateStatus` genérico: cada una tiene sus propias
guardas y su propio error.

**Galería.** `addProductImage` (idempotente, la primera imagen se marca principal sola),
`removeProductImage` (si cae la principal, asciende la de menor posición), `setPrimaryProductImage` y
`reorderProductImages`. Las cuatro se rechazan sobre un producto descontinuado.

**Taxonomía.** `createBrand`, `updateBrand`, `deleteBrand` y sus tres homólogas de categoría, todas
reservadas a `catalog-admin`.

**Tienda pública (anónima).** `listPublicProducts` (paginado, orden por nombre, filtros por categoría,
marca, nombre —coincidencia parcial, sin distinguir mayúsculas ni acentos— y rango de precio
inclusivo), `getPublicProduct` (por slug, con caché de 60 s),
`listBrands` y `listCategories` para la navegación.

**Idempotencia y caché.** Las cuatro operaciones que pueden duplicar un efecto **fuera** del proceso
—`createProduct`, `addProductImage`, `createBrand`, `createCategory`— exigen la cabecera
`Idempotency-Key` (TTL 24 h) y rechazan con `400` si no viene. Las dos fichas individuales
(`getPublicProduct`, `getProductForServices`) se cachean 60 s, invalidadas por `ProductUpdated`,
`ProductStatusChanged`, `BrandUpdated` y `CategoryUpdated`. Los listados no se cachean.

### Superficie servidor-a-servidor

Dos operaciones con **contrato propio**, bajo `/api/v1/services/**` y con `audience: services`:

| Operación | Endpoint | Qué promete |
|---|---|---|
| `getProductForServices` | `GET /api/v1/services/products/{productId}` | La ficha de un producto `active` o `discontinued`, con marca y categoría resueltas y `updatedAt` para que el consumidor sepa de cuándo es. Un borrador responde `404`. |
| `listProductsBatchForServices` | `GET /api/v1/services/products` | Hasta **100** ids en una llamada. Los inexistentes, los repetidos y los borradores se omiten sin fallar; el resultado sale ordenado por `sku`, no en el orden de la petición. |

Ni la autoría (`createdBy`, `updatedBy`) ni el control de versión salen por esta puerta. Los tres
consumidores declarados —`orders-service`, `cart-service`, `search-service`— reciben exactamente la
misma respuesta para el mismo id: quién pregunta no cambia lo que se responde.

## 5. Fronteras e integraciones

`catalog` **no depende de ningún otro servidor**: no declara capas `dependencies` ni `http-clients`.
Solo publica y sirve. Eso hace que se pueda desplegar y probar solo, y es lo que lo coloca en la
primera ola de construcción de cualquier sistema que lo use.

**Eventos publicados** (`reliability: outbox` — el evento se escribe en la misma transacción que el
agregado y un relay lo publica después, así que ningún cambio confirmado se queda sin anunciar):

| Canal | Eventos |
|---|---|
| `productEvents` | `ProductCreated`, `ProductUpdated`, `ProductStatusChanged`, `ProductDeleted` |
| `taxonomyEvents` | `BrandCreated`, `BrandUpdated`, `BrandDeleted`, `CategoryCreated`, `CategoryUpdated`, `CategoryDeleted` |

Los eventos de producto llevan la **ficha completa** (incluidos `brandName`, `categoryName` y la URL
de la imagen principal) para que un consumidor se actualice sin llamar de vuelta. La contrapartida
está declarada en el canal de taxonomía: al renombrarse una marca, el catálogo **no** re-emite los
eventos de sus miles de productos, así que quien copie `brandName` tiene que escuchar
`taxonomyEvents` y actualizar sus copias por `brandId`. `ProductDeleted` es la vía de baja: sin ella
una copia local se queda rancia para siempre. No hay suscripciones: el catálogo no consume eventos.

**Persistencia.** Modelo relacional, transacción **por agregado** (la fila del outbox entra en esa
misma transacción), y bloqueo optimista **solo en `Product`**. Los índices salen de las queries
declaradas: `[status, name]`, `[status, brandId]`, `[status, categoryId]`, `[status, price]`, el slug
único, `[updatedAt]` para la tabla del back-office y `[productId, position]` para la galería, más una
**unicidad condicionada** (`productId` donde `primary` es verdadero) que es lo único que cierra la
ventana de dos peticiones marcando principales distintas; la que pierde esa carrera responde
`409 CONCURRENT_MODIFICATION`, porque la galería es parte del agregado `Product`. El rastro de auditoría es `declared` solo
en `Product`. Nada se archiva ni se purga: un descontinuado se guarda para siempre porque los pedidos
históricos lo referencian, y su `sku` sigue ocupado para siempre porque identifica a **ese** artículo.

**Archivos.** Un bucket lógico, `productImages`: `image/jpeg`, `image/png`, `image/webp`, hasta 5 MB,
**público**. Las fotos son el escaparate, así que la URL es estable y cacheable por un CDN y viaja en
los eventos. La contrapartida aceptada: la URL es la única protección, de modo que la foto de un
producto en borrador es alcanzable por quien tenga el enlace.

**Acceso.** OIDC para usuarios y `client-credentials` con validación de audiencia para máquinas.
Cuatro permisos (`product:read`, `product:write`, `product:delete`, `taxonomy:write`) y dos roles:
`catalog-editor` carga y mantiene fichas y galerías; `catalog-admin` añade lo que es **estructura**
compartida —marcas, categorías— y el borrado. Los cuatro endpoints de la tienda son `public`; todo lo
demás exige token, y la regla por defecto es cerrada. Hay política `cors` porque la tienda y la SPA
de back-office llaman desde el navegador; los orígenes concretos son despliegue, no diseño.

## 6. Decisiones de diseño (qué / por qué)

| Decisión | Qué se eligió | Por qué, y qué se descartó |
|---|---|---|
| **Ciclo de vida de tres estados** | `draft → active → discontinued`, con vuelta desde los dos | Un flag `published` no distingue «todavía no está listo» de «ya no se vende», y sin transiciones declaradas ninguna guarda es verificable. Descartado: flag booleano; y no tener estado, que publicaría en la tienda cualquier ficha a medio rellenar. |
| **Imágenes como entidad hija** | `ProductImage` con identidad, orden y principal | Una ficha de tienda real necesita borrar una foto concreta, reordenar y elegir la de portada. Descartado: una lista de value objects, que obliga a reemplazar la galería entera y no permite referenciar una imagen. |
| **Frontera del agregado** | `Product` raíz + `ProductImage` internas; `Brand` y `Category` aparte | Es lo que hace **garantizable** «una sola principal» y «sin dos en la misma posición». Descartado: cada entidad como frontera propia, que deja imágenes huérfanas y dos principales posibles. |
| **Slug congelado al publicar** | Sigue al nombre en borrador; se congela en la primera publicación | Un enlace compartido o indexado no debe romperse porque alguien corrija una errata en el nombre. Descartado: recalcular siempre (URL coherente, enlaces roto) y slug escrito a mano (un campo más en cada alta). |
| **Borrado solo de borradores** | Un `draft` nunca publicado se borra; el resto se descontinúa | Otros servidores ya guardan el id de todo lo que se publicó alguna vez, y borrarlo les rompe. Descartado: borrado siempre con guarda de referencias; y no borrar nada, que deja para siempre los productos creados por error. |
| **§3.1 Fiabilidad de publicación** | `outbox` | `search-service` mantiene su índice **solo** con estos eventos: uno perdido es un producto que no aparece en la búsqueda, y nada lo delata. Descartado: `best-effort` (se descubre semanas después por descuadre) y `best-effort` con reconciliación por el endpoint de lote, que solo es honesto si cada consumidor hace de verdad ese barrido. |
| **§3.2 Idempotencia** | `client-key`, TTL 24 h, en `createProduct`, `addProductImage`, `createBrand`, `createCategory`; cabecera **obligatoria** (`400` si falta) | Son las cuatro cuyo duplicado **sale del servicio** en forma de evento, y ninguna clave natural lo desanda: dos fotos iguales son legítimas. La cabecera es obligatoria porque una garantía opcional es la que el cliente olvida justo el día del reintento. Descartado: apoyarse en la unicidad del sku (el gestor recibe un `409` donde esperaba éxito y no sabe si su producto existe) y aceptar el duplicado. |
| **§3.3 Caché** | 60 s solo en las dos fichas individuales, invalidadas por los eventos de producto **y** de taxonomía | La marca y la categoría van embebidas en la respuesta, así que también tienen que poder invalidarla. Los listados quedan fuera: sus filtros multiplican las claves y cualquier cambio las invalida. Descartado: sin caché (la portada pega al almacén en cada visita) y caché en los listados (un precio recién cambiado tardaría el TTL en aparecer). |
| **§3.4 Superficie M2M** | Operaciones y endpoints propios bajo `/services` | Los dos contratos crecen en direcciones opuestas: el de máquina hacia lotes y campos estables, el público hacia la pantalla. Descartado: `audience: both`, que comparte output, errores, paginación y scopes, y convierte todo cambio para la tienda en un cambio para el consumidor. |
| **§3.7 Frontera transaccional** | `per-aggregate` | Ninguna operación escribe dos agregados a la vez, así que basta y contiende menos. Descartado: `per-operation`, que es el default cómodo y deja de ser una decisión de consistencia. |
| **§3.8 Paginación** | offset, 20 por defecto, tope 100, en los cuatro listados | 20 sirve igual a una rejilla de tienda y a una tabla de back-office. `listBrands` y `listCategories` también paginan: son colecciones sin cota. Descartado: 12/48 (corto para el back-office), 50/200 (una portada cargaría 50 fichas de entrada) y navegación sin paginar. |
| **§3.9 Concurrencia** | `lockVersion` solo en `Product` → `409 CONCURRENT_MODIFICATION`; último-gana en la taxonomía | Dos gestores editando la misma ficha no deben pisarse el precio en silencio; una marca la toca un admin de tanto en tanto. Descartado: último-gana en todo (se pierde una edición sin que nadie se entere) y bloqueo en las tres entidades (tres formularios manejando el `409`). |
| **§3.9b Auditoría** | `declared`, solo en `Product` | La autoría es parte del contrato del back-office, y solo de `Product` para que la marca y la categoría embebidas en una ficha pública no arrastren quién gestiona la tienda. Descartado: declararla en las tres (habría que excluirla de cada salida pública) y `all` (el back-office no podría mostrar «editado por»). |
| **§3.10 Visibilidad del bucket** | `public` | Las fotos son el escaparate: URL estable, cacheable y transportable en un evento. Descartado: `private` con URL firmada (rompe el `primaryImageUrl` de los eventos y ningún CDN cachea lo que caduca) y descarga mediada (el catálogo pasaría a cargar con todo el tráfico de imágenes). |
| **Guardas de publicación** | Precio > 0 y al menos una imagen | Es lo primero que ve el consumidor. Descartado: publicar sin foto, y publicar sin ninguna guarda. |
| **Un solo código para cada fallo** | `BRAND_REFERENCE_NOT_FOUND` (422) ≠ `BRAND_NOT_FOUND` (404); `BRAND_SLUG_ALREADY_EXISTS` ≠ `BRAND_NAME_ALREADY_EXISTS` | El mismo `code` con dos status significa dos cosas para quien integra; y «los nombres no coinciden pero el slug sí» es un fallo distinto que el gestor resuelve de otra manera. |
| **Errores del framework como contrato** | Se aceptan `CONCURRENT_MODIFICATION`, `IDEMPOTENCY_KEY_IN_PROGRESS` y `IDEMPOTENCY_KEY_REUSED` canónicos | Ninguno es un hecho del catálogo: son del mecanismo, y tener el mismo nombre en cualquier stack generado de este diseño es lo que se quiere de ellos. Registrado en `decisions.yaml`. |
| **Identidad del llamante fuera del trabajo** | Sin `callerIdentity` | Los dos endpoints M2M son lecturas del catálogo entero: la credencial decide **si** puedes leer, no **qué** lees. No hay inquilino ni recurso en cuyo nombre se actúe. Registrado en `decisions.yaml`, y caduca si algún día el catálogo sirve vistas distintas por consumidor. |
| **Acceso por permiso, con el rol como puerta donde importa** | Permisos en las 12 operaciones de producto; `level: admin` + `roles: [catalog-admin]` en las 7 irreversibles o estructurales | Repetir los roles en las 25 reglas duplicaría lo que `roleGrants` ya dice en dos líneas. `catalog-editor` se define por lo que **no** alcanza; `keel validate` avisa de que ninguna regla lo exige por nombre, y se acepta con ese porqué. |
| **Versionado del contrato** | Convivencia de `/api/v1` y `/api/v2` | Permite evolucionar sin coordinar el despliegue de los tres consumidores a la vez. Descartado: solo aditivo (un error de contrato se arrastra para siempre) y romper coordinando (cada cambio es una ventana de mantenimiento). |
| **Ausencia** | Un campo sin valor **no viaja** | Convención única del servicio, en respuestas y en payloads de evento: es lo que hace que dos stacks produzcan el mismo JSON. Descartado: `null` explícito. Desde v0.1.1 está declarada en el manifiesto (`conventions.nulls: omit`) y ya no solo en la prosa de los escenarios. |
| **Retención de caché, observada por la señal del servidor** | Los escenarios de retención afirman el **acierto de caché**, no un cuerpo viejo | Toda vía por la que la ficha cambia está declarada en `invalidatedBy` —que es lo que se quiere de ella—, y sustituir el objeto de la imagen en el bucket con la misma clave no cambia ningún campo del cuerpo: la retención no es observable comparando respuestas. O se afirma sobre la señal del servidor —el mismo trato que «ningún evento abandonado» del outbox— o no se afirma, y sin ella una implementación que no cachea nada pasa todas las aserciones de invalidación. Descartado: quitar la aserción. Aceptado por escrito en `flow-review.yaml` (v0.1.2). |
| **El presupuesto de reintentos del relay es del generador** | `outbox` promete que un cambio confirmado no se queda sin anunciar **por una caída del canal**; el evento que agota los reintentos se da por abandonado, se cuenta y se señala | Prometer que ningún evento se abandona jamás es una promesa que ningún relay cumple, y esconderla es peor que declararla: un evento perdido en silencio es un producto que no aparece en la búsqueda y nadie se entera. Cuántos intentos son es decisión del stack, no del diseño. Precisado en v0.1.2. |
| **Carrera de la imagen principal** | El choque del índice único parcial responde `409 CONCURRENT_MODIFICATION`, sin código propio | Ninguna petición legítima puede chocar: las dos operaciones que marcan una principal desmarcan la anterior en la misma transacción. Solo choca la carrera de dos escrituras concurrentes sobre la galería, que es una modificación concurrente del agregado, así que «reintenta» es la respuesta correcta. Descartado: un `PRIMARY_IMAGE_CONFLICT` propio (minor), que nombraría un caso que el cliente no puede provocar a propósito. Decidido en v0.1.1. |

## 7. Ficha de reutilización: adoptar, derivar o evolucionar

### Contrato estable vs adaptable

**Estable** (cambiarlo rompe a quien ya integra, y es cambio **major**): los 32 códigos de error, los
nombres de los diez eventos y sus payloads, los dos canales lógicos, las rutas publicadas y los
status de éxito, los cuatro permisos y los dos roles, y el sobre de paginación.

**Adaptable sin romper a nadie** (cambio **minor** o **patch**): las reglas de negocio de
`use-cases`, los TTL de caché y de idempotencia, los tamaños de página, los límites del bucket
(content-types, `maxSizeMb`), el tope de diez imágenes por producto, la cota de cien ids del lote, los
índices sugeridos y la prosa de las descripciones.

### Puntos de extensión típicos

- **El lifecycle** admite estados nuevos sin tocar los existentes: un `archived` terminal, o un
  `pending-review` entre `draft` y `active`, se insertan declarando sus transiciones y la operación
  que las ejecuta.
- **Capas ausentes** que un derivado puede añadir: `dependencies` y `http-clients` (si el precio o el
  stock pasan a venir de otro servidor) y `mail` (avisar al proveedor de un alta).
- **Piezas reutilizables en otro servicio**: los value types `SKU`, `Slug` y `Price`; el patrón de
  slug congelado al publicar; el patrón de galería (entidad hija con `position` + `primary` y su
  unicidad condicionada); y la separación de tres superficies —pública, gestión y M2M— por prefijo.
- **Jerarquía de categorías**: `Category` es plana a propósito, y añadir un `parent` es
  autorreferencia (`embed` la admite). Hay que decidir entonces si filtrar por el padre incluye a los
  hijos, y declarar la invariante de no crear ciclos.

### Supuestos y limitaciones

Asume una **tienda mediana**: miles de productos y decenas de miles de visitas al día. Las decisiones
están calibradas para eso —paginación offset, cachés de 60 s en las fichas, un índice por criterio de
filtro y coincidencia parcial sobre el nombre—. A escala de cientos de miles de productos el orden
alfabético con offset y el filtro por nombre dejan de sostenerse: harían falta paginación por cursor y
un servicio de búsqueda dedicado, y eso obliga a revisar decisiones ya tomadas, no solo a añadir.

Asume **una sola tienda** (no hay inquilinos: quien tiene el rol lo tiene sobre todo el catálogo),
**una sola divisa** (constante del despliegue, no un dato del producto) y **un solo idioma**.

Queda fuera **a propósito**:

- **Stock e inventario** — lo lleva otro servidor. Las existencias cambian a otro ritmo y por otros
  motivos, y meterlas aquí haría que cada venta escribiera en el catálogo.
- **Variantes de producto** (talla, color) — un producto es un artículo vendible con un `sku` y un
  precio. Una tienda de ropa las necesita, y es lo primero que habría que mirar al derivar: cambia el
  agregado.
- **Multi-divisa y multi-idioma** — internacionalizar exigiría que `name` y `description` fueran
  colecciones por locale, y que el filtro por rango de precio dijera en qué divisa filtra.
- **Búsqueda con relevancia** — el filtro por nombre es coincidencia parcial, no full-text. Para eso
  está `search-service`, que mantiene su índice con los eventos de este servicio.

### Cómo reutilizarlo

`keel describe catalog` da el resumen mecánico previo (identidad, estado, capas, contenido) y
`keel registry show catalog` su ficha publicada, con enlaces a esta documentación.

- **Si sirve tal cual** —una tienda de catálogo con marcas y categorías planas, divisa única y el
  stock en otro servidor—, se **adopta**: `keel registry get catalog` lo trae sin tocarlo, con sus
  derivados al día, y se va **directo a generar**. Es el camino que se espera para la mayoría de los
  casos: el diseño está cerrado y su contrato es completo.
- **Si hay que cambiarlo** —variantes, jerarquía de categorías, multi-divisa, el precio viniendo de
  otro servidor—, se **deriva**: `keel new <nuevo> --from registry:catalog` clona el spec con linaje
  `basedOn` y `/keel-design` entrevista solo sobre lo que cambia. Derivar para acabar usándolo sin
  cambios obliga a regenerar a mano todo lo que ya estaba hecho.

## 8. Cobertura de comportamiento

`specs/catalog/validation-scenarios.md` es el contrato de equivalencia entre implementaciones: **26
flujos** que cubren las 25 operaciones, los 32 códigos de error declarados, los diez eventos, los tres
estados del lifecycle y sus cuatro transiciones (con una transición inválida por operación), las dos
cachés (invalidación por cada vía **y** retención), la idempotencia de las cuatro operaciones que la
declaran (reintento secuencial **y** carrera), las dos garantías del outbox (canal indisponible y
evento abandonado), la paginación de los cuatro listados, la autorización por operación (`401`/`403`),
los dos escenarios de CORS y el contrato completo de la superficie M2M con credencial de máquina.
