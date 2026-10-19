# Workflows, CI/CD y despliegue

## Flujos de negocio principales

1. **Programar y publicar un post**: el usuario crea el post en el calendario (frontend `launches`), el backend lo guarda (`POST /posts`) y se inicia un workflow de Temporal que lo publica a la hora programada mediante el proveedor del canal.
2. **Conectar un canal**: flujo OAuth por proveedor (`/integrations`, `/integrations/social/:integration`), con token almacenado en la tabla `Integration`. El workflow `refreshTokenWorkflow` renueva tokens.
3. **Autopost** (`/autopost`, `autoPostWorkflow`): publicación automática a partir de fuentes (p. ej. feeds RSS; el detalle exacto es **Por confirmar**).
4. **Correos**: `sendEmailWorkflow` y `digestEmailWorkflow` (señales en `apps/orchestrator/src/signals`).
5. **Video generado**: `generateVideoWorkflow` (módulo `videos/`).
6. **Racha**: `streakWorkflow`.
7. **Posts perdidos**: `missingPostWorkflow` detecta publicaciones no ejecutadas; se registra al arrancar solo con `RUN_CRON`.
8. **API pública y agentes**: `/public/v1` (subida de medios, posts, integraciones, analíticas), servidor MCP (`chat/start.mcp.ts`), Copilot (`/copilot`) y comando `run:agent`.
9. **Facturación**: Stripe (`/stripe`, `/billing`, `/payment`) y RevenueCat.

### Workflows de Temporal (`apps/orchestrator/src/workflows`)

| Workflow | Archivo |
|---|---|
| Publicación de posts (versionado v1.0.1 a v1.1.2) | `post-workflows/post.workflow.v1.x.x.ts` |
| `autoPostWorkflow` | `autopost.workflow.ts` |
| `digestEmailWorkflow` | `digest.email.workflow.ts` |
| `sendEmailWorkflow` | `send.email.workflow.ts` |
| `generateVideoWorkflow` | `generate.video.workflow.ts` |
| `missingPostWorkflow` | `missing.post.workflow.ts` |
| `refreshTokenWorkflow` | `refresh.token.workflow.ts` |
| `streakWorkflow` | `streak.workflow.ts` |

Activities: `post`, `autopost`, `email`, `integrations`, `video` (`apps/orchestrator/src/activities`).

El workflow más reciente (v1.1.2) usa `startToCloseTimeout` de 10 minutos, hasta 3 intentos con intervalo inicial de 2 minutos, y reintenta ante timeouts de heartbeat (3 minutos) cuando no hay detalles de heartbeat. Los heartbeats se envían cada 15 s.

Reglas de evolución (de `CLAUDE.md`): crear una nueva versión de workflow en vez de editar uno ya fusionado, y no cambiar parámetros de activities existentes.

## CI/CD (`.github/workflows`)

| Archivo | Disparador | Qué hace |
|---|---|---|
| `build.yml` | `push`, `pull_request`, `merge_group` | Node 22.12.0, pnpm 10, `pnpm install` y `pnpm run build`. |
| `build-containers.yml` | `workflow_dispatch` y push de tags | Construye `Dockerfile.dev` para amd64 y arm64 y publica en `ghcr.io/gitroomhq/postiz-app` con manifest multi-arquitectura. **Apunta al repositorio del upstream**; en este fork ese flujo no se usa para el despliegue (ver Coolify). |
| `codeql.yml` | push a `main` en `apps/**` y `libraries/**` | Análisis CodeQL. |
| `staging-conflicts.yml` | cron cada 10 min y manual | Intenta fusionar `main` en `staging` y resuelve conflictos con Claude; requiere los secretos `STAGING_MERGE_KEY` y `CLAUDE_CODE_OAUTH_TOKEN`. |
| `build-extension.yaml`, `publish-extension.yml` | Por confirmar disparador | Compilación y publicación de la extensión. |
| `issue-label-triggers.yml`, `stale.yml` | Eventos de issues | Gestión de issues. |
| `eslint` | (archivo sin extensión `.yml` en la carpeta) | Por confirmar si GitHub lo ejecuta. |

Otros: `Jenkins/Build.Jenkinsfile` y `Jenkins/BuildPR.Jenkinsfile` (heredados del upstream; uso en este fork **Por confirmar**), `railway.toml` (configuración de Nixpacks para Railway), `sonar-project.properties`, `.coderabbit.yaml`.

Ramas remotas observadas: `main` (base de este PR), `staging` y ramas de fix/feat ya integradas (`feat/coolify-deploy`, `fix/coolify-http-login-healthcheck`, `fix/coolify-temporal-healthchecks`, `fix/ci-lint-workflow-and-noisy-crons`). No existe rama `production`.

## Docker

`Dockerfile.dev` (usado también en producción en este fork):

1. Base `node:22.20-bookworm-slim`, instala `g++`, `make`, `python3-pip`, `bash`, `nginx`.
2. Crea el usuario `www`, instala globalmente `pnpm@10.6.1` y `pm2`.
3. Copia el repo a `/app` y `var/docker/nginx.conf` a `/etc/nginx/nginx.conf`.
4. `pnpm install` y `pnpm run build` (con `--max-old-space-size=4096`).
5. `CMD`: `nginx && pnpm run pm2`, es decir `prisma db push` y luego pm2 arranca `backend`, `frontend` y `orchestrator` en paralelo.

`var/docker/nginx.conf` (puerto 5000): `/api/` va al backend `:3000`, `/uploads/` sirve el volumen de subidas con cabeceras restrictivas (CSP, `nosniff`) y solo tipos de imagen y mp4, y `/` va al frontend `:4200`. Límite de cuerpo 2 GB.

`docker-compose.yaml` es el ejemplo del upstream (imagen oficial, puerto 4007). `docker-compose.dev.yaml` es solo para desarrollo local (Postgres, Redis, pgAdmin, RedisInsight y Temporal con sus redes).

## Despliegue en Coolify

Archivo: `docker-compose.coolify.yaml` (build pack "Docker Compose").

Configuración en Coolify (según los comentarios del archivo):

- Docker Compose Location: `/docker-compose.coolify.yaml`.
- Dominio del servicio `postiz`: `https://<dominio>:5000` (el 5000 es el puerto interno de nginx, no un puerto público).
- Variables de entorno: `POSTIZ_URL` (obligatoria, `https://<dominio>`). Las `SERVICE_PASSWORD_*` (`SERVICE_PASSWORD_64_JWT`, `SERVICE_PASSWORD_POSTGRES`, `SERVICE_PASSWORD_TEMPORAL`) las genera Coolify.
- Si se sirve por HTTP plano (por ejemplo un dominio sslip.io temporal), definir `NOT_SECURED=true`; de lo contrario el navegador descarta la cookie de login. Quitarla cuando haya HTTPS.
- Las claves de redes sociales, almacenamiento, correo y OpenAI son opcionales (por defecto vacías) y se rellenan en la pantalla de variables de Coolify, seguido de redeploy.
- No se publican puertos del host; solo el proxy de Coolify llega a `postiz:5000`.

Servicios del stack:

| Servicio | Imagen | Notas |
|---|---|---|
| `postiz` | Build local de `Dockerfile.dev` | nginx + backend + frontend + orquestador. Volúmenes `postiz-config` (`/config/`) y `postiz-uploads` (`/uploads/`). Healthcheck con `node -e fetch` contra `/api/` y `/auth` por el puerto 5000 (no hay curl/wget en la imagen), `start_period` 300 s. |
| `postiz-postgres` | `postgres:17-alpine` | BD de la aplicación (usuario y BD `postiz`). |
| `postiz-redis` | `redis:7.2` | |
| `temporal-postgresql` | `postgres:16` | BD de Temporal. |
| `temporal-elasticsearch` | `elasticsearch:7.17.27` | Visibilidad avanzada; heap de 256 MB, single-node, sin seguridad xpack. |
| `temporal` | `temporalio/auto-setup:1.28.1` | Namespace `default`; healthcheck con `temporal operator cluster health`. |
| `temporal-ui` | `temporalio/ui:2.34.0` | Puerto interno 8080, sin publicar. |

`postiz` depende (con `service_healthy`) de `postiz-postgres`, `postiz-redis` y `temporal`. Conexiones internas: `DATABASE_URL` a `postiz-postgres:5432`, `REDIS_URL` a `postiz-redis:6379`, `TEMPORAL_ADDRESS=temporal:7233`, `BACKEND_INTERNAL_URL=http://localhost:3000`.

Orden de arranque: PostgreSQL de Temporal y Elasticsearch, luego Temporal, luego Temporal UI y la aplicación (esta última también espera BD y Redis).

### Puntos a revisar en el despliegue (Por confirmar)

- `RUN_CRON` no está definido en el compose de Coolify, así que `missingPostWorkflow` no se registra automáticamente. Confirmar si es intencional.
- `temporal-ui` no tiene dominio ni puerto publicado: para usarla habría que asignarle un dominio en Coolify o acceder por túnel.
- `NEXT_PUBLIC_*` se resuelven en build y en runtime; verificar que cambiar `POSTIZ_URL` no requiera reconstruir la imagen.
- Variables soportadas por el código pero no pasadas por el compose (por ejemplo `STRIPE_*`, `APPLE_*`, `TIKTOK_BUSINESS_*`, `DUB_*`, `RUN_CRON`, `EXCLUDE_QUEUE`, `WORKER_CONCURRENCY_DIVIDER`, `TEMPORAL_NAMESPACE`) no llegarán al contenedor a menos que se añadan al compose.
- La base de datos se sincroniza con `prisma db push --accept-data-loss` en cada arranque (`pm2-run`); no hay migraciones versionadas en este flujo. Evaluar el riesgo en producción.
- Estrategia de respaldo de los volúmenes (`postiz-postgres-data`, `temporal-postgres-data`, `postiz-uploads`): no está documentada en el repo.
