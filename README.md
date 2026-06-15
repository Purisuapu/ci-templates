# ci-templates

Workflows reutilizables de GitHub Actions para toda la organización. Los proyectos los invocan en lugar de implementar su propia lógica de CI/CD.

---

## Workflows disponibles

| Workflow | Descripción |
|---|---|
| `build.yml` | Instala dependencias y compila el proyecto (Node.js) |
| `test.yml` | Ejecuta la suite de tests |
| `docker.yml` | Build y push de imagen Docker a GHCR |
| `deploy.yml` | Deploy vía SSH + docker compose + notificación a Discord |
| `restart.yml` | Reinicia un compose project vía SSH |
| `ssh-run.yml` | Ejecuta un script bash arbitrario en el servidor vía SSH |
| `ssh-test.yml` | Verifica la conexión SSH y que Docker esté disponible en el servidor |
| `release.yml` | Crea GitHub Release con notas automáticas + notificación a Discord |
| `renovate-notify.yml` | Notifica a Discord cuando Renovate abre una PR |

---

## Documentación

- [Guía de uso completa](docs/guia-uso.md) — ejemplos detallados, todos los inputs y solución de problemas

---

## Cómo usarlos

Desde cualquier repositorio del proyecto, en el workflow `ci.yml`:

```yaml
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
    uses: PURISUAPU/ci-templates/.github/workflows/docker.yml@main
    with:
      image-name: nombre-del-servicio
    secrets: inherit

  deploy:
    needs: docker
    uses: PURISUAPU/ci-templates/.github/workflows/deploy.yml@main
    with:
      service-name: nombre-del-servicio
    secrets: inherit
```

`secrets: inherit` pasa automáticamente todos los secrets que `org-admin` sincronizó en el repo.

---

## Referencia de inputs

### `build.yml`

| Input | Descripción | Default |
|---|---|---|
| `node-version` | Versión de Node.js | `20` |
| `working-directory` | Directorio de trabajo | `.` |
| `install-command` | Comando de instalación | `npm ci` |
| `build-command` | Comando de build (vacío = omitir) | `npm run build --if-present` |

### `test.yml`

| Input | Descripción | Default |
|---|---|---|
| `node-version` | Versión de Node.js | `20` |
| `working-directory` | Directorio de trabajo | `.` |
| `test-command` | Comando de tests | `npm test` |

### `docker.yml`

| Input | Descripción | Default |
|---|---|---|
| `image-name` | Nombre de la imagen en GHCR | **requerido** |
| `dockerfile` | Ruta al Dockerfile | `Dockerfile` |
| `context` | Contexto de build | `.` |
| `push` | Subir imagen al registry | `true` |

### `deploy.yml`

| Input | Descripción | Default |
|---|---|---|
| `service-name` | Nombre del servicio (= nombre del repo) | **requerido** |
| `deploy-path` | Ruta en el servidor donde está el `docker-compose.yml` | `/opt/<service-name>` |
| `compose-file` | Nombre del archivo compose en el repo | `docker-compose.yaml` |
| `sync-files` | Archivos adicionales a copiar al servidor (separados por coma) | `''` |
| `environment` | Entorno (`production` / `staging`) | `production` |
| `notify-discord` | Notificar a Discord | `true` |

Variables de org requeridas (llegan vía `org-admin`): `SERVER_IP`, `DEPLOY_SSH_USER`.
Secret requerido: `SSH_PRIVATE_KEY`. Secrets opcionales: `DISCORD_WEBHOOK`, `GHCR_READ_TOKEN` (si el servicio usa una imagen privada de GHCR, el workflow hace `docker login ghcr.io` en el servidor antes del pull).

> **Nota:** la copia de archivos usa SSH+stdin (`cat > remote-file < local-file`) en lugar de SCP para evitar la dependencia del subsistema SFTP.

La notificación Discord incluye el campo **Source** con el evento que disparó el deploy (`push`, `workflow_dispatch`, `schedule`).

### `restart.yml`

| Input | Descripción | Default |
|---|---|---|
| `compose-name` | Nombre del proyecto docker compose a reiniciar | **requerido** |
| `notify-discord` | Notificar a Discord | `false` |

Secret requerido: `SSH_PRIVATE_KEY`. Secret opcional: `DISCORD_WEBHOOK`.

### `ssh-run.yml`

| Input | Descripción | Default |
|---|---|---|
| `script` | Script bash a ejecutar en el servidor | **requerido** |
| `timeout` | Tiempo máximo de ejecución (ej: `30m`, `120m`) | `30m` |

Secret requerido: `SSH_PRIVATE_KEY`. Variables requeridas: `SERVER_IP`, `DEPLOY_SSH_USER`.

### `release.yml`

| Input | Descripción | Default |
|---|---|---|
| `draft` | Crear como borrador | `false` |
| `prerelease` | Marcar como pre-release | `false` |
| `generate-notes` | Generar notas desde commits | `true` |
| `notify-discord` | Notificar a Discord | `true` |

Secret opcional: `DISCORD_WEBHOOK`.

---

## Estructura

```
ci-templates/
└── .github/
    └── workflows/
        ├── build.yml
        ├── test.yml
        ├── docker.yml
        ├── deploy.yml
        ├── restart.yml
        ├── ssh-run.yml
        ├── ssh-test.yml
        ├── release.yml
        └── renovate-notify.yml
```

---

## Reglas

- Este repo no contiene código de aplicación, solo workflows.
- Está excluido de la sincronización de secrets/variables de `org-admin`.
- Siempre referenciar desde `@main` para recibir actualizaciones automáticamente.
- Cambios aquí afectan a todos los proyectos — testear antes de mergear a `main`.
