# Referencia: cómo el runner resuelve y ejecuta una action

Documento de estudio. No es necesario para usar ni mantener `ci-preflight`,
pero explica los mecanismos que están detrás de las decisiones de diseño.

---

## 1. El ciclo completo

```
   uses: owner/repo@ref
            │
            ▼
   ┌────────────────────────────────────────────────┐
   │ FASE 1 · RESOLUCIÓN                            │
   │  a) Traduce la referencia a un ref de git      │
   │  b) Descarga el repo TAL COMO ESTÁ en ese ref  │
   │     → /home/runner/work/_actions/owner/repo/…  │
   │  c) Lee y valida action.yml                    │
   │                                                │
   │  NO compila. NO instala. NO buildea nada.      │
   └────────────────────────────────────────────────┘
            │
            ▼
   ┌────────────────────────────────────────────────┐
   │ FASE 2 · PREPARACIÓN DE ENTRADAS               │
   │  Mezcla los `with:` con los `default:`         │
   └────────────────────────────────────────────────┘
            │
            ▼
   ┌────────────────────────────────────────────────┐
   │ FASE 3 · EJECUCIÓN (según runs.using)          │
   └────────────────────────────────────────────────┘
        │              │                  │
   node24         docker            composite
        │              │                  │
   node dist/     build/pull +      expande los steps
   index.js       docker run        dentro del job
            │
            ▼
   ┌────────────────────────────────────────────────┐
   │ FASE 4 · SALIDAS                               │
   │  Lee $GITHUB_OUTPUT, $GITHUB_ENV, $GITHUB_PATH │
   │  Ejecuta el `post:` si existe                  │
   └────────────────────────────────────────────────┘
```

### "No buildea nada" es literal

El runner ejecuta los archivos commiteados en ese ref. Es la razón por la que
las JavaScript actions deben versionar su `dist/`: si el bundle no está en el
tag, el consumidor recibe un repositorio sin código ejecutable.

### La validación no rechaza runtimes deprecados

Un `using` que nunca existió (por ejemplo `python`) falla en la Fase 1. Un
runtime válido pero deprecado (`node16`, `node20`) **no falla**: el runner lo
coerciona a `node24` y emite un warning.

El warning puede además nombrar una versión distinta de la declarada
(`actions/runner#4295`): un `action.yml` con `node16` produce un aviso sobre
Node 20. El mensaje es un texto genérico del mecanismo de coerción, calculado
antes de la sustitución.

Consecuencia práctica: al auditar una action de terceros, no confíes en que un
runtime obsoleto vaya a fallar visiblemente. Léelo del `action.yml`.

---

## 2. Cómo viajan los inputs, según el tipo

| Tipo | Cómo llegan al código | Cómo se devuelven outputs |
|---|---|---|
| `node24` | Variables de entorno `INPUT_<NOMBRE>`, leídas por `core.getInput()` | `core.setOutput()` |
| `docker` | Variables `INPUT_<NOMBRE>`, leídas a mano en el entrypoint | `echo "k=v" >> $GITHUB_OUTPUT` |
| `composite` | Expresión `${{ inputs.x }}`, expandida como **texto** antes de ejecutar | `outputs.x.value` apuntando al output de un step |

La fila del composite es la excepción y tiene dos consecuencias:

**No existen variables `INPUT_*`.** Verificado empíricamente: un `env | grep
INPUT_` dentro de un composite no devuelve nada.

**La expansión textual es un vector de inyección.** El motor sustituye el valor
en el script antes de que bash lo vea. Si el valor viene de una fuente que
controla un tercero, ese tercero está escribiendo tu script:

```yaml
# VULNERABLE: el título del PR se convierte en código
run: echo "${{ github.event.pull_request.title }}"

# SEGURO: bash recibe el valor como dato
env:
  TITLE: ${{ github.event.pull_request.title }}
run: echo "$TITLE"
```

---

## 3. Formas de referenciar una action

| Forma | Ejemplo | Requiere checkout | Resuelve contra |
|---|---|---|---|
| Remota | `owner/repo@v1` | No | El ref indicado |
| Remota en subcarpeta | `owner/repo/path@v1` | No | El ref indicado |
| Workspace-relativa | `./.github/actions/x` | **Sí** | Lo que haya en el disco |
| Self-repository | `$/.github/actions/x` | No | El commit que ejecuta el workflow |
| Imagen directa | `docker://ghcr.io/org/img@sha256:…` | No | El digest |

### `./` versus `$/`

`./` apunta al **directorio de trabajo del job**, que empieza vacío y se llena
con `actions/checkout`. Resuelve contra lo que ese checkout haya traído, que no
es necesariamente el commit que disparó el workflow:

```yaml
- uses: actions/checkout@v7
  with:
    ref: main          # ← puede ser otro commit
- uses: ./.github/actions/deploy    # ← ejecuta la versión de main
```

Escenario de fallo: abres un PR desde una rama basada en el commit A; alguien
mergeó el commit B a `main` rompiendo la action; tu PR falla por un cambio que
no está en tu rama.

`$/` resuelve contra el commit que ejecuta el workflow, sin mirar el disco.
Verificado: el runner lo expande a `owner/repo@<SHA completo>` y descarga los
archivos a `_actions/`, dejando el workspace intacto.

El caso serio es con reusable workflows. Si un consumidor pinea
`owner/toolkit/.github/workflows/build.yml@a1b2c3d` y ese workflow usa `./` por
dentro, la action interna viene del checkout, no del pin. **El pinning cubre el
workflow pero no lo que el workflow ejecuta.** Con `$/`, el pin se propaga.

### Archivos de la action vs archivos del consumidor

Distinción que causa confusión:

| Qué | De dónde viene | ¿Necesita checkout? |
|---|---|---|
| `action.yml` y scripts propios de la action | `_actions/`, descargado por el runner | No |
| Archivos del repositorio sobre los que la action opera | El workspace | **Sí** |

`ci-preflight` lee el `.nvmrc` del consumidor, así que el checkout es
prerrequisito aunque la action misma se resuelva sin él.

---

## 4. Comportamiento del shell

`shell: bash` se expande a:

```
/bin/bash --noprofile --norc -eo pipefail {0}
```

| Flag | ¿Activo con `shell: bash`? | ¿Lo agrega `set -euo pipefail`? |
|---|---|---|
| `-e` | Sí | Nada nuevo |
| `-o pipefail` | Sí | Nada nuevo |
| `-u` | **No** | **Sí** |

Para desactivar el fail-fast hay que dar una plantilla explícita: `shell: bash {0}`.

Sin fail-fast, un comando que falla no detiene el script. En `ci-preflight`
eso produjo un run **verde** con Node 22 instalado cuando el `.nvmrc` decía 24:
`cat` falló, la variable quedó vacía, y `setup-node` con versión vacía no
instala nada — usa el Node preinstalado del runner.

Es el modo de fallo más caro que existe en CI: un pipeline que reporta éxito sin
haber verificado lo que dice verificar.

---

## 5. Archivos especiales del runner

| Variable | Es | Alcance | Persiste |
|---|---|---|---|
| `$GITHUB_OUTPUT` | Archivo | El step que lo escribe | Como output del step |
| `$GITHUB_ENV` | Archivo | **Todo el job** | Como variable de entorno |
| `$GITHUB_PATH` | Archivo | **Todo el job** | Se antepone al `PATH` |
| `$GITHUB_STEP_SUMMARY` | Archivo | El run | Markdown en la UI |

Las dos del medio son las que producen efectos sobre el job del consumidor, y
por eso su modificación es un cambio de contrato.

### Valores multilínea

`echo "k=v" >> $GITHUB_OUTPUT` solo sirve para una línea. Para varias se usa un
delimitador, que debe ser aleatorio si el contenido no es de confianza: un
contenido que incluya la línea del delimitador podría cerrar el bloque e
inyectar outputs falsos.

```bash
DELIM="$(openssl rand -hex 16)"
{
  echo "reporte<<$DELIM"
  cat archivo.txt
  echo "$DELIM"
} >> "$GITHUB_OUTPUT"
```

---

## 6. Encapsulación de outputs

Los outputs de los steps internos **no** se exponen automáticamente. Hay que
declararlos:

```yaml
outputs:
  node-version:
    value: ${{ steps.resolve.outputs.version }}
```

La indirección permite renombrar o dividir el step `resolve` sin romper
consumidores. Si GitHub expusiera los steps internos, un refactor puramente
interno sería un breaking change.

Es el mismo principio que separar interfaz pública de implementación en
cualquier lenguaje.

---

## 7. Validación que NO ocurre

| Mecanismo | ¿Valida? |
|---|---|
| `required: true` en un input de `action.yml` | **No.** Es documentación. La action corre con el input vacío, sin warning |
| `required: true` en `workflow_dispatch` | Sí, la UI de GitHub lo bloquea |
| `core.getInput('x', { required: true })` en una JS action | Sí, pero lo valida el **toolkit**, no la plataforma |

Un composite no tiene toolkit, así que todo guard se escribe a mano. Verificado
empíricamente: un input `required: true` no pasado produce cadena vacía y
ningún aviso.

---

## 8. Estrategias de cache

| Estrategia | Qué guarda | Riesgo |
|---|---|---|
| `node_modules` | Árbol instalado con binarios nativos | **Alto**: compilados para un SO/arquitectura/versión de Node específicos |
| `~/.npm` | Tarballs descargados | Bajo: artefactos neutros, `npm ci` reinstala correctamente |

La clave debe segmentar por todo lo que afecte la validez del contenido:

```
<prefijo>-<os>-<arch>-node<version>-<hash del lockfile>
```

Omitir `runner.arch` es un error sutil: `runner.os` devuelve `Linux` tanto en
`ubuntu-24.04` como en `ubuntu-24.04-arm`. Con `~/.npm` la consecuencia es
cosmética (se descarga lo que falta); con `node_modules` sería un binario para
la arquitectura equivocada, fallando en runtime con `invalid ELF header`.

### `cache-hit` y `restore-keys`

`restore-keys` permite un hit **parcial** por prefijo: si el lockfile cambió, se
restaura el cache anterior y npm descarga solo el delta.

Verificado: ante un hit parcial, `cache-hit` reporta `'false'`. El output
significa "coincidencia exacta", no "hubo restauración". Un consumidor que use
`if: cache-hit != 'true'` para saltarse trabajo caro lo ejecutará
innecesariamente en cada cambio de dependencia.

---

## 9. Versionado

```
   Commits en main
   ─────────────────────────────────────────►
     A         B         C         D
     │         │         │         │
   v1.0.0    v1.0.1    v1.1.0    v1.2.0    ← inmutables por convención
     │                             │
     └──── v1 ────────────────────►┘        ← móvil, se reescribe
```

| Referencia | Garantía | Quién controla el cambio |
|---|---|---|
| `@v1` | Ninguna, se mueve | El mantenedor |
| `@v1.0.0` | Convención social. Un tag es reescribible | El mantenedor, si actúa de mala fe |
| `@<sha>` | Criptográfica | El consumidor |
| `@main` | Ninguna | Nadie |

La diferencia entre `@v1.0.0` y `@<sha>` solo importa cuando la confianza falla
—cuenta comprometida, mantenedor malicioso, error humano— que es exactamente
cuando importa. Un tag es una promesa; un SHA es una garantía.

### Elegir estrategia

| | `@vX` | `@<sha>` + Dependabot |
|---|---|---|
| Parche urgente en N repos | Automático | N PRs a revisar |
| Un bug del mantenedor llega a producción | En N repos, sin aviso | Solo donde ya se mergeó |
| Costo humano recurrente | Cero | N PRs por release |

Para actions de terceros, SHA: el vector "mantenedor comprometido" es real y ha
ocurrido con actions populares.

Para actions internas donde el mismo equipo controla ambos lados, `@vX` es
defendible **si el movimiento del tag está automatizado y auditado**. El riesgo
de exigir SHA en muchos repos internos es que los PRs se mergeen sin leer,
convirtiendo el pinning en teatro.

---

## 10. Para seguir estudiando

- **Immutable actions**: publicación como artefacto OCI en un registry, donde
  la versión no se puede sobrescribir. Resuelve el problema del tag reescribible.
  Aún no disponible públicamente.
- **`pull_request_target` y el patrón "pwn request"**: `actions/checkout` v7
  rechaza por defecto el checkout de código de forks bajo ese evento.
- **Artifact attestations** (`actions/attest-build-provenance`): procedencia
  SLSA para artefactos publicados.
- **`@github/local-action`**: ejecutar una JS action localmente con el toolkit
  stubeado, sin commitear ni pushear.