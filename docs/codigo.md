# Código: estructura, APIs, modelos y configuración

## Estructura de carpetas

```
.
├── apps/
│   ├── backend/        API NestJS (src/api/routes, src/public-api, src/services/auth)
│   ├── orchestrator/   Workers de Temporal (src/workflows, src/activities, src/signals)
│   ├── frontend/       Next.js App Router (src/app, src/components)
│   ├── extension/      Extensión de Chrome (Vite + CRXJS), proveedores por cookie (Skool)
│   ├── commands/       Comandos CLI NestJS (run:agent, config:check, refresh)
│   └── sdk/            Paquete @postiz/node
├── libraries/
│   ├── nestjs-libraries/src/
│   │   ├── database/prisma/   schema.prisma + repositorios/servicios por feature
│   │   ├── integrations/      social/*.provider.ts, integration.manager
│   │   ├── dtos/              DTOs de validación
│   │   ├── temporal/          módulo Temporal, atributos de búsqueda, heartbeat
│   │   ├── chat/ y agent/     Mastra, MCP, herramientas de agentes
│   │   ├── upload/            almacenamiento local y Cloudflare R2
│   │   ├── emails/, newsletter/, short-linking/, services/payment/, 3rdparties/, videos/
│   │   ├── redis/, throttler/, sentry/, track/, openai/, user/
│   ├── helpers/               utilidades compartidas, configuration.checker, swagger, useFetch
│   └── react-shared-libraries/  formularios, toaster, i18n, Sentry
├── var/docker/        nginx.conf y scripts de Docker
├── dynamicconfig/     configuración de Temporal para desarrollo
├── Jenkins/           Jenkinsfiles heredados
├── .github/workflows/ CI
├── Dockerfile.dev, docker-compose*.yaml
└── .claude/           launch.json y settings de Claude Code
```

Se ignoran en esta documentación `node_modules`, `dist`, builds y `reports/junit.xml` (generado).

## Puntos de entrada

| App | Entrada | Puerto / comando |
|---|---|---|
| Backend | `apps/backend/src/main.ts` | `PORT` (3000). Habilita CORS con `FRONTEND_URL` y `MAIN_URL`, `ValidationPipe`, JSON de 50 MB en `/posts` y `/copilot`, Swagger y MCP. Al final ejecuta el `ConfigurationChecker`. |
| Orquestador | `apps/orchestrator/src/main.ts` | `ORCHESTRATOR_PORT` (3002), solo health check. |
| Frontend | `next dev/start -p 4200` | 4200 |
| Commands | `apps/commands/src/main.ts` | `run:agent`, `config:check`, `refresh` |

Scripts pm2 (usados por Docker): `pnpm run pm2` ejecuta `prisma-db-push` y luego arranca backend, frontend y orquestador.

## APIs del backend

Los controladores están en `apps/backend/src/api/routes`, registrados en `api.module.ts`. Prefijos de ruta (según `@Controller`):

| Prefijo | Controlador | Notas |
|---|---|---|
| `/auth` | auth | Login, registro, proveedores de login. Cookies seguras salvo `NOT_SECURED`. |
| `/user` | users | Perfil, organizaciones. |
| `/user/oauth-app`, `/user/approved-apps` | oauth-app, approved-apps | Aplicaciones OAuth propias y autorizadas. |
| `/oauth` | oauth | Servidor OAuth (usado por MCP/ChatGPT). |
| `/integrations` | integrations, no.auth.integrations | Canales conectados; flujo de conexión social. |
| `/posts` | posts | CRUD y programación de posts. |
| `/media` | media | Biblioteca de medios y subidas. |
| `/analytics` | analytics | Analíticas por canal. |
| `/autopost` | autopost | Reglas de autopost. |
| `/sets`, `/signatures` | sets, signature | Plantillas de configuración y firmas. |
| `/settings` | settings | Ajustes de la organización (equipo, webhooks, etc.). |
| `/webhooks` | webhooks | Webhooks salientes. |
| `/third-party` | third-party | Integraciones como HeyGen. |
| `/notifications`, `/announcements` | notifications, announcements | |
| `/billing`, `/payment`, `/stripe` | billing, payment, stripe | Suscripciones y webhooks de pago. |
| `/copilot` | copilot | Runtime de CopilotKit / agentes. |
| `/enterprise`, `/admin` | enterprise, admin | Funciones de administración. |
| `/monitor` | monitor | Chequeos de salud. |
| `/public` | public | Rutas públicas (p. ej. vista previa). |
| `/` | root | Ruta raíz (el healthcheck del compose consulta `/api/` vía nginx). |

### API pública (`/public/v1`)

`apps/backend/src/public-api/routes/v1/public.integrations.controller.ts`, autenticada con `PublicAuthMiddleware` (clave de API de la organización). Endpoints: `POST /upload`, `POST /upload-from-url`, `GET /find-slot/:id`, `GET|POST /posts`, `DELETE /posts/:id`, `DELETE /posts/group/:group`, `GET /is-connected`, `GET /groups`, `GET /integrations`, `GET /social/:integration`, `GET /users`, `GET /notifications`, `POST /generate-video`, `POST /video/function`, `DELETE /integrations/:id`, `GET /integration-settings/:id`, `GET /posts/:id/missing`, `PUT /posts/:id/settings|status|release-id`, `GET /analytics/:integration`, `GET /analytics/post/:postId`, `POST /integration-trigger/:id`. El SDK `@postiz/node` (`apps/sdk`) la consume.

En nginx, `/api/` se reescribe hacia el backend (el prefijo `/api` se elimina), por lo que la URL pública es `https://<dominio>/api/<ruta>`.

## Frontend

Rutas en `apps/frontend/src/app/(app)`: `auth`, `integrations`, `oauth`, `api`, y dentro de `(site)`: `launches` (calendario), `analytics`, `media`, `agents`, `plugs`, `billing`, `settings`, `third-party`, `admin`, `err`; además `(preview)/p/[id]`. También hay grupos `(extension)` y `(provider)`.

## Modelos de datos

Esquema Prisma: `libraries/nestjs-libraries/src/database/prisma/schema.prisma` (PostgreSQL). Modelos de negocio principales:

- **Organización y usuarios**: `Organization`, `User`, `UserOrganization` (rol), `Subscription`, `Customer`, `Credits`, `UsedCodes`.
- **Canales**: `Integration` (canal social conectado, tokens), `IntegrationsWebhooks`, `Webhooks`, `Plugs`, `ExisingPlugData`, `ThirdParty`, `Mentions`.
- **Contenido**: `Post`, `Comments`, `Tags`, `TagsPosts`, `Media`, `Signatures`, `Sets`, `AutoPost`, `PopularPosts`.
- **Comunicación**: `Notifications`, `Announcement`, `Errors`, `MessagesGroup`, `Messages`.
- **Marketplace/pagos**: `Orders`, `OrderItems`, `PayoutProblems`, `SocialMediaAgency`, `SocialMediaAgencyNiche`.
- **OAuth/MCP**: `OAuthApp`, `OAuthAuthorization`.
- **Otros heredados**: `GitHub`, `Trending`, `TrendingLog`, `ItemUser`, `Star`.
- **Mastra (agentes)**: `mastra_ai_spans`, `mastra_evals`, `mastra_messages`, `mastra_resources`, `mastra_scorers`, `mastra_threads`, `mastra_traces`, `mastra_workflow_snapshot`.
- Enums: `OrderStatus`, `From`, `State`, `SubscriptionTier`, `Period`, `Provider`, `Role`, `APPROVED_SUBMIT_FOR_ORDER`, `CreationMethod`, `ShortLinkPreference`, `AnnouncementColor`.

No hay carpeta de migraciones versionadas en el repo; el esquema se aplica con `prisma db push` (**Por confirmar** cómo se gestionan los cambios de esquema en producción).

## Variables de entorno

Solo se listan nombres y propósito. Fuentes: `.env.example`, `docker-compose.yaml`, `docker-compose.coolify.yaml` y el código. **No se incluyen valores.**

### Obligatorias (validadas por `ConfigurationChecker`)

| Variable | Propósito |
|---|---|
| `DATABASE_URL` | Cadena de conexión a PostgreSQL. |
| `REDIS_URL` | Conexión a Redis. |
| `JWT_SECRET` | Firma de sesiones (en Coolify: `SERVICE_PASSWORD_64_JWT`). |
| `MAIN_URL`, `FRONTEND_URL` | URL pública de la aplicación. |
| `NEXT_PUBLIC_BACKEND_URL` | URL pública del backend (`<url>/api`). |
| `BACKEND_INTERNAL_URL` | URL interna del backend para el frontend (`http://localhost:3000`). |
| `STORAGE_PROVIDER` | `local` o `cloudflare`. |

### Temporal y ejecución

| Variable | Propósito |
|---|---|
| `TEMPORAL_ADDRESS` | Dirección del servidor Temporal (`temporal:7233`). |
| `TEMPORAL_NAMESPACE` | Namespace (por defecto `default`). |
| `TEMPORAL_TLS`, `TEMPORAL_API_KEY` | TLS y clave para Temporal Cloud. |
| `EXCLUDE_QUEUE` | Colas de proveedor que un worker no debe ejecutar. |
| `WORKER_CONCURRENCY_DIVIDER` | Divide la concurrencia por proveedor entre N servidores. |
| `RUN_CRON` | Si está definida, registra `missingPostWorkflow`. |
| `ORCHESTRATOR_PORT`, `PORT` | Puertos del orquestador y backend. |
| `IS_GENERAL`, `API_LIMIT`, `DISABLE_REGISTRATION`, `NOT_SECURED` | Modo general, límite de peticiones, registro cerrado y cookies no seguras (cualquier valor no vacío las desactiva). |
| `NX_ADD_PLUGINS` | Ajuste de desarrollo. |

### Almacenamiento, correo y misceláneos

- Almacenamiento: `UPLOAD_DIRECTORY`, `NEXT_PUBLIC_UPLOAD_DIRECTORY`, `CLOUDFLARE_ACCOUNT_ID`, `CLOUDFLARE_ACCESS_KEY`, `CLOUDFLARE_SECRET_ACCESS_KEY`, `CLOUDFLARE_BUCKETNAME`, `CLOUDFLARE_BUCKET_URL`, `CLOUDFLARE_REGION`.
- Correo: `EMAIL_PROVIDER`, `RESEND_API_KEY`, `EMAIL_FROM_ADDRESS`, `EMAIL_FROM_NAME`.
- IA y UI: `OPENAI_API_KEY`, `OPENAI_OAUTH_CLIENT_ID`, `NEXT_PUBLIC_POLOTNO`, `NEXT_PUBLIC_DISCORD_SUPPORT`, `EVOLINK_API_KEY`.
- Seguridad: `DISABLE_SSRF_PROTECTION`, `DCR_VERIFIED_DOMAINS`.
- Extensión: `EXTENSION_ID`.
- Proveedores ocultos o migrados: `HIDDEN_PROVIDERS`, `MIGRATE_PROVIDERS`.

### Credenciales de redes sociales y login

Pares `*_CLIENT_ID` / `*_CLIENT_SECRET` (o equivalente) para: X (`X_URL`, `X_API_KEY`, `X_API_SECRET`), LinkedIn, Reddit, GitHub, YouTube, TikTok, `TIKTOK_BUSINESS_*`, Pinterest, Dribbble, Tumblr, Discord (`DISCORD_BOT_TOKEN_ID`), Mastodon (`MASTODON_URL`), Threads (`THREADS_APP_ID`/`SECRET`), Facebook (`FACEBOOK_APP_ID`/`SECRET`), Slack (`SLACK_ID`, `SLACK_SECRET`, `SLACK_SIGNING_SECRET`), Beehiiv (`BEEHIIVE_*`), Listmonk (`LISTMONK_*`) y Apple (`APPLE_*`). OAuth genérico: `POSTIZ_GENERIC_OAUTH`, `POSTIZ_OAUTH_*`, `NEXT_PUBLIC_POSTIZ_OAUTH_DISPLAY_NAME`, `NEXT_PUBLIC_POSTIZ_OAUTH_LOGO_URL`.

### Pagos y acortadores

- Stripe/RevenueCat: `FEE_AMOUNT`, `STRIPE_PUBLISHABLE_KEY`, `STRIPE_SECRET_KEY`, `STRIPE_SIGNING_KEY`, `STRIPE_SIGNING_KEY_CONNECT`, `REVENUECAT_SECRET_KEY`, `REVENUECAT_WEBHOOK_SECRET`, `IN_APP_PURCHASE_REJECT_SANDBOX`.
- Acortadores: `DUB_*`, `SHORT_IO_SECRET_KEY`, `KUTT_*`, `LINK_DRIP_*`.

### Específicas del despliegue en Coolify

`POSTIZ_URL` (obligatoria), `SERVICE_PASSWORD_64_JWT`, `SERVICE_PASSWORD_POSTGRES` y `SERVICE_PASSWORD_TEMPORAL` (generadas por Coolify). Las credenciales de redes sociales se pasan con valor vacío por defecto. Ver [workflows.md](workflows.md#despliegue-en-coolify).

## Comandos útiles

```bash
pnpm run prisma-generate   # regenerar cliente Prisma
pnpm run prisma-db-push    # aplicar esquema (usa --accept-data-loss)
pnpm run build             # frontend, backend y orquestador
npx eslint .               # lint, desde la raíz
```

## Por confirmar

- Lista completa y estado (activos/obsoletos) de las variables `*_API_KEY` de terceros realmente usadas.
- Cobertura de pruebas: no hay `*.spec.ts` según `CLAUDE.md`.
- Estrategia de migraciones de BD en producción.
