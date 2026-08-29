# CI Preflight

Composite action que prepara el entorno de CI para proyectos Node: resuelve la
versión desde un archivo del repositorio, valida los manifiestos npm, restaura
el cache de dependencias e instala.

Reemplaza los cinco o seis steps de arranque que suelen repetirse en cada
repositorio de una organización, y centraliza esa política en un solo lugar
versionado.

---

## Uso

```yaml
jobs:
  ci:
    runs-on: ubuntu-24.04
    steps:
      # REQUERIDO: esta action lee archivos de TU repositorio.
      # Sin checkout, el workspace está vacío y la action falla.
      - uses: actions/checkout@v7

      - uses: mantoniocc/ci-preflight@v1

      - run: npm test
```

Eso es todo. La action lee `.nvmrc`, instala esa versión de Node, restaura el
cache y ejecuta `npm ci`.

### Monorepo

```yaml
      - uses: mantoniocc/ci-preflight@v1
        with:
          working-directory: services/api
          cache-key-prefix: api
```

El `cache-key-prefix` distinto evita que dos servicios del mismo repositorio
compartan entrada de cache.

### Solo preparar el entorno, sin instalar

```yaml
      - uses: mantoniocc/ci-preflight@v1
        with:
          install: 'false'
```

---

## Qué garantiza

- **La versión de Node que corre es la que declara tu repositorio.** Si el
  archivo de versión falta, está vacío, o la versión instalada no coincide con
  la resuelta, la action falla. No hay degradación silenciosa a la versión que
  venga preinstalada en el runner.
- **Instalación reproducible.** Usa `npm ci`, que nunca modifica el lockfile y
  falla si el lockfile no coincide con `package.json`.
- **Rechaza lockfiles de npm 6** (`lockfileVersion: 1`), que no contienen datos
  de integridad del árbol completo. Esto no lo detecta `npm ci`.

## Qué NO garantiza

- No verifica el campo `engines` de tu `package.json`.
- No audita vulnerabilidades. Para eso, `npm audit` o Dependabot.
- No cachea `node_modules`, solo `~/.npm`. La instalación siempre se ejecuta.

---

## Inputs

| Input | Default | Descripción |
|---|---|---|
| `node-version-file` | `.nvmrc` | Ruta relativa al archivo que declara la versión de Node |
| `working-directory` | `.` | Directorio base desde el cual resolver rutas relativas |
| `install` | `'true'` | Si ejecutar `npm ci` |
| `cache` | `'true'` | Si restaurar y guardar el cache de dependencias |
| `cache-key-prefix` | `npm` | Prefijo de la clave de cache |

Todos los inputs son strings. `install: false` (booleano YAML) y
`install: 'false'` (string) se comportan igual aquí, pero la forma con comillas
es la explícita.

## Outputs

| Output | Valores | Descripción |
|---|---|---|
| `node-version` | ej. `24` | La versión resuelta desde el archivo |
| `cache-hit` | `'true'`, `'false'`, `''` | **Ver el gotcha más abajo** |
| `dependencies-installed` | `'true'`, `'false'` | Si esta action ejecutó la instalación |

---

## Efectos sobre el job

Esta action **modifica el entorno del job que la invoca**. No es un efecto
secundario accidental: es cómo funcionan las composite actions, cuyos steps se
ejecutan dentro del job del consumidor.

| Qué modifica | Consecuencia |
|---|---|
| `PATH` | Todos los steps posteriores usan la versión de Node que esta action instaló |
| `node_modules` del directorio de trabajo | Se borra y reinstala si `install: 'true'` |

### Interacción con un `setup-node` previo

```yaml
      - uses: actions/setup-node@v7
        with:
          node-version: 20        # tu decisión

      - uses: mantoniocc/ci-preflight@v1   # tu .nvmrc dice 24

      - run: npm test             # corre en Node 24, NO en 20
```

La action **pisa** la versión que hayas configurado antes. Si necesitas una
versión distinta de la que declara tu `.nvmrc`, cambia el `.nvmrc` o usa el
input `node-version-file` apuntando a otro archivo. No pongas un `setup-node`
antes esperando que gane.

---

## Gotchas

### `cache-hit: 'false'` no significa "no se restauró nada"

`cache-hit` indica **coincidencia exacta de clave**. Cuando cambias una
dependencia, la clave cambia, pero el cache anterior se restaura igual mediante
`restore-keys` — y `cache-hit` reporta `'false'`.

```yaml
      # INCORRECTO: se ejecuta en cada cambio de dependencia,
      # aunque el cache haya servido.
      - if: steps.pre.outputs.cache-hit != 'true'
        run: ./warm-up-caro.sh
```

Si necesitas saber si hubo restauración parcial, revisa el log del step de
cache. El output no lo distingue.

### El error no dice que el problema está en la action

Si esta action empieza a fallar sin que hayas cambiado nada en tu repositorio,
lo más probable es que el tag `v1` se haya movido a una versión con un bug.
El mensaje de error hablará de tu código, no de la action.

Para confirmarlo:

```bash
# Qué commit resuelve tu referencia ahora
gh api repos/mantoniocc/ci-preflight/commits/v1 --jq .sha

# Compara con el commit del último release conocido bueno
gh release list --repo mantoniocc/ci-preflight
```

Si difieren, pinea temporalmente a la versión anterior y abre un issue.

### El output `dependencies-installed` no distingue "falló" de "omitido"

Si la action falla antes de llegar al step de instalación, el job aborta y tus
steps posteriores no se ejecutan, así que la ambigüedad no se materializa. Pero
si usas `continue-on-error: true` sobre esta action, necesitas
`steps.<id>.outcome` para distinguir los casos:

```yaml
      - uses: mantoniocc/ci-preflight@v1
        id: pre
        continue-on-error: true

      - if: steps.pre.outcome == 'failure'
        run: echo "la action falló"

      - if: steps.pre.outputs.dependencies-installed == 'false'
        run: npm ci   # se omitió deliberadamente, instalo yo
```

### `working-directory` no afecta a los steps posteriores del consumidor

El input solo aplica dentro de la action. Tus steps siguientes arrancan en la
raíz del workspace.

---

## Errores comunes

| Mensaje | Causa | Solución |
|---|---|---|
| `No existe '.nvmrc' en /home/runner/work/...` | Falta el archivo, o falta `actions/checkout` | Crear el archivo, o agregar el checkout antes |
| `'.nvmrc' existe pero no contiene una version` | Archivo vacío | `echo 24 > .nvmrc` |
| `npm ci requiere un lockfile` | No hay `package-lock.json` | `npm install --package-lock-only` |
| `lockfileVersion 1` | Lockfile generado con npm 6 | `rm package-lock.json && npm install` con npm 7+ |
| `Se resolvio 'X' pero quedo instalada 'Y'` | `setup-node` no pudo satisfacer la versión | Verificar que la versión existe y el formato es válido |

---

## Versiones

| Referencia | Qué recibes |
|---|---|
| `@v1` | Lo último de la línea 1.x. **Este tag se mueve.** |
| `@v1.0.0` | Ese release. No se mueve por convención, pero técnicamente un tag es reescribible |
| `@<sha>` | Ese commit exacto. Inmutable por criptografía |

El tag `v1` se actualiza con cada release de la línea 1.x. Los cambios que
rompen compatibilidad se publican bajo `v2` y `v1` deja de recibirlos.

Qué estrategia de referencia usar es una decisión de la política de supply
chain de tu organización, no una recomendación de esta action.

---

## Requisitos

- Runner Linux, macOS o Windows (probado en `ubuntu-24.04`)
- Runner versión 2.336.0 o superior si la consumes con la sintaxis `$/`
- `actions/checkout` ejecutado antes

## Limitaciones conocidas

- El step de validación de manifiestos usa `node -p` antes de que `setup-node`
  se haya ejecutado, dependiendo del Node preinstalado en la imagen del runner.
  Funciona en todos los runners GitHub-hosted actuales, pero es una dependencia
  implícita del entorno.