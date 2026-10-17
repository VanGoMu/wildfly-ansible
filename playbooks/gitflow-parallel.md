# Procedimiento: GitFlow para entregas solapadas

Este procedimiento define un flujo Git para mantener **tres entregas en paralelo** (`develop1`, `develop2`, `develop3`), con ramas de release independientes, soporte de versiones antiguas y propagación explícita de fixes. La regla central es: **`main` nunca se reescribe ni se rebasa; solo avanza mediante merges revisados desde las ramas de release.** `--no-ff` se usa para conservar un merge commit visible y auditable en cada entrega. [git-scm](https://git-scm.com/docs/git-merge)

## 1. Objetivos y reglas

| Regla | Decisión |
|---|---|
| Línea de publicación | `main` contiene únicamente versiones publicadas o aprobadas para publicación |
| Desarrollo por entrega | Cada entrega tiene su propia rama de integración: `develop1`, `develop2`, `develop3` |
| Estabilización | Cada entrega se congela en `release/X.Y` antes de publicar |
| Soporte de versiones | Las versiones que reciben parches mantienen `support/X.x` |
| Fixes compartidos | Se propagan mediante `cherry-pick` o merge explícito, nunca automáticamente |
| Historial | Se prohíbe `rebase`, `reset --hard` y force-push en `main`, `develop*`, `release/*` y `support/*` |
| Merges a `main` | Siempre mediante PR/MR, CI verde, revisión y `--no-ff` |
| Tags | Se crean sobre commits de `main`; los fixes de una versión se etiquetan desde su rama de soporte |

`git merge --no-ff` crea un merge commit incluso cuando sería posible un fast-forward, lo que resulta útil para preservar la topología de release en el historial.  `git cherry-pick -x` aplica un commit concreto en otra rama y añade una referencia al SHA original, lo que facilita la trazabilidad entre versiones. [git-scm](https://git-scm.com/docs/git-merge)

## 2. Convención de ramas

```text
main                      Última versión estable/publicada. Protegida.

develop1                  Integración de la entrega 1.
develop2                  Integración de la entrega 2.
develop3                  Integración de la entrega 3.

feature/1-auth-sso        Funcionalidad para la entrega 1.
feature/2-billing-v2      Funcionalidad para la entrega 2.
feature/3-ai-assistant    Funcionalidad para la entrega 3.

release/1.0               Congelación y estabilización de la entrega 1.
release/2.0               Congelación y estabilización de la entrega 2.
release/3.0               Congelación y estabilización de la entrega 3.

support/1.x               Mantenimiento de la versión 1 publicada.
support/2.x               Mantenimiento de la versión 2 publicada.

hotfix/1.0.1              Fix urgente para la versión 1.
hotfix/2.0.1              Fix urgente para la versión 2.
```

### Protección recomendada

- `main`: solo merges desde `release/*`; no push directo.
- `develop1`, `develop2`, `develop3`: merges desde `feature/*` y fixes propagados; PR obligatorio.
- `release/*`: solo fixes de estabilización, versionado y cambios aprobados para la entrega.
- `support/*`: solo fixes aprobados para esa versión.
- `feature/*`: se permite rebase local antes de abrir PR; después, solo merges o squash según la política del equipo.

## 3. Grafo completo del flujo

### Grafo conceptual

```text
main ─────────────────────────●──────────────────●──────────────────●────────→
                              │                  │                  │
                           merge v1.0         merge v2.0         merge v3.0
                              │                  │                  │
                              │ tag v1.0.0       │ tag v2.0.0       │ tag v3.0.0
                              │                  │                  │
release/1.0 ─────●──●─────────╯                  │                  │
                /   \                             │                  │
        release     hotfix                         │                  │
        fixes      v1.0.1                          │                  │
               \                                   │                  │
                └──────────────► support/1.x ──●───┘                  │
                                                │                      │
                                                │ cherry-pick fix      │
                                                │                      │
release/2.0 ────────────────●──●────────────────╯                      │
                           /   \                                        │
                   release     hotfix                                    │
                   fixes      v2.0.1                                     │
                          \                                              │
                           └──────────────► support/2.x ──●──────────────┘
                                                           │
                                                           │ cherry-pick fix
                                                           │
release/3.0 ────────────────────────────●──●──────────────╯
                                       /   \
                               release     hotfix
                               fixes      v3.0.1
```

### Grafo Mermaid

```mermaid
gitGraph
   commit id: "base"
   branch develop1
   checkout develop1
   commit id: "D1: infra base"
   branch feature/1-auth-sso
   checkout feature/1-auth-sso
   commit id: "F1: Keycloak client"
   commit id: "F1: SSO login"
   checkout develop1
   merge feature/1-auth-sso id: "merge F1" tag: "entrega1-ready"

   branch release/1.0
   checkout release/1.0
   commit id: "R1: bump 1.0.0-rc1"
   commit id: "R1: fix config"
   checkout main
   merge release/1.0 id: "RELEASE v1.0.0" tag: "v1.0.0"

   branch support/1.x
   checkout support/1.x
   commit id: "S1: branch from v1.0.0"

   checkout develop1
   branch hotfix/1.0.1
   checkout hotfix/1.0.1
   commit id: "H1: fix token refresh"
   checkout support/1.x
   merge hotfix/1.0.1 id: "merge hotfix 1.0.1" tag: "v1.0.1"

   checkout main
   merge hotfix/1.0.1 id: "backport hotfix to main"

   branch develop2
   checkout develop2
   commit id: "D2: billing scaffold"
   branch feature/2-billing-v2
   checkout feature/2-billing-v2
   commit id: "F2: invoice model"
   commit id: "F2: payment adapter"
   checkout develop2
   merge feature/2-billing-v2 id: "merge F2"

   branch release/2.0
   checkout release/2.0
   commit id: "R2: bump 2.0.0-rc1"
   commit id: "R2: remove v1 flag"
   checkout main
   merge release/2.0 id: "RELEASE v2.0.0" tag: "v2.0.0"

   branch support/2.x
   checkout support/2.x
   commit id: "S2: branch from v2.0.0"

   checkout develop2
   branch hotfix/2.0.1
   checkout hotfix/2.0.1
   commit id: "H2: fix rounding"
   checkout support/2.x
   merge hotfix/2.0.1 id: "merge hotfix 2.0.1" tag: "v2.0.1"

   checkout develop3
   commit id: "D3: AI assistant base"
   branch feature/3-ai-assistant
   checkout feature/3-ai-assistant
   commit id: "F3: RAG index"
   commit id: "F3: MCP tools"
   checkout develop3
   merge feature/3-ai-assistant id: "merge F3"

   checkout support/2.x
   commit id: "S2: security patch"
   checkout release/3.0
   merge support/2.x id: "propagate S2 fix" tag: "fix-shared"

   checkout main
   merge release/3.0 id: "RELEASE v3.0.0" tag: "v3.0.0"
```

> **Nota sobre el grafo:** en un repositorio real, `develop2` y `develop3` normalmente se crean desde un punto común de `main` o desde `develop1`, según cuándo comenzó cada entrega. El grafo representa tres líneas de trabajo solapadas, no una secuencia estricta.

## 4. Flujo de una entrega

### 4.1 Crear la línea de desarrollo

Cada entrega parte de un estado conocido y estable:

```bash
# Entrega 1
git switch main
git pull --ff-only
git switch -c develop1

# Entrega 2, creada mientras la 1 sigue activa
git switch develop1
git pull --ff-only
git switch -c develop2

# Entrega 3, creada mientras la 2 sigue activa
git switch develop2
git pull --ff-only
git switch -c develop3
```

Si las entregas no deben heredar código entre sí, créalas desde el mismo commit base:

```bash
BASE=$(git rev-parse main)

git switch -c develop1 "$BASE"
git switch -c develop2 "$BASE"
git switch -c develop3 "$BASE"
```

### 4.2 Trabajar en una feature

```bash
# Feature de la entrega 1
git switch develop1
git pull --ff-only
git switch -c feature/1-auth-sso

# Trabajo local
git add .
git commit -m "feat(auth): add Keycloak SSO client"

# Antes de abrir PR, opcionalmente rebase local
git fetch origin
git rebase origin/develop1

git push -u origin feature/1-auth-sso
```

Abrir PR/MR:

```text
feature/1-auth-sso → develop1
```

Requisitos del PR:

- CI completo en verde.
- Revisión aprobada.
- Sin secretos, credenciales ni artefactos generados.
- Tests de regresión para las áreas afectadas.
- Etiqueta de entrega: `entrega-1`, `entrega-2` o `entrega-3`.

### 4.3 Integrar la feature

```bash
git switch develop1
git pull --ff-only
git merge --no-ff feature/1-auth-sso
git push origin develop1
```

El uso de `--no-ff` conserva un commit de fusión explícito, útil para identificar cuándo entró cada feature en la entrega. [git-scm](https://git-scm.com/docs/git-merge)

## 5. Congelar y publicar una entrega

### 5.1 Crear release

Se crea cuando la entrega está funcionalmente completa y entra en estabilización:

```bash
git switch develop1
git pull --ff-only
git switch -c release/1.0
git push -u origin release/1.0
```

A partir de aquí:

- `develop1` puede seguir recibiendo trabajo para una futura entrega, si aplica.
- `release/1.0` solo recibe fixes de estabilización, bump de versión, documentación y cambios aprobados.
- No se añaden features nuevas sin una decisión explícita de scope.

### 5.2 Estabilizar

```bash
git switch release/1.0
git pull --ff-only

# Fix de estabilización
git switch -c fix/1.0-release-config
# ...cambios...
git add .
git commit -m "fix(release): correct production configuration"

git push -u origin fix/1.0-release-config
```

PR:

```text
fix/1.0-release-config → release/1.0
```

Merge:

```bash
git switch release/1.0
git pull --ff-only
git merge --no-ff fix/1.0-release-config
git push origin release/1.0
```

### 5.3 Publicar en main

```bash
git switch main
git pull --ff-only

git merge --no-ff release/1.0
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin main
git push origin v1.0.0
```

Después de publicar, crear la rama de soporte:

```bash
git switch -c support/1.x v1.0.0
git push -u origin support/1.x
```

## 6. Fixes y soporte de versiones

### 6.1 Fix urgente para la versión 1

El fix parte de `support/1.x`, no de `main` ni de `develop2`/`develop3`:

```bash
git switch support/1.x
git pull --ff-only
git switch -c hotfix/1.0.1

# Corregir el problema
git add .
git commit -m "fix(auth): refresh expired tokens before retry"

git push -u origin hotfix/1.0.1
```

PR:

```text
hotfix/1.0.1 → support/1.x
```

Publicar el parche:

```bash
git switch support/1.x
git pull --ff-only
git merge --no-ff hotfix/1.0.1

git tag -a v1.0.1 -m "Hotfix 1.0.1"
git push origin support/1.x
git push origin v1.0.1
```

### 6.2 Propagar el fix a versiones posteriores

Solo si el fix es válido para la entrega 2:

```bash
git log support/1.x --oneline -5
# Localiza el SHA del fix, por ejemplo: a1b2c3d

git switch develop2
git pull --ff-only
git cherry-pick -x a1b2c3d
git push origin develop2
```

Si el fix debe llegar también a la release 3:

```bash
git switch release/3.0
git pull --ff-only
git cherry-pick -x a1b2c3d
git push origin release/3.0
```

`cherry-pick -x` añade al mensaje una referencia al commit original, lo que ayuda a evitar duplicidades y facilita auditoría. [git-scm](https://git-scm.com/docs/git-cherry-pick)

### 6.3 Decisión de propagación

| Fix | ¿A `develop2`? | ¿A `release/2.0`? | ¿A `develop3`? | ¿A `release/3.0`? |
|---|---:|---:|---:|---:|
| Bug presente en todas las versiones | Sí | Sí | Sí | Sí |
| Bug solo de la versión 1 | No | No | No | No |
| Vulnerabilidad de dependencia común | Sí | Sí | Sí | Sí |
| Cambio de configuración específico de v1 | No | No | No | No |
| Fix de un componente compartido | Sí | Sí | Sí | Sí |
| Fix que depende de una API eliminada en v2/v3 | No | No | Evaluar | Evaluar |

## 7. Problemas típicos y resolución

### Problema 1: la entrega 2 necesita reemplazar código de la entrega 1

**Síntoma:** `develop2` contiene cambios incompatibles con `release/1.0`, pero `main` debe avanzar a la versión 2.

**Solución:** no rebasar `main`. Publica la 1 desde `release/1.0`, etiqueta `v1.0.0`, crea `support/1.x` y publica la 2 desde `release/2.0`.

```bash
# Publicar v1
git switch main
git merge --no-ff release/1.0
git tag -a v1.0.0 -m "Release 1.0.0"
git push origin main --tags

# Mantener v1
git switch -c support/1.x v1.0.0
git push -u origin support/1.x

# Publicar v2
git switch main
git merge --no-ff release/2.0
git tag -a v2.0.0 -m "Release 2.0.0"
git push origin main --tags
```

**Lo que no se debe hacer:**

```bash
# PROHIBIDO en main
git rebase release/2.0
git reset --hard release/2.0
git push --force origin main
```

### Problema 2: un fix de v1 rompe la v2

**Síntoma:** el cherry-pick del fix `a1b2c3d` genera conflictos o tests fallidos en `develop2`.

```bash
git switch develop2
git cherry-pick -x a1b2c3d
# Conflicto
```

**Resolución:**

```bash
# Resolver conflictos
git status
# Editar ficheros
git add .
git cherry-pick --continue

# O cancelar si el fix no aplica
git cherry-pick --abort
```

Si el fix no es aplicable, documéntalo en el PR o issue de propagación:

```text
Fix v1.0.1 no portado a v2: la API X fue sustituida por Y en la entrega 2.
El equivalente se implementará en issue #123.
```

### Problema 3: dos equipos modifican el mismo fichero en entregas distintas

**Síntoma:** `feature/2-billing-v2` y `feature/3-ai-assistant` cambian el mismo módulo.

**Mitigación:**

- Mantener los cambios lo más pequeños posible.
- Integrar con frecuencia en `develop2` y `develop3`.
- Usar feature flags para desacoplar activación funcional de despliegue.
- Establecer un owner del módulo compartido.
- Evitar “robar” cambios de otra entrega mediante merges masivos.

```bash
# Sincronizar antes de seguir trabajando
git switch feature/3-ai-assistant
git fetch origin
git rebase origin/develop3
```

### Problema 4: un fix debe ir a varias releases

**Síntoma:** hay que corregir una vulnerabilidad en v1, v2 y v3.

**Solución:** crea un fix canónico, revísalo y propágalos con cherry-picks separados.

```bash
# Fix base en la versión más antigua afectada
git switch support/1.x
git switch -c hotfix/1.2.1
# ...fix...
git commit -m "fix(security): upgrade vulnerable dependency"

# Propagar
git switch support/2.x
git cherry-pick -x <sha>

git switch release/3.0
git cherry-pick -x <sha>
```

No hagas un único merge de `support/1.x` en todas las ramas: podrías arrastrar cambios específicos de la versión 1.

### Problema 5: alguien hizo rebase de una rama compartida

**Síntoma:** `develop2` tiene hashes distintos en local y remoto.

```bash
git fetch origin
git log --oneline --graph --decorate --all -30
```

**Resolución segura:**

- Identificar los commits duplicados.
- Crear una rama de recuperación desde el estado remoto.
- No hacer force-push sin coordinación.
- Revisar tags, pipelines y PRs afectados.

```bash
git switch -c recovery/develop2 origin/develop2
# Comparar y recuperar cambios necesarios
git log --oneline develop2..recovery/develop2
```

## 8. Comandos de referencia

### Estado y sincronización

```bash
git fetch --prune
git switch develop2
git pull --ff-only

git log --oneline --graph --decorate --all -40
git branch -vv
git tag --sort=-creatordate
```

### Features

```bash
git switch develop2
git switch -c feature/2-billing-v2

git add .
git commit -m "feat(billing): add invoice model"

git fetch origin
git rebase origin/develop2
git push -u origin feature/2-billing-v2
```

### Release

```bash
git switch develop2
git switch -c release/2.0
git push -u origin release/2.0
```

### Publicación

```bash
git switch main
git pull --ff-only
git merge --no-ff release/2.0
git tag -a v2.0.0 -m "Release 2.0.0"
git push origin main
git push origin v2.0.0
```

### Soporte

```bash
git switch -c support/2.x v2.0.0
git push -u origin support/2.x
```

### Hotfix

```bash
git switch support/2.x
git switch -c hotfix/2.0.1
# ...fix...
git commit -m "fix(billing): correct invoice rounding"

git push -u origin hotfix/2.0.1
```

### Propagación

```bash
git switch develop3
git cherry-pick -x <sha>
git push origin develop3
```

## 9. Checklist de release

Antes de fusionar `release/X.Y` en `main`:

- [ ] Todas las features aprobadas están mergeadas en la rama de desarrollo correspondiente.
- [ ] `release/X.Y` no contiene features fuera de scope.
- [ ] CI, tests de integración y tests de regresión están en verde.
- [ ] La versión se ha incrementado correctamente.
- [ ] Las notas de release están preparadas.
- [ ] Los cambios de configuración y migraciones están revisados.
- [ ] Se ha decidido qué fixes de versiones anteriores deben propagarse.
- [ ] `main` está actualizado y protegido.
- [ ] El merge a `main` es `--no-ff`.
- [ ] Se crea un tag anotado.
- [ ] Se crea o actualiza `support/X.x` si la versión recibirá parches.

## 10. Resumen operativo

- **Desarrollo:** `feature/*` → `developN`.
- **Estabilización:** `developN` → `release/X.Y`.
- **Publicación:** `release/X.Y` → `main` + tag.
- **Mantenimiento:** `support/X.x` + `hotfix/X.Y.Z` + tag.
- **Compatibilidad:** cherry-pick o merge explícito entre versiones, con revisión y CI.
- **Prohibido:** rebase, reset o force-push en ramas compartidas, especialmente `main`. [git-scm](https://git-scm.com/docs/git-merge)
