# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

TutoriaLLM is a self-hosted, AI-powered visual programming (Blockly) learning platform for K-12 education. Users learn by dragging/dropping blocks; an LLM tutor provides real-time guidance via Socket.IO. Admins manage tutorials, users, and training data (with vector-embedding-based knowledge retrieval).

## Monorepo Structure

This is a **pnpm workspace** monorepo managed with `wireit` for task orchestration.

- `apps/backend` — Hono + Node.js API server (port 3001)
- `apps/frontend` — React + Vite SPA (port 3000)
- `apps/docs` — Astro/Starlight documentation site
- `packages/extensions` — Shared Blockly extension blocks (Minecraft-Core, Server); built first as a dependency for both apps

## Development Commands

```bash
# Install dependencies (pnpm only — enforced via preinstall hook)
pnpm install

# Run both frontend and backend in dev mode (wireit orchestration)
pnpm dev

# Run tests (vitest, all workspaces)
pnpm test

# Run a single test file
pnpm vitest run apps/backend/src/modules/session/index.test.ts

# Type-check all apps
pnpm type:check

# Lint and format (Biome)
pnpm check       # lint + fix
pnpm format      # format only

# Spell-check
pnpm spell-check

# Commits must use commitizen (conventional commits enforced by commitlint)
pnpm commit
```

### Backend-specific

```bash
cd apps/backend

# DB migrations (Drizzle ORM)
pnpm drizzle:generate
pnpm drizzle:migrate

# Generate better-auth schema
pnpm auth:generate

# Drizzle Studio (local DB browser)
pnpm drizzle:studio

# CLI tool
pnpm cli
```

### Infrastructure

Docker Compose provides PostgreSQL (pgvector/pgvector:pg14) and MinIO (S3-compatible storage). The backend `.env` file configures DB credentials, S3, OpenAI keys, and CORS origin.

```bash
docker compose up          # production-like
docker compose -f docker-compose.yml -f docker-compose.dev.override.yml up  # dev with hot reload
```

## Architecture

### Backend (`apps/backend/src`)

- **Framework**: Hono with `@hono/zod-openapi` — all routes are typed and generate OpenAPI specs
- **Auth**: `better-auth` with drizzle adapter; plugins: `admin`, `username`, `anonymous`. Auth routes at `/auth/*`
- **Database**: PostgreSQL via Drizzle ORM. Schema in `src/db/schema/` (session, tutorial, training, auth tables). Uses pgvector for embedding-based knowledge search
- **Context**: `src/context.ts` defines the Hono `Context` type with `db`, `user`, `session` variables injected by `src/middleware/inject.ts`
- **Modules**: Each module (`session`, `tutorials`, `admin`, `config`, `health`, `image`, `vm`, `vmProxy`) follows the pattern: `index.ts` (controller) + `routes.ts` (OpenAPI route defs) + `schema.ts` (Zod schemas)
- **Socket.IO**: Real-time session sync at `/session/socket/connect`. The socket server (`src/modules/session/socket/`) handles workspace diffs (RFC 6902 JSON Patch), LLM calls, and VM lifecycle events
- **VM**: User code runs in Node.js `worker_threads` via `vm` sandbox (`src/modules/vm/tsWorker.ts`). Each session gets an isolated Hono micro-server proxied via `vmProxy`. Blockly workspace JSON is converted to JavaScript by `src/libs/blocklyCodeGenerator.ts`
- **LLM**: Uses Vercel AI SDK (`ai` package) with OpenAI provider via `src/libs/openai.ts`. Configurable `OPENAI_API_ENDPOINT` for custom endpoints. Knowledge retrieval uses cosine similarity on pgvector embeddings (`text-embedding-3-small`)
- **Type-safe client**: The backend exports `AppType` from `src/index.ts` and `hcWithType`/`adminHcWithType` from `src/hc.ts` for end-to-end typed API calls in the frontend

### Frontend (`apps/frontend/src`)

- **Router**: TanStack Router with file-based routing (`src/routes/`). Route tree auto-generated to `routeTree.gen.ts`
- **Data fetching**: TanStack Query. The typed Hono client (`src/api/index.ts`) wraps `hcWithType` from `backend/hc`
- **State**: Jotai atoms in `src/state.ts`; IndexedDB for offline/local persistence via `src/indexedDB.tsx`
- **Real-time**: Socket.IO client (`src/libs/socket.ts`). Session state is synced using RFC 6902 JSON Patch diffs (`rfc6902`)
- **UI**: Tailwind CSS + Radix UI primitives + shadcn-style components in `src/components/ui/`. Feature components in `src/components/features/editor/` (Blockly editor, dialogue, session overlay) and `src/components/features/admin/`
- **i18n**: react-i18next; config in `src/i18n/`
- **Auth**: `better-auth` browser client in `src/libs/auth-client.ts`

### Extensions Package (`packages/extensions/src`)

Defines reusable Blockly block sets (`Minecraft-Core`, `Server`). Each extension exports `Blocks` and `Toolbox`. The `Block` type (`src/types/block.ts`) is the interface all custom blocks must satisfy. This package is built before both apps.

## Key Conventions

- **Linter/Formatter**: Biome (not ESLint/Prettier). Config in `biome.json`. `console.log` is warned; use `console.info` / `console.error` / `console.warn` instead.
- **Pre-commit hooks** (via `simple-git-hooks`): auto-runs `format`, `lint:repo` (sherif), `check`, and `type:check` on commit.
- **Path aliases**: Both apps use `@/` mapped to their respective `src/` directory via tsconfig paths.
- **Catalog versions**: Shared dependency versions are pinned in `pnpm-workspace.yaml` under `catalog:` and referenced as `"catalog:"` in package.json files.
- **Tests**: Vitest. Backend integration tests use `testcontainers` for real Postgres. Test helper at `tests/test.helper` sets up DB and auth. Playwright e2e tests are excluded from `vitest run`.
