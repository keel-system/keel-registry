# Contribuir un diseño al registry

Este repositorio publica **diseños reutilizables**, no implementaciones. Un diseño que entra aquí lo va a
descargar alguien que no habló contigo y va a generar un servidor a partir de él: el listón está puesto para
que eso funcione sin preguntarte nada.

## El recorrido, de principio a fin

Los once pasos, en orden. Lo que cada uno tiene que cumplir está detallado en las secciones siguientes.

### Preparación

**1. Instala la CLI y clona el repo.** Este registry **es un workspace Keel** (`specs/`, `docs/`, `schema/`,
`templates/`), así que al clonarlo ya tienes el payload: **no ejecutes `keel init`** — ver
[Compatibilidad con la herramienta](#compatibilidad-con-la-herramienta).

```bash
npm i -g keel-core          # deja disponible el comando `keel`
git clone https://github.com/keel-system/keel-registry && cd keel-registry
git checkout -b add-<slug>
```

**2. Mira si el problema ya está resuelto.** `keel registry` y `keel registry search <dominio>`. Si ya existe
algo cercano, lo tuyo es probablemente una **variante** y no un diseño nuevo: eso cambia el paso 3 y te obliga
a justificar en el PR por qué no vale la hermana que ya está.

### El diseño

**3. Crea el spec.**

```bash
keel new <slug>                                   # diseño nuevo
keel new <familia>-<variante> --from <origen>     # variante de uno que ya existe
```

La segunda forma estampa el linaje en `service.basedOn` y lo deja auditable. Ver
[Variantes del mismo problema](#variantes-del-mismo-problema).

**4. Diséñalo.** `/keel-design specs/<slug>`, que entrevista capa a capa. Si el servicio consume otros
servidores, su bloque de integración invoca `/keel-consume`. Al cerrar produce `validation-scenarios.md` y
ejecuta `/keel-handoff` (el `DESIGN.md`).

**5. Valida.** `/keel-validate specs/<slug>` — la CLI (schemas por capa + referencias cruzadas) más la
revisión semántica de calidad. Tiene que quedar verde **sin `--wip`**.

**6. Genera los derivados.** `/keel-docs specs/<slug>` y, si el diseño expone superficie servidor-a-servidor,
`/keel-integrate specs/<slug>`. Comprueba con `keel describe <slug>` que no queda ninguno `stale`,
`unstamped` ni `orphan`.

**7. Escribe `specs/<slug>/design.yaml`.** Los metadatos de publicación. Ver
[El sidecar `design.yaml`](#el-sidecar-designyaml).

**8. Regenera el índice.** `keel index`, que reescribe la tabla del `README.md` entre sus marcadores y el
`index.json`. **Nunca a mano**: es el único escritor de esa región.

### El envío

**9. Commitea todo junto**: `specs/<slug>/` (capas + `design.yaml` + `validation-scenarios.md`),
`docs/<slug>/`, y el `README.md` y el `index.json` regenerados.

**10. Abre el PR** con lo que pide [El pull request](#el-pull-request).

**11. La CI ejecuta tres puertas**: `keel validate` sobre cada diseño, `keel index --check` y
`keel init --check`. Las tres son de solo lectura y sus mensajes dicen qué arreglar.

> **Los dos tropiezos más frecuentes.** Que `service.name` no coincida con el nombre del directorio (punto 5
> del listón: los derivados viven en `docs/<service.name>/` y dos diseños se pisarían), y suponer que la CI
> valida con tu CLI: valida con una **versión pineada**, así que un diseño puede pasar en tu máquina y fallar
> en el PR.

## El listón

Es lo que produce el recorrido de arriba, y lo que revisa el PR. Un diseño se acepta cuando cumple **todo**
esto:

1. **`keel validate specs/<slug>` en verde, sin `--wip`.** Sin capas en estado plantilla, sin referencias
   cruzadas roscadas, sin `description` que empiece por `TODO`.
2. **Escenarios de validación cerrados.** `specs/<slug>/validation-scenarios.md` con los flujos `FL-*` en
   Given/When/Then. Son el contrato de equivalencia: lo que hace que un servidor generado de este diseño se
   pueda declarar correcto en cualquier stack. Un diseño sin escenarios no es reutilizable, es un borrador.
3. **Derivados al día.** `keel describe <slug>` no reporta ningún derivado `stale`, `unstamped` ni `orphan`.
   Como mínimo `DESIGN.md` (de `/keel-handoff`); y si el diseño tiene capa `api` o `messaging`, también los
   contratos formales y el panel de `/keel-docs`.
4. **`design.yaml` presente y válido** contra `schema/design.schema.json` (ver más abajo).
5. **`service.name` idéntico al nombre del directorio.** Los derivados viven en `docs/<service.name>/`: si
   el nombre no coincide con el slug, dos diseños se pisan los derivados. `keel index` lo avisa.
6. **Identificadores en inglés, prosa en español.** Todo nombre del DSL (types, entidades, campos,
   operaciones, eventos, errores, roles, canales, buckets) en inglés; las `description`, la documentación y
   los comentarios, en español. Es la regla canónica de la metodología, y los generadores derivan de esos
   nombres los directorios, clases y tablas del código.
7. **Cero tecnología en el diseño.** Ni framework, ni ORM, ni broker, ni proveedor de auth, ni motor de base
   de datos concreto. Si tu diseño solo tiene sentido con Postgres o con Kafka, todavía no es un diseño.
8. **`keel index --check` en verde**, es decir: `keel index` ejecutado y su resultado commiteado (paso 8 del
   recorrido).

## El sidecar `design.yaml`

Los metadatos de publicación van en `specs/<slug>/design.yaml`, fuera del DSL: el manifiesto describe el
servicio, no cómo se distribuye. Se valida con `schema/design.schema.json`.

```yaml
family: notifications        # agrupa variantes del mismo problema; por defecto, el propio slug
variant: multichannel        # obligatorio cuando family ≠ slug
summary: >                   # una o dos líneas: qué resuelve. Alimenta el índice
  Notificaciones multicanal con plantillas versionadas y entrega garantizada.
differsIn: >                 # solo en familias con varias variantes: qué la distingue de sus hermanas
  Añade SMS y push sobre email, con outbox y reintentos por canal.
maturity: reference          # draft | stable | reference
tags: [email, sms, push, outbox, templates]
author: tu-usuario
license: Apache-2.0
requires: [catalog]          # otros diseños del registry que este consume (coherente con la capa dependencies)
```

Sobre `maturity`: empieza en `draft`. `stable` es para diseños que ya sostuvieron una implementación real.
`reference` lo reservan los mantenedores para los diseños que sirven de muestra de la metodología.

**No inventes tags.** Etiqueta lo que el diseño declara de verdad: si pones `outbox`, la capa `messaging`
debe declarar `reliability: outbox`; si pones `cache`, alguna operación debe declarar `cache`.

## Variantes del mismo problema

Varios diseños pueden resolver el mismo dominio de formas distintas —`notifications` solo email, multicanal,
o solo push— y eso es deseable: quien descarga elige la que encaja en vez de recortar la grande.

Las variantes son **directorios planos** con slug `<familia>-<variante>`, y la familia se declara en el
sidecar:

```
specs/notifications-multichannel/    family: notifications, variant: multichannel
specs/notifications-email-digest/    family: notifications, variant: email-digest
specs/notifications-push-only/       family: notifications, variant: push-only
```

`keel index` las agrupa en una subtabla comparativa, ordenadas de más madura a menos, usando `differsIn`
como columna de diferencias. Si tu variante deriva de otra del registry, créala con
`keel new <nueva> --from <origen>`: el linaje queda estampado en `service.basedOn` y se puede auditar.

## El pull request

1. Un diseño (o una variante) por PR.
2. **Pega la salida de `keel describe <slug>` en la descripción del PR.** Es la revisión en un vistazo:
   identidad, estado, capas, contenido y frescura de los derivados.
3. Explica en dos líneas **para quién es** el diseño y, si es una variante, **por qué no vale la
   hermana** que ya existe.

La CI ejecuta `keel validate` sobre cada diseño, `keel index --check` y `keel init --check`. Si falla, el
mensaje dice qué arreglar.

## Compatibilidad con la herramienta

Este repositorio y [keel-system/keel](https://github.com/keel-system/keel) evolucionan por separado, y para
que eso funcione el CI valida contra una **versión pineada** de la herramienta, no contra su última versión.
Consecuencias prácticas:

- **Un diseño no puede usar un DSL más nuevo que el que entiende la versión pineada.** Si lo hace, `keel
  validate` lo rechaza en el PR. Hay que subir el pin primero, y eso es un cambio deliberado de los
  mantenedores (variable de repo `KEEL_TOOL_REF`, o el valor por defecto en `.github/workflows/ci.yml`).
- **Un cambio incompatible en la herramienta no rompe tu PR.** Por eso está el pin. Quien avisa de que la
  herramienta se ha adelantado es `.github/workflows/compat.yml`, un job semanal que corre lo mismo contra
  `main` de la herramienta **sin bloquear a nadie**.
- **La CLI que valida es la instalada, no la copia de este repo.** `schema/`, `templates/` y `docs/dsl/` son
  copias del payload de la herramienta, útiles para leer y para el autocompletado del editor, pero `keel
  validate` usa siempre los schemas de su propio paquete. Que no se queden atrás lo comprueba
  `keel init --check`.
- **No ejecutes `keel init --force` en tu rama.** Es el remedio correcto en tu propio workspace, pero aquí el
  payload tiene que corresponder a la **versión pineada**, no a la CLI que tengas instalada: si la tuya va por
  delante, ese `--force` deja el repo desalineado con el pin y la puerta `keel init --check` se pone en rojo
  en tu PR. Sincronizar el payload es tarea de los mantenedores, y va junto con subir el pin. (Si lo
  ejecutaste sin querer: `git checkout -- schema templates docs/dsl docs/*.md .claude`; tus `specs/` y
  `docs/<slug>/` no forman parte del payload y no se tocan, igual que `README.md` y `CLAUDE.md`, que
  `--force` conserva.)

Si trabajas en local con una CLI más nueva que el pin, tu diseño puede validar en tu máquina y fallar en el
PR. La versión que usa el CI se ve en el propio workflow.

## Cambiar un diseño que ya está publicado

Con `/keel-evolve specs/<slug>`, nunca editando el spec a mano. Versiona el contrato (patch/minor/major
según lo que rompa) y **regenera en cascada todos los derivados**, que llevan estampada la versión de la que
nacieron. Un diseño publicado cuyos derivados quedaron atrás miente a quien lo descargue, y `keel describe`
lo delata.
