# RUNBOOK — ci-preflight

Documento para quien mantiene la action. Si buscas cómo usarla, ve al README.

---

## Modelo mental

Esta action es un artefacto de software con consumidores. Cada release afecta a
repositorios sobre los que no tienes visibilidad. Las dos consecuencias:

1. **El `action.yml` es una API pública.** Los `id` de los steps internos no lo
   son. Puedes renombrarlos libremente; no puedes tocar `inputs:` ni `outputs:`
   sin considerar semver.
2. **Mover el tag `v1` es un despliegue a producción.** No hay canary, no hay
   rollout gradual: el siguiente job de cada consumidor recibe el cambio.

---

## Publicar un release

### Orden de operaciones

El orden importa y cada paso tiene un porqué.

```
1. main verde                    ── nunca taguees sin CI en verde
2. Crear tag inmutable vX.Y.Z    ── el artefacto real
3. Crear GitHub Release          ── las notas alimentan los PRs de Dependabot
4. Verificar desde un consumidor ── último punto donde puedes retractarte
5. Mover el tag mayor vX         ── PUNTO DE NO RETORNO
```

**Por qué el paso 4 va antes del 5.** Mientras `v1` no se haya movido, ningún
consumidor recibió el cambio. Si algo está mal, borras el tag `vX.Y.Z` y nadie
se enteró. Después del paso 5, todos lo tienen.

### Procedimiento

```bash
# 1. Verificar que main está verde
git checkout main && git pull
gh run list --workflow=smoke.yml --limit 1

# 2. Tag anotado (NO lightweight: guarda autor, fecha y mensaje,
#    que es la única auditoría de quién publicó qué)
git tag -a v1.1.0 -m "Release v1.1.0: <resumen>"
git push origin v1.1.0

# 3. Release con notas útiles
gh release create v1.1.0 --title "v1.1.0" --notes "..."

# 4. VERIFICAR desde consumer-sandbox antes de continuar.
#    Cambiar la referencia a @v1.1.0 y ejecutar.

# 5. Mover el tag mayor
git tag -fa v1 -m "Actualizar v1 -> v1.1.0"
git push origin v1 --force
```

### Qué escribir en las notas de release

Cuando un consumidor pinea por SHA, Dependabot le abre un PR y **pega estas
notas en la descripción**. Es lo único que el revisor va a leer.

Incluir:

- Qué cambió, en términos del comportamiento observable
- Si algún default cambió
- Si el efecto sobre el entorno del job cambió
- Requisitos nuevos del consumidor

No incluir: lista de commits. GitHub ya la genera aparte.

---

## Decidir la versión

El contrato es el `action.yml` más los efectos sobre el entorno del job.

| Cambio | Versión |
|---|---|
| Corregir un bug sin cambiar el contrato | Patch |
| Agregar un input **cuyo default preserva el comportamiento actual** | Minor |
| Agregar un output | Minor |
| Mejorar un mensaje de error | Patch |
| Agregar un input cuyo default **cambia** el comportamiento | **Major** |
| Cambiar el default de un input existente | **Major** |
| Agregar un input requerido sin default | **Major** |
| Quitar o renombrar un input u output | **Major** |
| Agregar un step que modifica `PATH` o `GITHUB_ENV` | **Major** |

### La regla que más se equivoca

> Un input opcional no es retrocompatible por ser opcional. Es retrocompatible
> si su **default preserva el comportamiento anterior**.

Ejemplo: agregar `strict-engines` con default `'true'` hace fallar a consumidores
que no tocaron nada. Es major, aunque el input sea "opcional".

---

## Deprecar un comportamiento

Nunca cambies un default de golpe. Tres pasos, con al menos un release de
distancia entre cada uno:

```
vX.Y.0   Agregar el input con el default que preserva el comportamiento actual.
         Quien quiera el nuevo, opta explícitamente.

vX.Z.0   Con el default sin cambiar, emitir un warning cuando se detecte la
         condición:
           echo "::warning title=Deprecado::Esto será un error en v(X+1)"

v(X+1).0.0  Cambiar el default. Documentar la migración en las notas de release.
```

El warning intermedio es lo que le da al consumidor la oportunidad de reaccionar
antes de que su pipeline se rompa. Saltárselo convierte una migración ordenada
en un incidente.

---

## Revertir un release malo

### Diagnóstico

```bash
# ¿A qué commit apunta v1 ahora?
git ls-remote --tags origin
gh api repos/mantoniocc/ci-preflight/commits/v1 --jq .sha
```

### Procedimiento

```bash
# 1. Revertir el código con git revert, NO con reset --hard.
#    reset reescribe la historia de una rama que otros pueden tener
#    clonada. revert agrega un commit que deshace, dejando rastro
#    auditable de que hubo un bug y de que se corrigió.
git revert <sha-del-commit-malo> --no-edit
git push

# 2. ESPERAR a que CI confirme que main está sano otra vez.
#    Este es el paso que se salta la gente con prisa, y es el que
#    evita publicar un segundo bug encima del primero.
gh run list --workflow=smoke.yml --limit 1

# 3. Mover v1 al commit sano
git tag -fa v1 -m "Rollback: v1 -> <sha sano>"
git push origin v1 --force

# 4. Publicar un patch para quienes pinean por versión o SHA.
#    NO reciben el rollback del tag móvil: hay que darles una
#    versión nueva a la que puedan actualizar.
git tag -a v1.1.1 -m "Release v1.1.1: revierte el bug de v1.1.0"
git push origin v1.1.1
gh release create v1.1.1 --title "v1.1.1" --notes "Revierte <descripción>."
```

**El paso 4 es el que más se olvida.** Mover `v1` arregla a los consumidores del
tag móvil. Los que pinean por SHA o por versión exacta siguen en la versión
rota hasta que exista una versión nueva. Un rollback que solo mueve el tag deja
a parte de tus consumidores rotos.

### Qué NO hacer

| Acción | Por qué no |
|---|---|
| Borrar el tag `v1.1.0` | Quien ya lo pineó pasa de "versión con bug" a "referencia inexistente". Peor. |
| `git reset --hard` sobre `main` | Reescribe historia compartida |
| Mover `v1` sin esperar a CI | Riesgo de publicar un segundo bug encima del primero |

---

## Cómo se ve un incidente desde el consumidor

Contexto para dimensionar la urgencia: cuando el tag `v1` apunta a código roto,
el consumidor **no ve ninguna referencia a la action** en el error. Ve un fallo
sobre sus propios archivos, en un repositorio que no cambió.

Solo llega a la causa si se le ocurre revisar el historial de tags de un
repositorio ajeno. En la práctica, eso significa que el tiempo de diagnóstico
del consumidor es de horas, no de minutos.

Consecuencia operativa: **cuando detectes que publicaste un release malo,
revierte primero y comunica después.** Cada minuto adicional multiplica los
pipelines rotos.

---

## Mantenimiento periódico

### Dependabot

Configurado en `.github/dependabot.yml`, monitorea tanto `.github/workflows/`
como el `action.yml` de esta composite action (verificado en el dependency
graph).

Al revisar un PR de Dependabot:

1. Leer las release notes de la action actualizada
2. Verificar que el comentario del tag junto al SHA quedó actualizado
3. Confirmar que `smoke.yml` pasa
4. Si es un cambio mayor de una action interna, considerar si el
   comportamiento de esta action cambia para sus consumidores

### Verificar que los tags están consistentes

```bash
# v1 debe apuntar al mismo commit que el último vX.Y.Z de la línea
git fetch --tags --force
git ls-remote --tags origin | sort -k2
```

---

## Limitaciones internas conocidas

Fragilidades de la implementación que no afectan el contrato público y no están
documentadas en el README, porque el consumidor no puede actuar sobre ellas.

### El guard de manifiestos depende del Node preinstalado del runner

El step `Validar manifiestos npm` usa `node -p` para leer `lockfileVersion`,
pero se ejecuta **antes** de `setup-node`. Funciona porque las imágenes de los
runners GitHub-hosted traen Node preinstalado.

| | |
|---|---|
| Riesgo | Falla en un runner self-hosted sin Node, o si alguien reordena los steps |
| Mitigación evaluada | Mover el guard después de `setup-node` |
| Por qué no se aplicó | Perderíamos el fail-fast: el objetivo del guard es abortar antes de instalar Node, que es la operación cara |
| Alternativa futura | Reemplazar `node -p` por `grep`/`sed` sobre el JSON, a costa de robustez del parseo |

Si se reordenan los steps, verificar que este guard siga ejecutándose antes de
`setup-node`.

---

## Estructura del repositorio

```
ci-preflight/
├── action.yml                    ← el contrato público
├── .nvmrc                        ← fixture para el smoke test
├── package.json                  ← fixture, dependencia trivial
├── package-lock.json             ← fixture
├── .github/
│   ├── dependabot.yml
│   └── workflows/smoke.yml       ← consume la action contra sí misma con $/
├── README.md                     ← para consumidores
├── RUNBOOK.md                    ← este archivo
└── docs/action-references.md     ← referencia sobre resolución y versionado
```

El `smoke.yml` usa `$/` en vez de `./` para que la action se resuelva contra el
commit que ejecuta el workflow, sin depender de qué haya traído el checkout.

---

## Convenciones del código

| Convención | Por qué |
|---|---|
| Inputs a `run:` vía `env:`, no interpolados en el script | Un valor interpolado se convierte en código; uno en `env` es un dato. Previene inyección de comandos |
| `set -euo pipefail` en cada `run:` | `shell: bash` ya aporta `-e` y `pipefail`; el `set` agrega `-u` y hace el script correcto si se copia a otro contexto |
| Guards que fallan antes de operaciones caras | Ahorra minutos y evita entradas de cache basura |
| Sin `continue-on-error` en steps internos | Deja una anotación roja en un run verde que el consumidor no puede explicar. Manejar el fallo dentro del script |
| Steps que emiten outputs no llevan `if:` | Un step condicional no emite outputs cuando se salta, dejando el contrato incompleto |
| Verificar el resultado, no solo la entrada | El guard de versión instalada existe porque `setup-node` con versión vacía no falla: usa el Node del runner |

### Cuándo agregar un guard

Un guard vale la pena cuando su ausencia produce un **falso verde**. Cuando su
ausencia produce un rojo que ya era claro, es ruido.

- No validamos la sincronización del lockfile: `npm ci` ya falla con un mensaje
  claro. Duplicarlo no aporta.
- Sí validamos la versión instalada: nadie más lo detecta, y sin el guard el
  pipeline reporta verde con la versión equivocada.