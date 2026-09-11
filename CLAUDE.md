# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Postiz is a tool to schedule social media and chat posts to 28+ channels. It also covers:
- Schedule posts
- Calendar view
- Analytics
- Team management
- Media library

A post is added to the calendar, gets picked up into a Temporal workflow, and is published at the scheduled time.

## Monorepo layout

PNPM workspaces, root-only `package.json` for dependencies (no per-app lockfiles). Workspace packages: `apps/*` and `libraries/*` (see `pnpm-workspace.yaml`).

- `apps/backend` — the API (NestJS). Mostly controllers; business logic is imported from `libraries/nestjs-libraries`.
- `apps/orchestrator` — Temporal (NestJS). Contains all workflows and activities for background jobs (e.g. publishing a scheduled post).
- `apps/frontend` — the web app. **Next.js (App Router)**, not Vite — routing lives under `apps/frontend/src/app`, components under `apps/frontend/src/components`.
- `apps/extension` — browser extension for cookie-based platform auth (Vite + React + CRXJS).
- `apps/commands` — CLI-style NestJS commands (`nestjs-command`).
- `apps/sdk` — published `@postiz/node` SDK for the public API.
- `libraries/nestjs-libraries` — shared backend code: Prisma database layer, DTOs, provider integrations, agent/chat, email, payments, uploads, etc. Used by both `backend` and `orchestrator`.
- `libraries/helpers` — cross-cutting utilities shared by backend, orchestrator, and frontend (auth, config, decorators, the `useFetch` fetch hook, etc.).
- `libraries/react-shared-libraries` — shared React code (forms, toaster, i18n/translation, Sentry) used by `frontend` and `extension`.

We are using only pnpm — never use npm/yarn, and never install frontend components from npmjs; write native components instead.

## Commands

Run from the repo root unless noted.

```bash
pnpm install                 # installs deps and runs prisma-generate via postinstall
pnpm run dev                 # runs extension + orchestrator + backend + frontend in parallel
pnpm run dev-backend         # backend + frontend only
pnpm run dev:backend         # backend only (also clears apps/backend/dist first)
pnpm run dev:frontend        # frontend only (port 4200)
pnpm run dev:orchestrator    # orchestrator only
pnpm run build               # builds frontend, backend, orchestrator (workspace-concurrency=1)
pnpm run build:backend
pnpm run build:frontend
pnpm run build:orchestrator
pnpm test                    # jest --coverage --detectOpenHandles, project-wide
npx eslint .                 # lint — must be run from the repo root, there is no per-app lint script
```

Prisma (schema at `libraries/nestjs-libraries/src/database/prisma/schema.prisma`):

```bash
pnpm run prisma-generate     # regenerate the Prisma client
pnpm run prisma-db-push      # push schema changes to the dev DB (--accept-data-loss)
pnpm run prisma-db-pull      # pull DB schema back into schema.prisma
pnpm run prisma-reset        # force-reset the dev DB and push schema
```

Local infra: `pnpm run dev:docker` (`docker-compose.dev.yaml` — Postgres/Redis for local dev).

There is currently no unit test suite in the repo (no `*.spec.ts` files exist yet) even though `pnpm test` / jest / jest-junit are wired up — don't assume prior test coverage exists for code you're touching.

To run a single test file once tests exist: `pnpm exec jest path/to/file.spec.ts`.

## Backend architecture

Backend requests must pass through all layers, no shortcuts:

```
DTO >> Controller >> Service >> Repository
```

or, when a manager coordinates multiple services:

```
DTO >> Controller >> Manager >> Service >> Repository
```

- Controllers live in `apps/backend/src/api/routes` (and `public-api/routes` for the public API). They should stay thin — parse/validate via DTOs and delegate.
- Services/repositories/DTOs live in `libraries/nestjs-libraries/src` (e.g. `database/prisma/posts/posts.service.ts` + `posts.repository.ts`, `dtos/posts/...`). Each Prisma-backed feature typically gets its own folder under `database/prisma/<feature>` with a `.repository.ts` and `.service.ts`.
- Never use raw SQL — always go through Prisma.
- The system is in production with many existing users. Any schema or behavior change needs to consider backward compatibility and whether a migration is required.

### Social/integration providers

Per-platform logic (Facebook, Instagram, LinkedIn, etc.) lives under `libraries/nestjs-libraries/src/integrations/social/*.provider.ts`, one file per platform, all implementing the shared `social.integrations.interface.ts`. Generic code must never branch on a specific provider (no `if (facebookProvider) {}` inside generic code) — instead extend the provider interface with a new method, call it generically, and implement the platform-specific behavior only inside that platform's provider file. The same pattern applies to other pluggable integrations (`newsletter/providers`, `short-linking/providers`, `services/payment/providers`).

## Orchestrator (Temporal) architecture

`apps/orchestrator/src/workflows` and `.../activities` define all background jobs (post publishing, autopost, email, video, integrations).

Critical constraints, because Temporal replays workflow history against the *current* code:
- **Never edit a workflow file that's already merged to `origin/main`** — changing it breaks replay for any workflow already in flight. Instead, add a new versioned workflow file (see `workflows/post-workflows/post.workflow.v1.x.x.ts` for the existing pattern) and update every call site to point at the new version.
- **Never change an existing activity's parameters** — this breaks in-flight workflows calling it. Instead, add a new activity with the new signature and use it from a new workflow version.

## Frontend conventions

- UI primitives live in `apps/frontend/src/components/ui`; check there (and similar existing components) before building new ones.
- Routing is in `apps/frontend/src/app` (Next.js App Router); feature components in `apps/frontend/src/components`.
- Before writing/styling a component, check `apps/frontend/src/app/colors.scss`, `apps/frontend/src/app/global.scss`, and `apps/frontend/tailwind.config.js` (Tailwind 3). Every `--color-custom*` variable is deprecated — don't use it.
- Data fetching always goes through SWR, using the `useFetch` hook from `libraries/helpers/src/utils/custom.fetch.tsx`.
- Each SWR call must live in its own hook, and hooks must follow `react-hooks/rules-of-hooks` — never suppress with `eslint-disable-next-line`. Valid:
  ```ts
  const useCommunity = () => {
    return useSWR(...);
  };
  ```
  Invalid (conditional/lazy hook calls hidden behind another function):
  ```ts
  const useCommunity = () => {
    return {
      communities: () => useSWR<CommunitiesListResponse>("communities", getCommunities),
      providers: () => useSWR<ProvidersListResponse>("providers", getProviders),
    };
  };
  ```

## General code guidance

- Avoid creating new files that are pure algorithmic logic in isolation — it's usually the wrong abstraction; prefer integrating with existing services/patterns.
- Match existing patterns: before adding code, find a similar existing example in the codebase and follow its shape rather than inventing a new one.
- After finishing a change, it's worth a second pass (e.g. a fresh agent) specifically comparing the new code against existing equivalents in the system to confirm it isn't a stylistic outlier.

## Pull requests

Always follow `.github/PULL_REQUEST_TEMPLATE.md`. Two sections are non-negotiable in every PR description, including one-line fixes:

- **`# What kind of change does this PR introduce?`** — must state the type, the area touched (backend/frontend/orchestrator/a specific provider or screen), and 1–3 concrete sentences on what changed and where (key function/endpoint/file/field), plus what deliberately stayed the same. Just "Bug fix." is not acceptable.
- **`# QA`** — real, numbered steps (setup, action, expected result) a reviewer can run without asking the author anything, written as `1. [ ] step` checkboxes on plain lines (not inside a fenced code block — those are ignored by the extractor). `# Testing`, `# Test plan`, `# How to test`, `# How to verify`, `# Verification`, `# Steps to test`, and `# Manual testing` are also recognized headings, but prefer `# QA`. Never leave the template placeholder, and never write `N/A`/`TBD`/`todo`/`none`/an empty checkbox as the whole section — those all count as missing QA.
