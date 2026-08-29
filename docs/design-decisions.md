# Decisiones de diseño

Por qué `ci-preflight` está construida así. Cada decisión incluye la evidencia
que la respalda: en su mayoría, experimentos ejecutados en un runner real
durante la construcción de la action.

Para los mecanismos generales del runner, ver
[`action-references.md`](action-references.md).

---

## 1. Fallar es mejor que degradar

**Decisión:** si la action no puede cumplir su promesa, aborta. No usa valores
por defecto ni continúa con lo que encuentre.

**Evidencia.** Durante la construcción se ejecutó la resolución de versión sin
fail-fast (`shell: bash {0}`) y con el `.nvmrc` ausente. Resultado:

```
cat: .nvmrc: No such file or directory     ← el error ocurrió
Version resuelta: []                        ← y se ignoró
setup-node → node: v22.23.2                 ← usó el Node del runner
✅ Workflow verde
```

`setup-node` con `node-version` vacío no falla: no instala nada y deja el Node
preinstalado en la imagen. El `.nvmrc` decía 24, el CI corrió en 22, y todo
pasó en verde.

**Por qué importa:** el modo de fallo más caro en CI no es el rojo, es el verde
injustificado. Un pipeline que reporta éxito sin verificar lo que dice
verificar es peor que no tener pipeline, porque genera confianza que nadie va a
cuestionar. La información estaba en el log —el step final imprime la versión—
y nadie mira el log de un check verde.

**Regla derivada:** una action que promete "instala la versión declarada en el
repositorio" debe tratar "no encontré la declaración" como error, no como caso
a resolver con un default.

---

## 2. Cuándo agregar un guard

**Decisión:** un guard se agrega cuando su ausencia produce un **falso verde**.
Cuando su ausencia produce un rojo que ya era claro, es ruido.

| Situación | ¿Lo detecta npm? | ¿Guard propio? | Criterio |
|---|---|---|---|
| Lockfile desincronizado | Sí, con mensaje claro | No | Duplicar no aporta |
| Falta `package.json` | Sí, pero tras restaurar cache | Sí | Fallar antes ahorra tiempo y evita cache basura |
| Falta lockfile | Sí | Sí | Además la clave de cache quedaría inservible |
| `lockfileVersion: 1` | **No** | Sí | Nadie más lo detecta |
| Versión instalada ≠ resuelta | **No** | Sí | Nadie más lo detecta (ver §1) |

**Contraste deliberado:** no validamos la sincronización del lockfile aunque
sería fácil, porque `npm ci` ya falla con un buen mensaje. Sí verificamos la
versión instalada aunque `setup-node` "debería" garantizarla, porque es el
único mecanismo que detecta ese fallo.

No es una contradicción: el criterio no es "cuánto podemos validar", es "qué
pasa si no validamos".

---

## 3. Verificar el resultado, no solo la entrada

**Decisión:** además del guard de entrada (el archivo existe y tiene contenido),
hay un guard de salida que compara la versión instalada con la resuelta.

**Por qué:** el guard de entrada cubre "el archivo falta". No cubre que
`setup-node` falle parcialmente, que el formato del archivo sea válido pero no
resoluble, o que un cambio futuro en `setup-node` altere su comportamiento.

Es defensa en profundidad: dos guards independientes sobre el mismo riesgo, en
extremos opuestos del flujo.

---

## 4. Cachear `~/.npm`, no `node_modules`

| | `node_modules` | `~/.npm` |
|---|---|---|
| Qué guarda | Árbol instalado con binarios nativos compilados | Tarballs descargados |
| Velocidad | Mayor: salta `npm ci` | Menor: `npm ci` siempre corre |
| Riesgo al restaurar en otro contexto | Binarios para la arquitectura equivocada | Faltan tarballs; npm los descarga |
| Cuándo se detecta el problema | En runtime, con `invalid ELF header` | Nunca (solo es más lento) |

**Decisión:** `~/.npm`. La velocidad que se gana cacheando `node_modules` se
paga en bugs que solo aparecen en CI y que no mencionan el cache en ningún
momento.

### La clave debe segmentar por todo lo que afecta la validez

```
<prefijo>-<os>-<arch>-node<version>-<hash del lockfile>
```

`runner.arch` no es opcional: `runner.os` devuelve `Linux` tanto en
`ubuntu-24.04` como en `ubuntu-24.04-arm`. Sin la arquitectura, un job ARM
restauraría el cache de un job x64 con la misma clave.

Con `~/.npm` la consecuencia sería cosmética. Con `node_modules` sería un fallo
en runtime. La decisión de qué cachear y la de cómo segmentar la clave están
acopladas.

### `restore-keys` y el guion final

```yaml
restore-keys: |
  npm-Linux-X64-node24-      ← el guion ancla el límite del segmento
  npm-Linux-X64-
```

Sin el guion, un `.nvmrc` con `2` produciría la clave `npm-Linux-X64-node2`,
que hace match por prefijo con los caches de `node20`, `node22` y `node24`.

---

## 5. Inputs a los scripts vía `env`, no interpolados

```yaml
# El motor sustituye TEXTO antes de que bash exista.
# Un valor como  '; rm -rf /; echo '  se convierte en comandos.
run: |
  if [[ "${{ inputs.install }}" != 'true' ]]; then

# Bash recibe el valor como DATO. El mismo string es inofensivo.
env:
  INSTALL: ${{ inputs.install }}
run: |
  if [[ "$INSTALL" != 'true' ]]; then
```

**Decisión:** todos los inputs que llegan a un `run:` pasan por `env`.

En esta action los valores vienen del consumidor, no de un tercero, así que el
riesgo es bajo. Se aplica igual porque es el patrón correcto y porque una
action puede evolucionar hacia consumir datos de eventos, donde sí importa.

---

## 6. `set -euo pipefail` aunque sea casi redundante

`shell: bash` se expande a `bash --noprofile --norc -eo pipefail {0}`, así que
`-e` y `pipefail` ya están activos.

| Flag | ¿Activo con `shell: bash`? | ¿Lo agrega el `set`? |
|---|---|---|
| `-e` | Sí | Nada nuevo |
| `-o pipefail` | Sí | Nada nuevo |
| `-u` | **No** | **Sí** |

**Decisión:** se mantiene el `set` completo. Aporta `-u`, que atrapa variables
no definidas —un typo como `$VERISON` se expandiría a cadena vacía y el script
continuaría—, y deja el script correcto si alguien lo copia a un contexto donde
las banderas por defecto sean otras.

---

## 7. Los steps que emiten outputs no llevan `if:`

**Problema encontrado:** el step de instalación tenía `if: inputs.install ==
'true'`, y el output declarado apuntaba a ese step:

```yaml
outputs:
  dependencies-installed:
    value: ${{ steps.install.outputs.installed }}
```

Con `install: 'false'` el step se salta, no existe, y el consumidor recibe
cadena vacía en vez de `'false'`.

**Intento fallido:** agregar un segundo step `install-skipped` que emitiera
`installed=false`. No funciona: ese valor vive en
`steps.install-skipped.outputs.installed`, y el output declarado sigue
apuntando a `steps.install`.

**Solución:** quitar el `if:` y mover la condición dentro del script.

```yaml
- name: Instalar dependencias
  id: install
  shell: bash
  env:
    INSTALL: ${{ inputs.install }}
  run: |
    if [[ "$INSTALL" != 'true' ]]; then
      echo "installed=false" >> "$GITHUB_OUTPUT"
      exit 0
    fi
    npm ci
    echo "installed=true" >> "$GITHUB_OUTPUT"
```

**Regla derivada:** un step condicional no puede emitir outputs de forma
confiable. Si un output es parte del contrato, el step que lo produce no debe
ser condicional.

---

## 8. Sin `continue-on-error` en steps internos

**Verificado:** `continue-on-error` **sí funciona** en steps de composite
actions. El job continúa y termina en verde.

Pero la anotación de error queda visible en el summary del run. Un consumidor
ve un check verde con una anotación roja que **no puede explicar**, porque
proviene de código que no escribió.

| Estrategia | Anotación visible | Para una action publicada |
|---|---|---|
| `continue-on-error: true` | Sí, roja | No |
| Manejarlo en el script (`\|\| true`, `if`) | No | Sí |

**Decisión:** la tolerancia a fallos se implementa dentro del script, donde se
controla qué se reporta. Es la misma lógica que la §7: lo que es parte del
contrato no se delega al mecanismo del runner.

---

## 9. `required: true` no valida nada

**Verificado:** un input declarado `required: true` sin default, no pasado por
el consumidor, produce **cadena vacía y ningún aviso**. La action se ejecuta
normalmente.

| Mecanismo | ¿Valida? |
|---|---|
| `required: true` en `action.yml` | No. Es documentación |
| `required: true` en `workflow_dispatch` | Sí, la UI lo bloquea |
| `core.getInput('x', { required: true })` en JS | Sí, pero lo valida el toolkit |

Un composite no tiene toolkit. Todo guard se escribe a mano.

**Consecuencia:** si esta action llegara a necesitar un input obligatorio, hay
que verificarlo explícitamente y fallar con un mensaje que diga cómo pasarlo.
Confiar en `required: true` produciría un fallo lejano y confuso: un input vacío
que llega hasta una llamada HTTP y devuelve 401, dirigiendo al consumidor a
revisar credenciales cuando el problema estaba en su YAML.

---

## 10. `$/` en el smoke test, no `./`

El workflow de smoke consume la action con `$/`, que resuelve contra el commit
que ejecuta el workflow en vez de contra el contenido del workspace.

**Verificado:** el runner expande `$/` a `owner/repo@<SHA completo>`, donde el
SHA es el commit del workflow, y descarga los archivos a `_actions/`, dejando el
workspace intacto.

```
action_path = /home/runner/work/_actions/mantoniocc/ci-preflight/<sha>
workspace   = /home/runner/work/ci-preflight/ci-preflight     ← vacío
```

Con `./`, la referencia resolvería contra lo que haya traído el checkout, que no
es necesariamente el commit del workflow.

**Distinción que esto reveló:** los archivos **de la action** no necesitan
checkout; los archivos **del consumidor** sobre los que la action opera, sí. El
smoke test necesita checkout porque la action lee un `.nvmrc`, no porque `$/` lo
requiera.

---

## 11. Mensajes de error accionables

Todos los guards usan `::error title=` con un mensaje que incluye qué se
buscaba, dónde, y cómo arreglarlo.

```
No existe '.nvmrc' en /home/runner/work/... Crealo con: echo 24 > .nvmrc
```

**Por qué importa más de lo que parece.** Cuando el tag `v1` se movió a un
commit roto durante las pruebas, el consumidor vio un error sobre *sus* archivos
en un repositorio que no había cambiado. El mensaje no menciona la action.

Lo único que le dio un hilo del que tirar fue que el mensaje **nombra el
archivo** que buscó — un `.nvmrc.backup` que él nunca configuró. Con un
`cat: No such file` genérico, ni siquiera habría sabido que el nombre era raro.

El síntoma aparece en el pipeline del consumidor; la causa está en un repositorio
ajeno. El mensaje de error es el único puente entre ambos.

---

## 12. Todos los inputs son strings

Los inputs de una action llegan siempre como string. `default: true` en YAML
produce el booleano, que el runner convierte a la cadena `"true"`.

```yaml
# INCORRECTO: la cadena "false" es truthy. La condición siempre pasa.
if: inputs.install

# CORRECTO
if: inputs.install == 'true'
```

**Decisión:** los defaults se escriben entrecomillados (`'true'`, `'false'`) y
todas las comparaciones son contra strings, para que el tipo real quede
explícito en el código.
