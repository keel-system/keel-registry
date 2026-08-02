# Mantener el registry

Este documento es para quien tiene permiso de merge. Lo que un colaborador necesita está en
[CONTRIBUTING.md](CONTRIBUTING.md); aquí está lo que **solo** hacen los mantenedores: mover el pin de la
herramienta, refrescar el payload, promover la madurez de un diseño y aceptar uno de la comunidad.

## Por qué el registry va por detrás a propósito

Este repositorio y [keel-system/keel](https://github.com/keel-system/keel) evolucionan por separado. La
dependencia es **unilateral y declarada**: `ci.yml` valida contra una versión pineada de la herramienta
(`vars.KEEL_TOOL_REF`, o el valor por defecto del workflow), nunca contra su última versión.

Eso significa que un cambio incompatible en la herramienta **no puede tumbar los PR de la comunidad**. El
precio es que la incompatibilidad se descubriría tarde, al intentar subir el pin, y por eso existe
[`compat.yml`](.github/workflows/compat.yml): un canario semanal (lunes 06:00 UTC, y `workflow_dispatch` para
lanzarlo a mano) que corre las tres puertas contra `main` de la herramienta, en frío y **sin bloquear a
nadie**. Su resumen dice si el pin se puede subir y, si no, qué falló.

**Nadie tiene que acordarse de vigilar esto.** El canario avisa; la tarea empieza cuando se pone rojo (o
cuando se quiere adoptar algo nuevo de la herramienta).

Hay una sola excepción a la dirección de la dependencia, y va al revés — ver
[El formato del índice](#el-formato-del-índice).

> **Estado actual: sin pin.** La herramienta todavía no publica ninguna etiqueta, así que el valor por
> defecto de `ci.yml` es `main`. Todo lo de arriba describe el régimen al que se vuelve en cuanto exista la
> primera versión etiquetada; mientras tanto, un cambio incompatible en la herramienta **sí** puede poner en
> rojo un PR aquí, y `compat.yml` avisa de lo mismo que ya bloquea. La primera tarea de mantenimiento
> pendiente es, precisamente, pinear: etiquetar la herramienta y poner ese tag como default del workflow.

## Subir el pin de la herramienta

Es la tarea principal, y es **un solo PR**: el pin y el payload se mueven juntos, porque el payload tiene que
corresponder a la versión que valida.

**1. Comprueba que el canario está verde.** Lánzalo a mano contra el ref que quieras adoptar:
`workflow_dispatch` con `ref: <tag o sha>`. Si sale rojo, resuelve primero lo que reporte — el resto de los
pasos no arregla una incompatibilidad real.

**2. El ref tiene que existir y ser inmutable.** Un tag (`v0.3.1`) o un sha, nunca una rama: el sentido del
pin es que dos ejecuciones del CI a un mes de distancia validen igual.

**3. Instala esa versión exacta en local** y trabaja con ella, no con la que tengas puesta:

```bash
git clone https://github.com/keel-system/keel /tmp/keel-tool && cd /tmp/keel-tool
git checkout <ref>
npm install && npm link --workspace packages/keel-core
keel --version
```

**4. Refresca el payload:**

```bash
cd <registry>
keel init --force
```

`--force` **conserva** `README.md`, `AGENTS.md`, `CLAUDE.md`, `.gitignore`, `.gitattributes` y
`contracts/README.md` (los lista como *«es tuyo, no se sobrescribe»*), y no toca `specs/`, `docs/<slug>/` ni
`publish.yaml`, que no son payload — `publish.yaml` ni siquiera aparece en esa lista de conservados porque la
CLI no lo siembra nunca: es config de este repo. Revisa el diff igualmente: debe limitarse a `schema/`, `templates/`, `docs/dsl/`, los
`docs/*.md` del método y las skills de `.claude/` y `.opencode/`.

Dos cosas que `--force` **no** hace y hay que mirar a mano: no borra huérfanos (una skill retirada en la
herramienta se queda aquí, y `keel init --check` tampoco la reporta — se detecta leyendo el `git status` y
comparando con las skills del payload), y no actualiza `AGENTS.md`, que es customizable: si la plantilla de
la herramienta cambió, el texto propio de este repo (la sección *«Este workspace es el registry»*) se
conserva y el resto se reconcilia a mano contra `assets/core/AGENTS.md`.

**5. Actualiza el pin** en `.github/workflows/ci.yml` (o la variable de repo `KEEL_TOOL_REF` si es una prueba
que no quieres commitear).

**6. Reindexa y verifica las tres puertas**, las mismas que corre el CI:

```bash
keel index
keel validate specs/<slug>    # por cada diseño
keel index --check
keel init --check
```

**7. Un PR con todo junto**, con el resumen del canario enlazado y una línea por cada cosa que el salto de
versión trae (una versión nueva del DSL, una capa nueva, una regla de validación nueva).

### Si al subir el pin un diseño publicado deja de validar

Es el caso interesante, y **la respuesta depende de quién tiene la culpa**:

| Causa | Qué hacer |
|---|---|
| La herramienta ganó una **regla de validación** que el diseño incumple de verdad | El diseño se arregla, no la regla. Avisa a su owner de [CODEOWNERS](CODEOWNERS) y abre un issue; si el arreglo es del diseño en sí, se hace con `/keel-evolve specs/<slug>`, que versiona el contrato y propaga a los derivados. **El pin no sube hasta que el diseño esté al día.** |
| La regla nueva es **ruido o está mal planteada** (avisa de algo legítimo) | El pin no sube. Se corrige en la herramienta, no aquí, y no se silencia con un cambio cosmético del diseño. |
| El diseño usa una construcción que la versión pineada **entiende** y la nueva **rompe** | Es una regresión de la herramienta. Se reporta allí; el pin se queda donde está. El canario seguirá rojo, y eso es información correcta. |

La regla de fondo: **el pin nunca sube dejando un diseño roto detrás**, ni siquiera con un diseño en
`draft`. Un diseño que no valida es un diseño que alguien descargará y no podrá generar.

Y no se congela un diseño excluyéndolo del bucle de `ci.yml`: si un diseño no puede validar, se arregla o se
retira, porque el bucle recorre `specs/*/` y esa cobertura completa es la garantía del repo.

## El formato del índice

La única dependencia que va en la otra dirección: `keel-core` conoce este registry
(`DEFAULT_REGISTRY_URL` en `src/lib/registry-source.js` apunta a `index.json` de `main`).

Por eso, cuando la herramienta sube `INDEX_SCHEMA_VERSION`, el orden se **invierte**: primero se publica una
CLI que lea el formato nuevo y **solo después** el registry regenera su `index.json`. Al contrario, un
`index.json` con un `schemaVersion` mayor del que entiende la CLI de un usuario hace que `keel registry` lo
rechace pidiéndole que actualice — y quien queda roto es quien solo quería hojear diseños.

Consecuencia práctica: al subir el pin a una versión que cambió el formato del índice, el `keel index` del
paso 6 reescribe `index.json` en el formato nuevo, y eso **rompe a los usuarios con la CLI anterior**. Ese
salto se anuncia en el `README.md` antes de mergearlo.

## Aceptar un diseño de la comunidad

Además del listón de [CONTRIBUTING.md](CONTRIBUTING.md), que el CI ya comprueba, lo que el CI **no** puede
juzgar y sí tiene que revisar una persona:

1. **Que sea un diseño y no una implementación disfrazada.** Cero tecnología: si solo tiene sentido con
   Postgres o con Kafka, todavía no es un diseño.
2. **Que los tags digan la verdad.** `outbox` exige `reliability: outbox` en `messaging`; `cache`, alguna
   operación con `cache`. Es lo único del sidecar que nada valida mecánicamente.
3. **Que los escenarios de `validation-scenarios.md` ejerciten el diseño**, no que existan. Son el contrato
   de equivalencia: con ellos se declara correcto un servidor generado en cualquier stack.
4. **Si es una variante, que su `differsIn` justifique existir.** Es la columna con la que alguien elegirá
   entre hermanas; «añade más cosas» no es una diferencia.
5. **Añade sus dos rutas a `CODEOWNERS`** (`/specs/<slug>/` y `/docs/<slug>/`) con el usuario que lo
   mantiene. Sin eso, los PR futuros sobre ese diseño no le llegan a nadie que lo conozca.

## Promover la madurez

`maturity` en `specs/<slug>/design.yaml` la mueven los mantenedores, no el autor:

- **`draft`** — donde entra todo diseño nuevo.
- **`stable`** — cuando ya sostuvo **una implementación real**. No es «lleva tiempo sin cambiar».
- **`reference`** — muestra de la metodología, elegida deliberadamente. Es lo que se enseña primero, así que
  se reserva a diseños que se pueden leer como ejemplo de cómo se diseña con Keel.

Un cambio de madurez es un cambio del sidecar, así que hay que reindexar (`keel index`): la tabla del
`README.md` ordena las variantes de una familia de más madura a menos.

## Lo que nunca se edita a mano

- **La región del `README.md` entre `<!-- keel:servicios:start -->` y `<!-- keel:servicios:end -->`** e
  `index.json`: los escribe `keel index`, que es su único escritor. El resto del `README.md` sí es prosa
  propia del repo.
- **`schema/`, `templates/`, `docs/dsl/`, `.claude/` y `.opencode/`**: son copias del payload de la herramienta. Se
  refrescan con `keel init --force` y nada más. Un arreglo que haga falta ahí se hace en la herramienta.
- **Un spec publicado**: se cambia con `/keel-evolve specs/<slug>`, que versiona el contrato y regenera los
  derivados en cascada. Editarlo a mano deja derivados que mienten, y `keel describe <slug>` lo delata.

La excepción, por contraste: **`publish.yaml` sí se edita a mano**. No es payload (la CLI no lo siembra) ni
contenido de ningún diseño: declara dónde está publicado este repo (`repo`, `branch`) y es lo que hace que
`keel index` enlace el panel y los visores de cada diseño por `htmlpreview.github.io` en vez de en relativo,
que GitHub muestra como código fuente. Solo se toca si el repo se mueve de sitio o cambia de rama por
defecto; nada lo valida más allá de su schema, así que un `repo` mal escrito no rompe ninguna puerta del CI
— deja enlaces rotos en la portada, que es peor. Al cambiarlo, reindexar (`keel index`) y abrir un enlace.
