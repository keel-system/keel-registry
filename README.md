# Keel Registry

**Diseños de servidores listos para reutilizar.** Elige uno, tráetelo —tal cual o ajustándolo— y genera
tu implementación en la tecnología que quieras.

Cada entrada de este repositorio es un **diseño Keel** completo: un directorio `specs/<diseño>/` con un
artefacto YAML declarativo por capa —dominio, casos de uso, API, seguridad, mensajería, persistencia,
storage— más sus escenarios de validación, y un `docs/<diseño>/` con la ficha de decisiones, los contratos
formales (OpenAPI/AsyncAPI), colecciones Postman y un panel visual. Ninguno menciona framework, ORM, broker
ni lenguaje: eso se decide al generar.

> **Es un repositorio git, no un servicio.** No hay API ni `keel publish`: se contribuye por pull request y
> se consume clonando o con `keel registry`. El índice de abajo y `index.json` los genera `keel index`.

## Cómo reutilizar un diseño

Primero **descubrir**, contra este repo y sin clonarlo:

```bash
keel registry                       # todos los diseños, agrupados por familia
keel registry search notifications  # filtra por nombre, tags, dominio o descripción
keel registry show catalog          # la ficha completa de uno
```

Después hay **dos formas de traértelo**, y la diferencia no es de comodidad sino de intención: si el
diseño te sirve tal cual, no hay nada que rediseñar.

### ¿Te sirve tal cual? Adóptalo

```bash
keel registry get catalog
```

Trae el diseño **sin tocarlo** —mismo nombre, misma versión, misma `description`— y sus derivados
publicados en `docs/catalog/`: la ficha de decisiones, los contratos OpenAPI/AsyncAPI, las colecciones
Postman y el panel. Como nacen con la misma versión que documentan, `keel describe catalog` los ve al día
y puedes ir directo a generar:

```bash
keel-spring build specs/catalog     # y después, dentro del proyecto, /keel-generate-spring
```

No se copia `specs/<diseño>/design.yaml`: son los metadatos con los que el diseño se publica **aquí**
(autor, licencia, madurez), y en tu workspace te presentarían como su publicador. Sí se estampa
`service.basedOn` aunque el nombre y la versión coincidan, para que quede constancia de que ese diseño
*es* `catalog@0.3.0`. Si más adelante necesitas cambiarlo, no es una derivación: es una evolución normal,
y entra por `/keel-evolve`.

### ¿Hay que cambiarlo? Derívalo

```bash
keel new mi-catalogo --from registry:catalog
```

Clona **solo el spec** —las capas y `validation-scenarios.md`—, estampa el linaje en `service.basedOn`,
deja la `description` marcada como pendiente de revisar y **renombra el servicio al nombre que tú elijas**:
el slug del registry no llega a tu código. Los derivados de `docs/` no se traen, porque describen al
servicio de origen y se regeneran al cerrar tu diseño. A partir de ahí, `/keel-design` arranca en modo
derivación y entrevista solo sobre lo que cambia.

### Sin la CLI a mano

El camino manual es equivalente. Para adoptar, copiar las dos carpetas:

```bash
git clone https://github.com/keel-system/keel-registry
cp -r keel-registry/specs/catalog specs/ && rm specs/catalog/design.yaml
cp -r keel-registry/docs/catalog  docs/
keel index
```

Para derivar, la CLI resuelve el origen por ruta:

```bash
keel new mi-catalogo --from ../keel-registry/specs/catalog
```

## Diseños disponibles

Cada fila enlaza el diseño (`specs/`) y, cuando existen, su **ficha de decisiones** (`DESIGN.md`), su
**panel visual** (`overview.html`) y su **contrato servidor-a-servidor** (`INTEGRATION.md`). Las familias
con varias variantes remiten a una subtabla que las compara.

<!-- keel:servicios:start -->
| Diseño | Dominio | Madurez | Resumen | Capas | Documentación |
|---|---|---|---|---|---|
| [`catalog`](specs/catalog/) | commerce | borrador | Catálogo comercial de productos con su marca y su categoría: gestión interna con publicación y retirada, catálogo público de solo lectura, y superficie servidor-a-servidor (ficha por id y por lote) más eventos de alta, cambio y baja para que otros servidores mantengan su copia al día. | `domain`, `use-cases`, `api`, `security`, `messaging`, `persistence`, `storage` | [diseño](docs/catalog/DESIGN.md) · [panel](docs/catalog/overview.html) · [integración](docs/catalog/INTEGRATION.md) |
<!-- keel:servicios:end -->

**Madurez**: `referencia` = diseño ejemplar, mantenido como muestra de la metodología · `estable` = usable
en producción · `borrador` = en construcción, puede cambiar.

## Contribuir un diseño

Se aceptan diseños nuevos y variantes de los existentes. El listón y el proceso están en
[CONTRIBUTING.md](CONTRIBUTING.md); en resumen: el diseño pasa `keel validate` sin `--wip`, trae sus
escenarios de validación cerrados, sus derivados al día y un `design.yaml` con los metadatos del registry.

Si tienes permiso de merge, [MAINTAINERS.md](MAINTAINERS.md) cubre lo que no hace un colaborador: subir el
pin de la herramienta, refrescar el payload, promover la madurez de un diseño y aceptar uno de la comunidad.

## Sobre este repositorio

Es un **workspace Keel** sembrado con `keel init`, así que la CLI funciona aquí tal cual:

```bash
keel validate specs/catalog     # schemas por capa + referencias cruzadas
keel describe catalog           # identidad, capas, contenido y frescura de los derivados
keel index                      # regenera la tabla de arriba y index.json
keel index --check              # comprueba, sin escribir, que el índice está al día (esto corre en CI)
```

`schema/`, `templates/`, `docs/dsl/` y `.claude/skills/` son el payload de la CLI, no contenido del
registry: se refrescan con `keel init --force` al actualizar `keel-core`. `services/` está en `.gitignore`:
aquí se publican diseños, no implementaciones.

La herramienta y la metodología viven en [keel-system/keel](https://github.com/keel-system/keel).
