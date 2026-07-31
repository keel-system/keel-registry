# catalog — Documento de diseño

> specs/catalog v0.4.0. Diseño cerrado; el porqué de las decisiones se entrevistó al cerrarlo.

## 1. Propósito y alcance

`catalog` mantiene el **catálogo comercial** de un negocio —productos, con su marca y su categoría—
como fuente de verdad para el resto del sistema. Sirve a tres públicos a la vez, y esa triple
audiencia es lo que da forma al diseño:

- un **equipo interno** que redacta fichas, sube fotografías, publica y retira productos;
- el **visitante anónimo** de una tienda web, que solo ve lo publicado;
- **otros servidores** (pedidos, facturación) que necesitan resolver un producto por su id o por
  lotes, y que mantienen una copia local alimentada por los eventos que este servicio publica.

Queda **fuera** a propósito: precios por cliente o por canal, promociones y descuentos, inventario y
stock, variantes de producto (talla, color), traducción de las fichas y cualquier noción de tienda u
organización — un despliegue sirve un catálogo.

## 2. Modelo de dominio

**Value types.** `SKU` (referencia comercial, `^[A-Z0-9][A-Z0-9-]{3,19}$`), `Slug` (identificador
legible para URLs), `Money` (value object `amount` + `currency`) y `ProductStatus`
(`draft` · `active` · `retired`) y `ActivationStatus` (`active` · `inactive`, compartido por marca y
categoría). Existen para que la regla viva en un sitio: el patrón del sku o la escala del importe no
se repiten campo a campo.

**Entidades.**

| Entidad | Papel | Campos notables |
|---|---|---|
| `Brand` | Marca comercial | `name` (único), `slug` (**computed** del name), `status` (lifecycle `active`↔`inactive`), `createdAt`/`updatedAt` (**generated**) |
| `Category` | Familia comercial, **plana** (sin jerarquía) | `name` (único), `slug` (**computed**), `status` (lifecycle `active`↔`inactive`) |
| `Product` | Artículo del catálogo | `sku` (único), `price` (`Money`), `status` (lifecycle), `tags` (lista ≤10, valores sin identidad) |
| `ProductImage` | Fotografía de la galería | `file` (bucket `productImages`), `position` (**computed**: última libre, se recompacta al borrar), `altText` |

No hay campos `sensitive`: todo lo que el catálogo guarda es información comercial destinada a
publicarse.

**Agregados.** Tres, y la frontera es deliberada:

- `Brand` y `Category` son raíces **solas**: cambian sin arrastrar a los productos que las
  referencian, y un producto las referencia por el id de la raíz.
- `Product` agrupa **`Product` + `ProductImage`**: la galería no tiene vida fuera de su producto —se
  añade, se reordena y se borra siempre en el contexto del producto, y la invariante "un producto
  publicado tiene al menos una imagen" solo es verificable si ambos cambian en la misma transacción.

**Ciclo de vida de `Brand` y `Category`.** `active ↔ inactive`, reversible en ambos sentidos y **sin
estado terminal**: una marca o categoría **nunca se elimina**. Desactivarla impide asignarla a
productos nuevos y la retira de la navegación pública, pero los productos que ya la referencian
conservan su estado y siguen publicados.

**Ciclo de vida de `Product`.** `draft → active → retired`, con `retired` **terminal**:

| Transición | La ejecuta | Emite |
|---|---|---|
| (alta) → `draft` | `createProduct` | `ProductCreated` |
| `draft → active` | `publishProduct` | `ProductUpdated` |
| `draft → retired` | `retireProduct` | `ProductRetired` |
| `active → retired` | `retireProduct` | `ProductRetired` |

## 3. Invariantes y reglas clave

- Un producto `active` tiene `price.amount > 0` **y** al menos una imagen.
- Un producto `retired` no admite cambios en su ficha ni vuelve a un estado anterior.
- El `sku` es único entre **todos** los productos, **incluidos los retirados**: nunca se reutiliza.
- Todo producto referencia una marca y una categoría existentes; como ninguna de las dos se elimina
  jamás, la referencia **nunca queda colgante** —y eso no depende de ninguna comprobación previa al
  borrado, que con frontera `per-aggregate` no sería fiable.
- Un producto solo puede crearse o reasignarse hacia una marca y una categoría `active`; conservar las
  que ya tenía es válido aunque se hayan desactivado.
- Las posiciones de la galería son consecutivas desde 1 y sin repetir; la de posición 1 es la
  principal. Al borrar una imagen, las restantes se recompactan.
- La unicidad de `name` y el filtro `q` comparan **ignorando mayúsculas, minúsculas y acentos**:
  `ACME`, `acme` y `Ácme` son el mismo nombre.
- Los `update*` son **reemplazo total**: un campo opcional omitido queda sin valor.

## 4. Qué hace

**Gestión (usuarios autenticados).** Marcas y categorías se crean, se editan, se consultan, se listan
y se **desactivan o reactivan** (`deactivateBrand`/`activateBrand` y sus homólogas de categoría). No
hay operación de borrado: es lo que garantiza que ningún producto quede apuntando al vacío. Los productos se manejan con operaciones **nombradas por su intención**, no
como updates genéricos: `createProduct`, `updateProduct`, `publishProduct`, `retireProduct`,
`addProductImage`, `removeProductImage`, `reorderProductImages`, más `getProduct` y `listProducts`,
que ven el catálogo en **cualquier** estado. Todas las lecturas devuelven `brand` y `category`
**resueltas como objetos anidados** —nunca un `brandId` suelto—, con la misma forma en las trece.

`createProduct` es **idempotente** por clave de cliente (`Idempotency-Key`, 24 h).

**Catálogo público (sin credencial).** Cuatro operaciones, que son toda la superficie de la tienda:

- `getPublishedProduct` y `listPublishedProducts` ven **solo** productos `active`; un producto en
  `draft` o `retired` responde `404`, no `403`. Ambas traen `brand` y `category` anidadas para que la
  tienda pinte la tarjeta sin una segunda llamada por producto. `getPublishedProduct` se sirve de una
  **caché de 300 s** invalidada por `ProductUpdated` y `ProductRetired`; `listPublishedProducts` no
  lleva caché.
- `listPublishedBrands` y `listPublishedCategories` alimentan la **navegación**: devuelven únicamente
  las marcas y categorías **`active`** con al menos un producto `active`, sin `createdAt`/`updatedAt`
  y **sin caché**. Sin ellas la tienda podría filtrar por `brandId`/`categoryId` pero no descubrirlos.

**Superficie servidor-a-servidor.** Contrato propio, con operaciones y endpoints propios —nunca
compartidos con los de usuarios:

| Operación | Endpoint | Qué promete |
|---|---|---|
| `getProductForServices` | `GET /api/v1/internal/products/{productId}` | Un producto por id, en **cualquier** estado, con `brand` y `category` ya resueltos como objetos anidados. Caché de 300 s |
| `getProductsByIds` | `POST /api/v1/internal/products/batch` | De 1 a 100 ids en una llamada. Devuelve los encontrados **en el orden pedido** y **omite** los inexistentes, sin error |

Ambas exigen credencial de máquina con scope `product:read` y audiencia `catalog`. El consumidor
declarado es `order-service`.

## 5. Fronteras e integraciones

**Eventos publicados** (canal lógico `productEvents`): `ProductCreated`, `ProductUpdated` y
`ProductRetired`. El payload es un **snapshot completo** —el consumidor se pone al día sin volver a
llamar—, salvo por un detalle deliberado: viaja `brandId`/`categoryId`, **nunca** `brandName` ni
`categoryName`. `ProductRetired` es la **vía de baja** de toda copia local. No hay suscripciones: este
servicio no consume eventos de nadie.

**Persistencia**: modelo relacional; `Product` con clave natural `sku`, `Brand` y `Category` con
`slug`, `ProductImage` con `(productId, position)`. Índices derivados de las queries reales del
catálogo público y del listado de gestión.

**Almacenamiento**: un bucket lógico `productImages`, `public`, `image/png`/`image/jpeg`/`image/webp`,
5 MB por archivo. Al ser `public`, cada campo `file` se proyecta en las respuestas y en los eventos
como la **URL absoluta de lectura** del objeto —la tienda carga la foto directamente, sin pasar por el
servicio—; el host base es configuración de despliegue. Esa proyección no es un campo del DSL sino una
**convención de equivalencia**, y como tal está fijada en `validation-scenarios.md`, que es lo que
obliga a todo generador a producir la misma forma.

**Acceso**: OIDC para usuarios, `client-credentials` con validación de audiencia para máquinas. Dos
roles con mínimo privilegio —`catalog-admin` (todo) y `catalog-editor` (redacta fichas, **no** publica
ni toca marcas y categorías)— sobre un catálogo único de permisos que sirve a la vez de catálogo de
scopes. `cors` declarado porque la tienda y el back-office llaman desde el navegador; los orígenes
concretos son despliegue, no diseño.

## 6. Decisiones de diseño (qué / por qué)

| Decisión | Qué se eligió | Por qué | Alternativa descartada |
|---|---|---|---|
| **Fiabilidad de publicación** | `best-effort` | El diseñador acepta que un evento se pierda si el broker está caído en el instante del commit; `getProductsByIds` sirve de vía de resincronización cuando el consumidor la necesita | `outbox` (recomendado por el agente): ningún evento se pierde, a costa de arrastrar la escritura del evento a la transacción |
| **Baja de producto** | Retirada lógica (`retired`) + `ProductRetired` | Conserva el histórico para pedidos y facturas ya emitidos, y da al consumidor una señal de baja limpia | Borrado físico con `ProductDeleted`: un pedido antiguo referenciaría un id inexistente |
| **`sku` de un retirado** | Queda **reservado para siempre** | El sku identifica al producto en documentos ya emitidos; reciclarlo haría que un histórico apunte a otro artículo | Liberarlo al retirar: permitiría reponer una referencia descatalogada, a costa de que el mismo sku signifique dos cosas según la fecha |
| **Lote M2M con ids ausentes** | Resultado **parcial**, sin error | Un lote que falla entero por un id borrado obliga al consumidor a reintentar uno por uno | Fallar con `404`; devolver además la lista de ausentes |
| **Retirada de marca/categoría** | **No se borran: se desactivan** (`active ↔ inactive`) | Con frontera `per-aggregate`, comprobar "no tiene productos" y borrar caen en transacciones distintas: un `createProduct` concurrente colaría un producto apuntando a una marca ya borrada. Sin borrado, la referencia nunca puede quedar colgante — la integridad deja de depender de una carrera | Bloquear el borrado con `BRAND_IN_USE`/`CATEGORY_IN_USE` (era el diseño hasta v0.2.0, con esa carrera abierta); bajar la frontera a `per-operation`; borrar en cascada |
| **Efecto de desactivar** | Los productos existentes siguen publicados; la marca desaparece del menú de la tienda | Desactivar es una decisión de **surtido futuro**, no una retirada de catálogo: lo ya publicado sigue vendiéndose, pero deja de ofrecerse como vía de navegación | Que no cambie nada para el visitante (entonces "inactiva" no sería observable); retirar en cascada sus productos, con un `ProductRetired` por cada uno |
| **Caché y renombrado de marca** | *Aceptado*: hasta 300 s con el nombre viejo | La ficha cacheada lleva `brand`/`category` embebidos y no hay eventos de marca ni categoría con los que invalidarla. Renombrar es raro y el desfase es acotado | Añadir `BrandUpdated`/`CategoryUpdated` al `invalidatedBy`; quitar la caché |
| **Nombres de marca en los eventos** | Solo `brandId`/`categoryId` | Un renombrado dejaría el nombre rancio en cada copia local sin forma de detectarlo; el consumidor resuelve el nombre por la superficie M2M, que lo devuelve incorporado | Eventos propios de marca y categoría; reemitir un `ProductUpdated` por producto afectado; aceptar el rancio |
| **Idempotencia de `createProduct`** | `client-key`, 24 h | Un reintento por red perdida no debe crear un duplicado que además ya se replicó por evento | Sin idempotencia (el sku único frena el duplicado, pero el reintento legítimo se ve como error); `payload-hash` |
| **Clave de idempotencia reutilizada** | `IDEMPOTENCY_KEY_REUSED` (`409`) | Devolver el resultado anterior le haría creer al cliente que creó lo que mandó | Devolver el resultado original en silencio |
| **Caché de las lecturas** | 300 s, invalidada por `ProductUpdated` y `ProductRetired` | El catálogo se lee mucho más de lo que se escribe, y la invalidación acota el rancio | Sin caché (todo el pico del catálogo público cae en la base de datos); cachear solo la lectura pública |
| **Concurrencia** | `optimisticLocking: all` → `CONCURRENT_MODIFICATION` (`409`) | El producto tiene máquina de estados y precio: perder una edición ajena en silencio tiene coste real | Último escritor gana: la edición del primero desaparece sin que nadie lo sepa |
| **Frontera transaccional** | `per-aggregate` | Elección explícita del diseñador sobre los tres agregados declarados; ninguna operación toca dos, así que no hay cambio que pueda confirmar a medias | `per-operation` (recomendado por el agente, dado que ninguna operación cruza agregados) |
| **Publicación en el evento** | `ProductUpdated` con `status: active` | Menos contrato que mantener; el snapshot ya lleva el estado y el consumidor filtra | Evento propio `ProductPublished` |
| **Superficie M2M** | Operaciones y endpoints propios | `api.endpoints` se indexa por operación: compartirlo sería compartir output, errores, paginación y scopes entre dos contratos que crecen en direcciones opuestas | `audience: both` sobre las operaciones de usuarios |
| **Visibilidad del bucket** | `public` | Son fotos de catálogo público servidas por la web sin mediación del servicio | `private` + URL firmada: exigiría una operación que la emita y una llamada por foto |
| **Galería de imágenes** | Entidad hija `ProductImage` con `position`, máximo 8 | El DSL veta `list: true` sobre `type: file`, y una imagen tiene identidad propia (se borra y se reordena por id). `position: 1` designa la principal con un solo concepto | Marca `primary` independiente del orden; conjunto sin orden; galería sin cota |
| **Última imagen de un publicado** | Se impide (`LAST_IMAGE_OF_ACTIVE_PRODUCT`, `409`) | Sostiene siempre la invariante "un `active` tiene al menos una imagen" | Despublicar automáticamente (un borrado de foto retiraría el producto sin que nadie lo pidiera); permitir fichas publicadas sin foto |
| **Navegación de la tienda** | Operaciones públicas propias (`listPublishedBrands`, `listPublishedCategories`), filtradas a las que tienen catálogo publicado | La tienda podía filtrar por marca y categoría pero no descubrirlas; y una faceta sin productos publicados es un enlace muerto en el menú | Abrir `listBrands`/`listCategories` al público (compartiría output y paginación con el back-office); exponerlas todas sin filtrar; añadir un contador de productos |
| **Caché de la navegación** | **Sin caché** | Lo que muestran depende del nombre de la marca y de si conserva algún producto `active`, y **no hay eventos de marca ni de categoría** que puedan invalidarla: una caché serviría un menú desfasado tras cada renombrado, sin vía de invalidación | Caché de 300 s invalidada por eventos de producto (no cubriría los renombrados); caché larga de 3600 s |
| **Autorización a nivel de dato** | Catálogo único: cualquier editor edita cualquier producto | Es un catálogo corporativo único; queda dicho en voz alta en vez de asumido | Editor adscrito a marcas: exigiría modelar la asignación editor–marca y filtrar los listados |
| **Semántica de los `update*`** | Reemplazo total (omitir = dejar sin valor) | Son `PUT`; la fusión no dejaría forma de borrar una `description` sin un convenio extra | Fusión: los campos ausentes conservan su valor |
| **Colisión de slug** | Error propio `SLUG_ALREADY_EXISTS` (`409`) | Es un fallo distinto del de nombre y se arregla distinto: cambiando el nombre | Desambiguar con sufijo (`acme-2`); reusar el error de nombre |
| **`retired` sin vuelta atrás** | *Aceptado* | Se aceptó no añadir `reactivateProduct`: una retirada por error obliga a crear otro producto, y su sku ya no se puede reusar | Operación de reactivación `retired → draft` |
| **Retención de retirados** | *Aceptado* | Se aceptó que los productos retirados se acumulen sin purga: todo listado de gestión los arrastra indefinidamente | Operación programada de purga a los N meses |
| **Proyección de `brand`/`category`** (v0.4.0) | **Uniforme**: las 13 operaciones que devuelven `Product` la resuelven con `embed` | Hasta v0.3.0 los dos listados devolvían `brandId`/`categoryId` planos mientras las otras once embebían. No era una decisión: `validation-scenarios.md` ya exigía los objetos anidados en ambos listados, así que el spec contradecía a su propio contrato de validación, y el desfase no habría salido hasta fallar la suite de integración. Un listado con id obliga al cliente a N+1 llamadas por página, justo donde más caro sale | Dejar los listados con `brandId`/`categoryId` planos y embeber solo en el detalle: es la única forma de aligerar un listado que el DSL admite, y cuesta N+1 llamadas por página al cliente |
| **Forma de la referencia embebida** (v0.4.0) | **No se elige**: el objeto embebido lleva los campos propios del agregado sin sus relaciones, igual en las trece operaciones | DSL 2.3 dejó fuera a propósito elegir los campos del objeto embebido, por ser "una proyección arbitraria, no un hecho del dominio", así que el diseño no tiene palanca — y el workspace no dobla el DSL para acomodar a un diseño. Al descubrirlo se vio que los derivados de v0.3.0 se contradecían: `openapi.yaml` documentaba `{id, name, slug}` y un escenario público asertaba `brand.status`. La ambigüedad se cerró a favor de la semántica del DSL y quedó fijada como convención de equivalencia en `validation-scenarios.md` | Recortar con `exclude` de dot-path (consigue el efecto, pero por la puerta de atrás de una decisión que el DSL tomó a propósito); pedir un primitivo de proyección al DSL |
| **`brand.status` visible sin credencial** (v0.4.0) | *Aceptado* | `listPublishedProducts` y `getPublishedProduct` son públicos y la proyección embebida no se puede recortar, así que el visitante puede ver que la marca de un producto publicado está `inactive`. Son datos comerciales sin sensibilidad, y el listado no expone nada que el detalle público no expusiera ya desde v0.3.0 | Volver a `brandId`/`categoryId` planos en el listado público: evitaría la exposición, a costa de reintroducir el N+1 por página en el escaparate |
| **Versionado del contrato M2M** | *Aceptado* | Se aceptó no comprometer una política de solape ante un cambio incompatible; el consumidor no sabe de antemano cuánto aviso tendrá | Declarar en `INTEGRATION.md` el compromiso de `/api/v2` con N meses de solape |

## 7. Ficha de reutilización: evolucionar o derivar

### Contrato estable vs adaptable

**Estable** (cambiarlo rompe a alguien): los códigos de error (`SKU_ALREADY_EXISTS`,
`PRODUCT_NOT_FOUND`, `BRAND_INACTIVE`, `MAX_IMAGES_REACHED`…), los nombres de evento en pasado
(`ProductCreated`, `ProductUpdated`, `ProductRetired`) y la forma de su `data`, los endpoints
publicados —muy en especial los dos `audience: services`—, el canal `productEvents`, y los roles,
permisos y scopes.

**Adaptable** sin romper a nadie: las `rules` y `preconditions` de los casos de uso, el TTL de la
caché y su `invalidatedBy`, la ventana de idempotencia, las cotas del bucket (`maxSizeMb`,
`allowedContentTypes`), los índices sugeridos, el tamaño de página y el máximo de imágenes.

Versionado del spec según `docs/methodology.md`: **patch** para prosa y reglas internas, **minor**
para operaciones, campos opcionales y errores nuevos, **major** para quitar o renombrar un código de
error, un evento, un endpoint o un campo del payload.

### Puntos de extensión típicos

- **`lifecycle`**: hay sitio natural para `reactivateProduct` (`retired → draft`) o un estado
  intermedio de revisión entre `draft` y `active`. El patrón `active ↔ inactive` de marca y categoría
  es directamente reutilizable para cualquier catálogo auxiliar que se añada.
- **Contador de facetas**: `listPublishedBrands`/`listPublishedCategories` son el sitio natural para
  añadir el número de productos publicados de cada una, si el menú lo necesita.
- **`Category` plana** es el punto de extensión más evidente: añadir `parent` (auto-referencia
  `many-to-one`) la convierte en árbol, con sus invariantes de ciclo y profundidad.
- **Capas ausentes**: `dependencies` y `http-clients`. Un derivado que necesite enriquecer la ficha
  desde un PIM o un proveedor de precios las añade sin tocar lo existente.
- **Proyección de la referencia embebida**: hoy la fija el DSL y no se puede estrechar. Un derivado
  que necesite una cara pública más magra tiene dos vías, ninguna dentro de `embed`: quitar el
  `embed` de las operaciones públicas y asumir el N+1, o esperar a que el DSL incorpore un primitivo
  de proyección. Cambiar el generador para que recorte **no** vale: dejaría de ser equivalente.
- **Piezas reutilizables en otro servicio**: los value types `SKU`, `Slug` y `Money`; el patrón
  galería (entidad hija con `position` recompactada); el par lectura pública filtrada por estado +
  lectura M2M sin filtrar; y el par `get` + `getByIds` como superficie de resincronización.

### Supuestos y limitaciones

- **Un solo tenant**: un despliegue sirve **un** catálogo corporativo. No hay organización, tienda ni
  espacio de nombres, y la autorización lo asume (cualquier editor edita todo).
- **Volumen de decenas de miles de productos**, no millones. El diseño pagina y acota, pero el filtro
  `q` es coincidencia parcial sobre `name`/`sku`/`tags`, **no** un motor de búsqueda: ningún índice
  sostiene esa coincidencia, y a partir de cierto tamaño hace falta un servicio de búsqueda aparte.
- **Moneda única de facto**: el modelo admite `currency` por producto, pero el servicio no convierte
  ni compara importes entre monedas, así que un catálogo multimoneda necesita esa lógica fuera.
- **Sin internacionalización**: `name` y `description` existen en un solo idioma. Traducir la ficha
  exige modelar el idioma, que este diseño no cubre.
- **Sin variantes de producto** (talla, color) ni **inventario**: una talla distinta es aquí un
  producto distinto con su propio sku.
- **Consistencia eventual con los consumidores**: con `best-effort`, la copia de otro servidor puede
  divergir tras una caída del broker hasta que se resincronice por `getProductsByIds`.
- **Los datos no se borran nunca**: ni productos (se retiran) ni marcas o categorías (se desactivan).
  El diseño no tiene purga ni archivado, así que el volumen solo crece.
- **El nombre de marca y categoría puede verse desfasado hasta 300 s** en la ficha pública y en la M2M
  tras un renombrado, porque no hay eventos de marca ni categoría que invaliden la caché.

### Cómo derivar

```bash
keel describe catalog                     # identidad, capas, contenido y frescura de los derivados
keel new mi-catalogo --from catalog       # clona el diseño y estampa el linaje en service.basedOn
/keel-design specs/mi-catalogo            # arranca en modo derivación: entrevista solo lo que cambia
```

Desde el registry, sin clonar el repo: `keel new mi-catalogo --from registry:catalog`.
