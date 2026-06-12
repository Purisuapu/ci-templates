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

`secrets: inherit` pasa automáticamente todos los secrets y variables que `org-admin` sincronizó en el repo — no es necesario declararlos uno a uno.

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
    if: github.ref == 'refs/heads/main'
    uses: PURISUAPU/ci-templates/.github/workflows/deploy.yml@main
    with:
      service-name: nombre-del-servicio
      # deploy-path: '/opt/mi-servicio'  # opcional, por defecto /opt/<service-name>
    secrets: inherit

  release:
    needs: docker
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

### `deploy.yml` — Deploy vía SSH

Se conecta al servidor vía SSH, hace `docker compose pull` y reinicia el servicio. Notifica el resultado a Discord.

```yaml
jobs:
  deploy:
    uses: PURISUAPU/ci-templates/.github/workflows/deploy.yml@main
    with:
      service-name: mi-servicio          # Requerido — nombre del repo/servicio
      deploy-path: /opt/mi-servicio      # Opcional — por defecto /opt/<service-name>
      environment: production            # Opcional: production | staging
      notify-discord: true               # Opcional
    secrets: inherit
```

**Variables de org requeridas** (sincronizadas desde `org-admin`):
- `SERVER_IP` — IP o dominio del servidor
- `DEPLOY_SSH_USER` — usuario SSH

**Secret requerido** (sincronizado desde `org-admin`):
- `SSH_PRIVATE_KEY` — clave privada SSH

**Prerrequisito en el servidor:** el directorio `/opt/<service-name>/` debe existir con un `docker-compose.yml` que use la imagen de GHCR. El usuario SSH debe tener la clave pública correspondiente a `DEPLOY_SSH_KEY` en `~/.ssh/authorized_keys`.

**Qué ejecuta en el servidor:**
```bash
cd /opt/<service-name>
docker compose pull
docker compose up -d --remove-orphans
docker image prune -f
```

---

### `release.yml` — GitHub Release

Crea la release en GitHub con notas generadas automáticamente desde los commits, y notifica a Discord.

```yaml
jobs:
  release:
    uses: PURISUAPU/ci-templates/.github/workflows/release.yml@main
    with:
      draft: false          # Opcional — true crea borrador sin notificar Discord
      prerelease: false     # Opcional
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
| `SSH: handshake failed` en deploy.yml | Clave pública no instalada en el servidor | Añadir la clave pública de `DEPLOY_SSH_KEY` en `~/.ssh/authorized_keys` del servidor |
| `no such file or directory` en deploy.yml | El directorio del servicio no existe en el servidor | Crear `/opt/<service-name>/` y añadir el `docker-compose.yml` |
| `docker compose pull` falla con 401 | El servidor no tiene acceso a GHCR | Hacer `docker login ghcr.io` en el servidor con un PAT |
| Release no genera notas | No hay commits convencionales desde la última release | Las notas automáticas requieren el formato `feat:`, `fix:`, etc. |
| El workflow no aparece en el repo destino | El archivo `.github/workflows/ci.yml` no existe en ese repo | Asegurarse de que el repo se creó desde `template-project` |
