# CI Preflight

Composite action que prepara el entorno de CI para proyectos Node: resuelve la
versión desde un archivo del repositorio, valida los manifiestos npm, restaura
el cache de dependencias e instala.

Reemplaza los steps de arranque que suelen repetirse en cada repositorio de una
organización y centraliza esa política en un solo lugar versionado.

---

## Uso

```yaml
jobs:
  ci:
    runs-on: ubuntu-24.04
    steps:
      # Requerido: esta action lee archivos de tu repositorio.
      - uses: actions/checkout@v7

      - uses: mantoniocc/ci-preflight@v1

      - run: npm test
```

Lee `.nvmrc`, instala esa versión de Node, restaura el cache y ejecuta `npm ci`.

### Monorepo

```yaml
      - uses: mantoniocc/ci-preflight@v1
        with:
          working-directory: services/api
          # Un prefijo distinto por servicio evita que compartan
          # la misma entrada de cache.
          cache-key-prefix: api
```

### Preparar el entorno sin instalar

```yaml
      - uses: mantoniocc/ci-preflight@v1
        with:
          install: 'false'
```

### Usar los outputs

```yaml
      - uses: mantoniocc/ci-preflight@v1
        id: pre

      - run: echo "Corriendo en Node ${{ steps.pre.outputs.node-version }}"
```

---

## Contrato

### Inputs

| Input | Default | Descripción |
|---|---|---|
| `node-version-file` | `.nvmrc` | Ruta relativa al archivo que declara la versión de Node |
| `working-directory` | `.` | Directorio base desde el cual resolver rutas relativas |
| `install` | `'true'` | Si ejecutar `npm ci` |
| `cache` | `'true'` | Si restaurar y guardar el cache de dependencias |
| `cache-key-prefix` | `npm` | Prefijo de la clave de cache |

Todos los inputs son strings.

### Outputs

| Output | Valores | Descripción |
|---|---|---|
| `node-version` | ej. `24` | Versión resuelta desde el archivo |
| `cache-hit` | `'true'` \| `'false'` \| `''` | `'true'` solo ante coincidencia **exacta** de clave. Un `'false'` no significa que no se haya restaurado nada: ver Troubleshooting |
| `dependencies-installed` | `'true'` \| `'false'` | Si esta action ejecutó la instalación |

### Garantías

- **La versión de Node que corre es la que declara tu repositorio.** Si el
  archivo falta, está vacío, o la versión instalada no coincide con la
  resuelta, la action falla en vez de continuar con la versión preinstalada en
  el runner.
- **Instalación reproducible.** Usa `npm ci`, que nunca modifica el lockfile y
  falla si el lockfile no coincide con `package.json`.
- **Rechaza lockfiles de npm 6** (`lockfileVersion: 1`), que no contienen datos
  de integridad del árbol completo.

### Fuera de alcance

No verifica el campo `engines`, no audita vulnerabilidades y no cachea
`node_modules` — la instalación siempre se ejecuta.

---

## Efectos sobre el job

Esta action **modifica el entorno del job que la invoca**. Es cómo funcionan las
composite actions: sus steps se ejecutan dentro del job del consumidor.

| Qué modifica | Consecuencia |
|---|---|
| `PATH` | Los steps posteriores usan la versión de Node que esta action instaló |
| `node_modules` del directorio de trabajo | Se borra y reinstala si `install: 'true'` |

El input `working-directory` solo aplica dentro de la action; tus steps
posteriores arrancan en la raíz del workspace.

### Precedencia sobre un `setup-node` previo

```yaml
      - uses: actions/setup-node@v7
        with:
          node-version: 20

      - uses: mantoniocc/ci-preflight@v1   # .nvmrc dice 24

      - run: npm test                      # corre en Node 24
```

La versión del archivo gana. Para usar una versión distinta, cambia el archivo
o apunta `node-version-file` a otro.

---

## Requisitos

- `actions/checkout` ejecutado antes
- Un archivo de versión (`.nvmrc` por defecto) y un lockfile en el repositorio
- Runner Linux, macOS o Windows. Probado en `ubuntu-24.04`
- Runner 2.336.0 o superior si la consumes con la sintaxis `$/`

---

## Versiones

| Referencia | Qué recibes |
|---|---|
| `@v1` | Lo último de la línea 1.x. **Este tag se mueve con cada release** |
| `@v1.0.0` | Ese release |
| `@<sha>` | Ese commit exacto |

Los cambios que rompen compatibilidad se publican bajo `v2`; `v1` deja de
recibirlos.

Qué estrategia de referencia usar es una decisión de la política de supply chain
de tu organización.

---

## Troubleshooting

### `No existe '.nvmrc' en /home/runner/work/...`

Falta el archivo, o falta `actions/checkout` antes de la action. El mensaje
incluye el directorio donde buscó: si el directorio está vacío, es lo segundo.

```bash
echo 24 > .nvmrc
```

### `'.nvmrc' existe pero no contiene una version`

Archivo vacío o solo con espacios. Debe contener la versión sin la `v` inicial:
`24`, `24.5.0`, `lts/*`.

### `npm ci requiere un lockfile`

```bash
npm install --package-lock-only
```

### `<lockfile> usa lockfileVersion 1`

El lockfile se generó con npm 6. Regenéralo con npm 7 o superior:

```bash
rm package-lock.json && npm install
```

### `Se resolvio 'X' pero quedo instalada 'Y'`

`setup-node` no pudo satisfacer la versión pedida. Verifica que la versión
existe y que el formato del archivo es válido.

### La action falla y no cambié nada en mi repositorio

Si consumes por `@v1`, el tag pudo haberse movido a una versión nueva. El error
hablará de tus archivos, no de la action.

```bash
# Qué commit resuelve tu referencia ahora
gh api repos/mantoniocc/ci-preflight/commits/v1 --jq .sha

# Releases publicadas
gh release list --repo mantoniocc/ci-preflight
```

Si el commit es más nuevo que el último release conocido bueno, pinea
temporalmente a la versión anterior y abre un issue.

### `cache-hit: 'false'` pero el cache sí sirvió

`cache-hit` indica coincidencia **exacta** de clave. Cuando cambias una
dependencia la clave cambia, pero el cache anterior se restaura igual mediante
`restore-keys`, y el output reporta `'false'`.

```yaml
      # Se ejecuta en cada cambio de dependencia, aunque el cache haya servido.
      - if: steps.pre.outputs.cache-hit != 'true'
        run: ./warm-up-caro.sh
```

Si necesitas distinguir la restauración parcial, revisa el log del step de
cache. El output no lo expone.

### Distinguir "la action falló" de "omití la instalación"

Con `install: 'false'`, `dependencies-installed` devuelve `'false'`. Si la
action falla, el job aborta y tus steps posteriores no corren, así que el caso
no se presenta — salvo que uses `continue-on-error`:

```yaml
      - uses: mantoniocc/ci-preflight@v1
        id: pre
        continue-on-error: true

      - if: steps.pre.outcome == 'failure'
        run: echo "la action falló"

      - if: steps.pre.outputs.dependencies-installed == 'false'
        run: npm ci
```

---

## Documentación adicional

- [`RUNBOOK.md`](RUNBOOK.md) — publicar, revertir y mantener la action
- [`docs/design-decisions.md`](docs/design-decisions.md) — por qué la action
  está construida así
- [`docs/action-references.md`](docs/action-references.md) — cómo el runner
  resuelve y ejecuta actions