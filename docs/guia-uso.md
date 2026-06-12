# Guía de uso — ci-templates

Cómo invocar los workflows reutilizables desde cualquier proyecto de la organización.

---

## Concepto básico

Los workflows de `ci-templates` usan el trigger `workflow_call`. Un proyecto los invoca así:

```yaml
jobs:
  mi-job:
    uses: PURISUAPU/ci-templates/.github/workflows/nombre.yml@main
    with:
      input-1: valor
    secrets: inherit
```

`secrets: inherit` pasa automáticamente todos los secrets que `org-admin` sincronizó en el repo — no es necesario declararlos uno a uno.

---

## Pipeline completo

El [`template-project`](https://github.com/PURISUAPU/template-project) incluye este pipeline como punto de partida. Cópialo y ajusta los inputs al proyecto:

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  release:
    types: [published]

jobs:
  build:
    uses: PURISUAPU/ci-templates/.github/workflows/build.yml@main
    secrets: inherit

  test:
    needs: build
    uses: PURISUAPU/ci-templates/.github/workflows/test.yml@main
    secrets: inherit

  docker:
    needs: test
    if: github.ref == 'refs/heads/main' || github.event_name == 'release'
    uses: PURISUAPU/ci-templates/.github/workflows/docker.yml@main
    with:
      image-name: nombre-del-servicio
    secrets: inherit

  deploy:
    needs: docker
    if: github.ref == 'refs/heads/main' || github.event_name == 'release'
    uses: PURISUAPU/ci-templates/.github/workflows/deploy.yml@main
    with:
      service-name: nombre-del-servicio
      portainer-url: https://portainer.example.com
      image-name: nombre-del-servicio
    secrets: inherit

  release:
    needs: deploy
    if: github.event_name == 'release'
    uses: PURISUAPU/ci-templates/.github/workflows/release.yml@main
    secrets: inherit
```

---

## Workflows individuales

### `build.yml` — Instalación y compilación

Instala dependencias Node.js, compila el proyecto y sube los artifacts para pasos posteriores.

```yaml
jobs:
  build:
    uses: PURISUAPU/ci-templates/.github/workflows/build.yml@main
    with:
      node-version: '20'           # Opcional, default: '20'
      working-directory: '.'       # Opcional, default: '.'
      install-command: 'npm ci'    # Opcional
      build-command: 'npm run build --if-present'  # Opcional, vacío = omitir
    secrets: inherit
```

**Cuándo usarlo solo:** en PRs donde solo quieres verificar que el proyecto compila.

---

### `test.yml` — Tests

Ejecuta la suite de tests. Requiere que las dependencias estén instaladas (usa `npm ci` internamente).

```yaml
jobs:
  test:
    needs: build
    uses: PURISUAPU/ci-templates/.github/workflows/test.yml@main
    with:
      node-version: '20'        # Opcional
      working-directory: '.'    # Opcional
      test-command: 'npm test'  # Opcional
    secrets: inherit
```

**Cuándo usarlo solo:** para ejecutar tests sin build previo (proyectos sin paso de compilación).

---

### `docker.yml` — Build y push de imagen

Construye la imagen Docker y la sube a GHCR (`ghcr.io/PURISUAPU/<image-name>`).

```yaml
jobs:
  docker:
    uses: PURISUAPU/ci-templates/.github/workflows/docker.yml@main
    with:
      image-name: mi-servicio     # Requerido
      dockerfile: 'Dockerfile'    # Opcional
      context: '.'                # Opcional
      push: true                  # Opcional, false = solo build sin push
    secrets: inherit
```

**Tags generados automáticamente:**
- `sha-<7-chars>` — por cada commit
- `main` — al hacer push a main
- `1.2.3` y `1.2` — al publicar una release semántica

**Requisito:** el repo necesita permiso de escritura en GHCR. Se otorga automáticamente vía `GITHUB_TOKEN` con el permiso `packages: write` incluido en el workflow.

---

### `deploy.yml` — Deploy en Portainer

Actualiza el stack en Portainer y notifica el resultado a Discord.

```yaml
jobs:
  deploy:
    uses: PURISUAPU/ci-templates/.github/workflows/deploy.yml@main
    with:
      service-name: mi-servicio                    # Requerido — nombre del stack en Portainer
      portainer-url: https://portainer.example.com # Requerido
      image-name: mi-servicio                      # Requerido — nombre de la imagen Docker
      environment: production                       # Opcional: production | staging
      notify-discord: true                          # Opcional
    secrets: inherit
```

**Secrets necesarios** (llegan vía `org-admin`):
- `PORTAINER_TOKEN` — API key de Portainer
- `DISCORD_WEBHOOK` — para la notificación (solo si `notify-discord: true`)

**Cómo funciona el redeploy:** envía un POST al webhook del stack en Portainer con la nueva imagen. Portainer se encarga de hacer pull y reiniciar el contenedor.

---

### `release.yml` — GitHub Release

Crea la release en GitHub con notas generadas automáticamente desde los commits, y notifica a Discord.

```yaml
jobs:
  release:
    uses: PURISUAPU/ci-templates/.github/workflows/release.yml@main
    with:
      draft: false        # Opcional — true crea borrador sin notificar Discord
      prerelease: false   # Opcional
      generate-notes: true  # Opcional — notas automáticas desde commits
      notify-discord: true  # Opcional
    secrets: inherit
```

**Cuándo se ejecuta:** solo cuando el evento es `release: published`. No se ejecuta en push normal.

**Cómo crear una release:**
> GitHub → Repositorio → Releases → Draft a new release → crear tag semántico (`v1.2.3`) → Publish release

---

## Añadir un nuevo workflow a ci-templates

1. Crea el archivo en `.github/workflows/nombre.yml` con `on: workflow_call`.
2. Define los `inputs` y `secrets` que necesita.
3. Documenta el nuevo workflow en este archivo y en el `README.md`.
4. Haz merge a `main` — todos los proyectos que lo usen recibirán los cambios automáticamente.

> Cuidado: los cambios en `ci-templates@main` afectan a todos los proyectos en el mismo deploy. Prueba en una rama antes de mergear.

---

## Solución de problemas

| Síntoma | Causa probable | Solución |
|---|---|---|
| `Context access might be invalid: SECRET_NAME` | El linter del IDE no conoce los secrets del repo | Es solo una advertencia, el workflow funciona correctamente en runtime |
| `Error: buildx failed` en docker.yml | El Dockerfile tiene un error de sintaxis | Revisar el Dockerfile localmente con `docker build .` |
| Deploy a Portainer devuelve 401 | `PORTAINER_TOKEN` expirado o incorrecto | Regenerar el token en Portainer y actualizar el secret en `org-admin` |
| Release no genera notas | No hay commits convencionales desde la última release | Las notas automáticas requieren el formato `feat:`, `fix:`, etc. |
| El workflow no aparece en el repo destino | El archivo `.github/workflows/ci.yml` no existe en ese repo | Asegurarse de que el repo se creó desde `template-project` |
