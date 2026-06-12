# ci-templates

Workflows reutilizables de GitHub Actions para toda la organización. Los proyectos los invocan en lugar de implementar su propia lógica de CI/CD.

---

## Workflows disponibles

| Workflow | Descripción |
|---|---|
| `build.yml` | Instala dependencias y compila el proyecto (Node.js) |
| `test.yml` | Ejecuta la suite de tests |
| `docker.yml` | Build y push de imagen Docker a GHCR |
| `deploy.yml` | Redeploy en Portainer + notificación a Discord |
| `release.yml` | Crea GitHub Release con notas automáticas + notificación a Discord |

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
      portainer-url: https://portainer.example.com
      image-name: nombre-del-servicio
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
| `service-name` | Nombre del stack en Portainer | **requerido** |
| `portainer-url` | URL base de Portainer | **requerido** |
| `image-name` | Nombre de la imagen Docker | **requerido** |
| `environment` | Entorno (`production` / `staging`) | `production` |
| `notify-discord` | Notificar a Discord | `true` |

Secrets requeridos (llegan vía `org-admin`): `PORTAINER_TOKEN`, `DISCORD_WEBHOOK`.

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
        └── release.yml
```

---

## Reglas

- Este repo no contiene código de aplicación, solo workflows.
- Está excluido de la sincronización de secrets/variables de `org-admin`.
- Siempre referenciar desde `@main` para recibir actualizaciones automáticamente.
- Cambios aquí afectan a todos los proyectos — testear antes de mergear a `main`.
