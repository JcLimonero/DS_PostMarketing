# Documentación técnica — DS_PostMarketing

DS_PostMarketing es un **fork propio de [Postiz](https://github.com/gitroomhq/postiz-app)** (herramienta para programar publicaciones en redes sociales y canales de chat). El remoto `origin` apunta a `JcLimonero/DS_PostMarketing` y el upstream original es `gitroomhq/postiz-app`. La versión base declarada en `version.txt` es `v1.47.0`.

Lo propio de este fork (commits posteriores al upstream) es principalmente:

- Renombrado del repo y nota en el `README.md` raíz.
- `docker-compose.coolify.yaml`: stack de producción para **Coolify** que construye la imagen desde este repositorio (`Dockerfile.dev`) en lugar de bajar la imagen oficial de GHCR.
- Healthchecks del servicio `postiz` y de los servicios de Temporal, y soporte de login sobre HTTP plano (`NOT_SECURED`) para dominios temporales.
- `.claude/launch.json` con configuraciones de arranque local.

El resto del código es el de Postiz upstream; no se ha verificado qué tanto se ha modificado fuera de los puntos anteriores (**Por confirmar**).

## Índice

| Documento | Contenido |
|---|---|
| [arquitectura.md](arquitectura.md) | Componentes, capas, dependencias externas, diagramas Mermaid |
| [workflows.md](workflows.md) | Flujos de negocio, workflows de Temporal, CI/CD, despliegue en Coolify |
| [codigo.md](codigo.md) | Estructura de carpetas, módulos, endpoints, modelos de datos, variables de entorno |

## Stack

- **Monorepo pnpm** (`pnpm@10.6.1`, Node `>=22.12.0 <23`), workspaces `apps/*` y `libraries/*`.
- **Backend**: NestJS 11 (`apps/backend`), Prisma 6.5 sobre PostgreSQL, Redis (throttling y otros usos).
- **Orquestación**: Temporal (`apps/orchestrator`, SDK `@temporalio/*` vía `nestjs-temporal-core`).
- **Frontend**: Next.js App Router, React, Tailwind 3 (`apps/frontend`).
- **Extensión de navegador**: Vite + React (`apps/extension`).
- **IA / agentes**: Mastra, CopilotKit, LangChain/LangGraph, servidor MCP, OpenAI.
- **Observabilidad**: Sentry.

## Cómo correrlo localmente

Según `CLAUDE.md`, `package.json` y `docker-compose.dev.yaml`:

1. Requisitos: Node 22.x, pnpm 10.6.1, Docker.
2. Copiar `.env.example` a `.env` en la raíz y completar al menos `DATABASE_URL`, `REDIS_URL`, `JWT_SECRET`, `FRONTEND_URL`, `NEXT_PUBLIC_BACKEND_URL`, `BACKEND_INTERNAL_URL`, `STORAGE_PROVIDER` (ver [codigo.md](codigo.md#variables-de-entorno)).
3. Levantar infraestructura local: `pnpm run dev:docker` (Postgres, Redis, pgAdmin, RedisInsight y el stack de Temporal definidos en `docker-compose.dev.yaml`).
4. `pnpm install` (el `postinstall` ejecuta `prisma generate`).
5. `pnpm run prisma-db-push` para crear el esquema en la base local.
6. `pnpm run dev` (extensión + orquestador + backend + frontend en paralelo) o `pnpm run dev-backend` (solo backend + frontend). El frontend corre en el puerto `4200` y el backend en `3000`.

Para probar con la imagen completa (nginx + los tres procesos con pm2), ver la sección Docker de [workflows.md](workflows.md).

> El repositorio no tiene pruebas unitarias (`*.spec.ts`) según `CLAUDE.md`, aunque `pnpm test` (jest) está configurado.
