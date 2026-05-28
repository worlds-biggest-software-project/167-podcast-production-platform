# Podcast Production Platform — Phased Development Plan

> Project: 167-podcast-production-platform · Created: 2026-05-25
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary language | TypeScript (Node.js 22 LTS) | Full-stack coverage (API + frontend) in one language; strong ecosystem for audio processing orchestration, WebSocket-based real-time features, and LLM SDK integration; first-class OpenAPI tooling |
| API framework | Fastify 5 | Fastest Node.js HTTP framework; native OpenAPI 3.1 schema generation via @fastify/swagger; plugin architecture maps cleanly to phase-by-phase development |
| Database | PostgreSQL 16 | JSONB columns for evolving Podcast 2.0 namespace metadata; GIN indexes for containment queries; full-text search for transcripts; range partitioning for analytics; row-level security for multi-tenancy |
| ORM / query builder | Drizzle ORM | Type-safe SQL with zero runtime overhead; native JSONB operators; migration generation; better PostgreSQL feature coverage than Prisma (partitions, GIN indexes, generated columns) |
| Task queue | BullMQ (Redis-backed) | Audio processing, transcription, and AI generation are all async; BullMQ provides priority queues, job progress tracking, retry with backoff, and rate limiting; Redis also serves as cache layer |
| Object storage | S3-compatible (MinIO for self-hosted) | Audio files are large (50-500MB raw); S3 is the universal object storage API; MinIO provides identical API for self-hosted deployments; pre-signed URLs for secure direct upload/download |
| Audio processing | FFmpeg (via fluent-ffmpeg) | Industry standard for transcoding, loudness analysis (EBU R128), silence detection, and format conversion; no native dependency compilation needed; wraps the ffmpeg binary |
| Transcription | OpenAI Whisper API (cloud) / faster-whisper (self-hosted) | Whisper large-v3 achieves >95% accuracy for English; API for cloud deployments, faster-whisper Python sidecar for self-hosted; WebVTT output format |
| LLM provider | Anthropic Claude API (primary), OpenAI (fallback) | Show notes, chapter generation, clip selection, and guest research all require LLM calls; provider-agnostic adapter pattern allows switching |
| Frontend | Next.js 15 (App Router) | Server components for SEO (podcast website), client components for dashboard; same TypeScript stack; built-in API routes can proxy to Fastify backend |
| UI components | shadcn/ui + Tailwind CSS 4 | Unstyled, accessible components; copy-paste ownership (no dependency); Tailwind for rapid UI development |
| Audio player | wavesurfer.js | Waveform visualisation, region selection (for clip creation), multi-track display; well-maintained; Web Audio API based |
| Authentication | Lucia Auth v3 | Session-based auth with OAuth provider support (Google, GitHub); works with any database; no vendor lock-in; handles email/password and social login |
| RSS generation | fast-xml-parser | Streaming XML generation for RSS 2.0 + iTunes + Podcast 2.0 namespace feeds; handles CDATA, namespaces, and encoding correctly |
| Testing | Vitest + Supertest + Playwright | Vitest for unit/integration (same config as Vite), Supertest for API tests, Playwright for E2E browser tests |
| Linting / formatting | Biome | Single tool replacing ESLint + Prettier; faster execution; consistent formatting and linting in one pass |
| Containerisation | Docker + Docker Compose | Multi-service setup (API, worker, PostgreSQL, Redis, MinIO); Compose for local dev; Dockerfile for production |
| CI/CD | GitHub Actions | Standard for open-source; test, lint, build, and push Docker images on every PR |

### Project Structure

```
podcast-production-platform/
├── package.json
├── pnpm-workspace.yaml
├── turbo.json                          # Turborepo for monorepo task orchestration
├── Dockerfile
├── docker-compose.yml
├── .env.example
├── biome.json
├── packages/
│   ├── db/                             # Drizzle schema, migrations, seed
│   │   ├── src/
│   │   │   ├── schema/
│   │   │   │   ├── organisations.ts
│   │   │   │   ├── users.ts
│   │   │   │   ├── shows.ts
│   │   │   │   ├── episodes.ts
│   │   │   │   ├── audio-files.ts
│   │   │   │   ├── transcripts.ts
│   │   │   │   ├── persons.ts
│   │   │   │   ├── analytics.ts
│   │   │   │   ├── processing-jobs.ts
│   │   │   │   └── api-tokens.ts
│   │   │   ├── migrate.ts
│   │   │   └── seed.ts
│   │   ├── drizzle.config.ts
│   │   └── package.json
│   ├── shared/                         # Shared types, utils, constants
│   │   ├── src/
│   │   │   ├── types/
│   │   │   │   ├── show.ts
│   │   │   │   ├── episode.ts
│   │   │   │   ├── audio.ts
│   │   │   │   ├── rss.ts
│   │   │   │   └── api.ts
│   │   │   ├── constants/
│   │   │   │   ├── lufs-targets.ts
│   │   │   │   ├── mime-types.ts
│   │   │   │   └── itunes-categories.ts
│   │   │   └── utils/
│   │   │       ├── duration.ts
│   │   │       └── slug.ts
│   │   └── package.json
│   └── ui/                             # Shared UI components (shadcn/ui)
│       ├── src/
│       │   └── components/
│       └── package.json
├── apps/
│   ├── api/                            # Fastify API server
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/
│   │   │   │   ├── auth.ts
│   │   │   │   ├── database.ts
│   │   │   │   └── storage.ts
│   │   │   ├── routes/
│   │   │   │   ├── shows/
│   │   │   │   ├── episodes/
│   │   │   │   ├── audio/
│   │   │   │   ├── transcripts/
│   │   │   │   ├── distribution/
│   │   │   │   ├── analytics/
│   │   │   │   └── webhooks/
│   │   │   ├── services/
│   │   │   │   ├── show-service.ts
│   │   │   │   ├── episode-service.ts
│   │   │   │   ├── audio-service.ts
│   │   │   │   ├── transcription-service.ts
│   │   │   │   ├── rss-service.ts
│   │   │   │   ├── distribution-service.ts
│   │   │   │   └── ai-service.ts
│   │   │   └── middleware/
│   │   │       ├── tenant.ts
│   │   │       └── rate-limit.ts
│   │   ├── test/
│   │   └── package.json
│   ├── worker/                         # BullMQ job processors
│   │   ├── src/
│   │   │   ├── worker.ts
│   │   │   ├── processors/
│   │   │   │   ├── transcode.ts
│   │   │   │   ├── noise-removal.ts
│   │   │   │   ├── loudness-norm.ts
│   │   │   │   ├── transcribe.ts
│   │   │   │   ├── chapter-gen.ts
│   │   │   │   ├── show-notes-gen.ts
│   │   │   │   ├── clip-extract.ts
│   │   │   │   └── rss-build.ts
│   │   │   └── lib/
│   │   │       ├── ffmpeg.ts
│   │   │       └── llm.ts
│   │   ├── test/
│   │   └── package.json
│   └── web/                            # Next.js frontend
│       ├── src/
│       │   ├── app/
│       │   │   ├── (auth)/
│       │   │   ├── (dashboard)/
│       │   │   │   ├── shows/
│       │   │   │   ├── episodes/
│       │   │   │   ├── analytics/
│       │   │   │   └── settings/
│       │   │   └── (public)/           # Public podcast website
│       │   │       ├── [show-slug]/
│       │   │       └── [show-slug]/[episode-slug]/
│       │   ├── components/
│       │   └── lib/
│       ├── test/
│       └── package.json
└── fixtures/                           # Test audio files, sample RSS feeds
    ├── audio/
    ├── rss/
    └── transcripts/
```

---

## Phase 1: Foundation — Project Scaffold, Database, and Authentication

### Purpose

Establish the monorepo structure, database schema, authentication system, and Docker development environment. After this phase, a developer can run `docker compose up`, create an account, log in, and hit authenticated API endpoints. Every subsequent phase builds on this foundation.

### Tasks

#### 1.1 — Monorepo Scaffold and Tooling

**What**: Initialise the pnpm workspace, Turborepo config, Biome linting, and Docker Compose stack.

**Design**:

Root `package.json`:
```json
{
  "name": "podcast-production-platform",
  "private": true,
  "packageManager": "pnpm@9.15.0",
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "biome check .",
    "lint:fix": "biome check --write .",
    "db:migrate": "pnpm --filter @ppp/db migrate",
    "db:seed": "pnpm --filter @ppp/db seed"
  }
}
```

`docker-compose.yml`:
```yaml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ppp
      POSTGRES_USER: ppp
      POSTGRES_PASSWORD: ppp_dev_password
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: minioadmin
      MINIO_ROOT_PASSWORD: minioadmin
    ports: ["9000:9000", "9001:9001"]
    volumes: ["miniodata:/data"]
volumes:
  pgdata:
  miniodata:
```

`.env.example`:
```env
DATABASE_URL=postgresql://ppp:ppp_dev_password@localhost:5432/ppp
REDIS_URL=redis://localhost:6379
S3_ENDPOINT=http://localhost:9000
S3_ACCESS_KEY=minioadmin
S3_SECRET_KEY=minioadmin
S3_BUCKET=podcast-assets
S3_REGION=us-east-1
SESSION_SECRET=change-me-in-production
ANTHROPIC_API_KEY=sk-ant-...
OPENAI_API_KEY=sk-...
```

`turbo.json`:
```json
{
  "$schema": "https://turbo.build/schema.json",
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "dev": { "cache": false, "persistent": true },
    "test": { "dependsOn": ["^build"] },
    "lint": {}
  }
}
```

**Testing**:
- `Unit: biome check runs without errors on scaffolded files`
- `Integration: docker compose up starts postgres, redis, minio — all health checks pass`
- `Integration: pnpm install succeeds, turbo build succeeds with empty packages`

---

#### 1.2 — Database Schema and Migrations

**What**: Implement the Hybrid Relational + JSONB data model (Data Model Suggestion 3) using Drizzle ORM.

**Design**:

Core schema types in `packages/db/src/schema/`:

```typescript
// packages/db/src/schema/organisations.ts
import { pgTable, uuid, varchar, text, timestamp, jsonb } from "drizzle-orm/pg-core";

export const organisations = pgTable("organisations", {
  id: uuid("id").primaryKey().defaultRandom(),
  name: varchar("name", { length: 255 }).notNull(),
  slug: varchar("slug", { length: 100 }).notNull().unique(),
  plan: varchar("plan", { length: 50 }).notNull().default("free"),
  settings: jsonb("settings").notNull().default({}),
  billingEmail: varchar("billing_email", { length: 255 }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/users.ts
export const users = pgTable("users", {
  id: uuid("id").primaryKey().defaultRandom(),
  email: varchar("email", { length: 255 }).notNull().unique(),
  displayName: varchar("display_name", { length: 255 }).notNull(),
  avatarUrl: text("avatar_url"),
  passwordHash: text("password_hash"),
  authProvider: varchar("auth_provider", { length: 50 }),
  authProviderId: varchar("auth_provider_id", { length: 255 }),
  preferences: jsonb("preferences").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/organisation-members.ts
export const organisationMembers = pgTable("organisation_members", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  role: varchar("role", { length: 50 }).notNull().default("member"),
  permissions: jsonb("permissions").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueMember: unique().on(table.organisationId, table.userId),
}));

// packages/db/src/schema/shows.ts
export const shows = pgTable("shows", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  title: varchar("title", { length: 500 }).notNull(),
  slug: varchar("slug", { length: 200 }).notNull(),
  description: text("description"),
  language: varchar("language", { length: 10 }).notNull().default("en"),
  author: varchar("author", { length: 255 }),
  artworkUrl: text("artwork_url"),
  status: varchar("status", { length: 20 }).notNull().default("draft"),
  episodeCount: integer("episode_count").notNull().default(0),
  latestEpisodeAt: timestamp("latest_episode_at", { withTimezone: true }),
  itunesMeta: jsonb("itunes_meta").notNull().default({}),
  podcast2Meta: jsonb("podcast2_meta").notNull().default({}),
  seoMeta: jsonb("seo_meta").notNull().default({}),
  distribution: jsonb("distribution").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueSlug: unique().on(table.organisationId, table.slug),
}));

// packages/db/src/schema/episodes.ts
export const episodes = pgTable("episodes", {
  id: uuid("id").primaryKey().defaultRandom(),
  showId: uuid("show_id").notNull().references(() => shows.id, { onDelete: "cascade" }),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  title: varchar("title", { length: 500 }).notNull(),
  slug: varchar("slug", { length: 200 }).notNull(),
  description: text("description"),
  episodeNumber: integer("episode_number"),
  seasonNumber: integer("season_number"),
  durationSeconds: integer("duration_seconds"),
  publishedAt: timestamp("published_at", { withTimezone: true }),
  scheduledAt: timestamp("scheduled_at", { withTimezone: true }),
  status: varchar("status", { length: 20 }).notNull().default("draft"),
  artworkUrl: text("artwork_url"),
  createdBy: uuid("created_by").references(() => users.id),
  itunesMeta: jsonb("itunes_meta").notNull().default({}),
  podcast2Meta: jsonb("podcast2_meta").notNull().default({}),
  enclosure: jsonb("enclosure").notNull().default({}),
  aiContent: jsonb("ai_content").notNull().default({}),
  workflow: jsonb("workflow").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueSlug: unique().on(table.showId, table.slug),
}));

// packages/db/src/schema/audio-files.ts
export const audioFiles = pgTable("audio_files", {
  id: uuid("id").primaryKey().defaultRandom(),
  episodeId: uuid("episode_id").references(() => episodes.id, { onDelete: "set null" }),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  filename: varchar("filename", { length: 500 }).notNull(),
  mimeType: varchar("mime_type", { length: 100 }).notNull(),
  fileSizeBytes: bigint("file_size_bytes", { mode: "number" }).notNull(),
  durationSeconds: numeric("duration_seconds", { precision: 10, scale: 2 }),
  codec: varchar("codec", { length: 50 }),
  filePurpose: varchar("file_purpose", { length: 50 }).notNull().default("episode"),
  trackLabel: varchar("track_label", { length: 100 }),
  isOriginal: boolean("is_original").notNull().default(true),
  sourceFileId: uuid("source_file_id").references(() => audioFiles.id),
  processingStatus: varchar("processing_status", { length: 20 }).notNull().default("pending"),
  storage: jsonb("storage").notNull().default({}),
  audioAnalysis: jsonb("audio_analysis").notNull().default({}),
  id3Tags: jsonb("id3_tags").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/transcripts.ts
export const transcripts = pgTable("transcripts", {
  id: uuid("id").primaryKey().defaultRandom(),
  episodeId: uuid("episode_id").notNull().references(() => episodes.id, { onDelete: "cascade" }),
  format: varchar("format", { length: 20 }).notNull(),
  language: varchar("language", { length: 10 }).notNull().default("en"),
  content: text("content"),
  storageKey: text("storage_key"),
  publicUrl: text("public_url"),
  generationMeta: jsonb("generation_meta").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/persons.ts
export const persons = pgTable("persons", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  fullName: varchar("full_name", { length: 255 }).notNull(),
  email: varchar("email", { length: 255 }),
  bio: text("bio"),
  avatarUrl: text("avatar_url"),
  contactInfo: jsonb("contact_info").notNull().default({}),
  appearanceCount: integer("appearance_count").notNull().default(0),
  lastAppearedAt: timestamp("last_appeared_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/processing-jobs.ts
export const processingJobs = pgTable("processing_jobs", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  episodeId: uuid("episode_id").references(() => episodes.id, { onDelete: "set null" }),
  audioFileId: uuid("audio_file_id").references(() => audioFiles.id, { onDelete: "set null" }),
  jobType: varchar("job_type", { length: 50 }).notNull(),
  status: varchar("status", { length: 20 }).notNull().default("queued"),
  priority: integer("priority").notNull().default(5),
  params: jsonb("params").notNull().default({}),
  errorMessage: text("error_message"),
  startedAt: timestamp("started_at", { withTimezone: true }),
  completedAt: timestamp("completed_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/src/schema/analytics.ts
export const analyticsEvents = pgTable("analytics_events", {
  id: uuid("id").primaryKey().defaultRandom(),
  episodeId: uuid("episode_id").notNull(),
  showId: uuid("show_id").notNull(),
  eventType: varchar("event_type", { length: 50 }).notNull(),
  eventData: jsonb("event_data").notNull().default({}),
  occurredAt: timestamp("occurred_at", { withTimezone: true }).notNull().defaultNow(),
});

export const analyticsDaily = pgTable("analytics_daily", {
  showId: uuid("show_id").notNull(),
  episodeId: uuid("episode_id").notNull(),
  date: date("date").notNull(),
  stats: jsonb("stats").notNull().default({}),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  pk: primaryKey({ columns: [table.episodeId, table.date] }),
}));

// packages/db/src/schema/api-tokens.ts
export const apiTokens = pgTable("api_tokens", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  userId: uuid("user_id").notNull().references(() => users.id, { onDelete: "cascade" }),
  name: varchar("name", { length: 255 }).notNull(),
  tokenHash: varchar("token_hash", { length: 64 }).notNull().unique(),
  scopes: text("scopes").array().notNull().default([]),
  expiresAt: timestamp("expires_at", { withTimezone: true }),
  lastUsedAt: timestamp("last_used_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
});

export const webhooks = pgTable("webhooks", {
  id: uuid("id").primaryKey().defaultRandom(),
  organisationId: uuid("organisation_id").notNull().references(() => organisations.id, { onDelete: "cascade" }),
  url: text("url").notNull(),
  events: text("events").array().notNull(),
  secret: varchar("secret", { length: 255 }).notNull(),
  isActive: boolean("is_active").notNull().default(true),
  config: jsonb("config").notNull().default({}),
  lastTriggeredAt: timestamp("last_triggered_at", { withTimezone: true }),
  createdAt: timestamp("created_at", { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp("updated_at", { withTimezone: true }).notNull().defaultNow(),
});
```

GIN indexes added via raw SQL migration:

```sql
CREATE INDEX idx_shows_itunes ON shows USING gin(itunes_meta);
CREATE INDEX idx_shows_podcast2 ON shows USING gin(podcast2_meta);
CREATE INDEX idx_episodes_podcast2 ON episodes USING gin(podcast2_meta);
CREATE INDEX idx_episodes_ai ON episodes USING gin(ai_content);
CREATE INDEX idx_episodes_fts ON episodes USING gin(
  to_tsvector('english', title || ' ' || COALESCE(description, ''))
);
CREATE INDEX idx_transcripts_fts ON transcripts USING gin(
  to_tsvector('english', content)
);
CREATE INDEX idx_analytics_data ON analytics_events USING gin(event_data);
```

**Testing**:
- `Unit: Drizzle schema compiles without type errors`
- `Integration: drizzle-kit generate produces valid SQL migration files`
- `Integration: migration applies cleanly to fresh PostgreSQL 16 database`
- `Integration: migration is idempotent — running twice does not error`
- `Integration: all GIN indexes are created and queryable`
- `Fixture: seed script inserts sample organisation, user, show, and episode`

---

#### 1.3 — Fastify API Server with Health Checks

**What**: Set up the Fastify server with plugins for database, Redis, and OpenAPI documentation.

**Design**:

```typescript
// apps/api/src/server.ts
import Fastify from "fastify";
import cors from "@fastify/cors";
import swagger from "@fastify/swagger";
import swaggerUi from "@fastify/swagger-ui";
import { databasePlugin } from "./plugins/database";
import { redisPlugin } from "./plugins/redis";

export async function buildServer() {
  const app = Fastify({
    logger: { level: process.env.LOG_LEVEL ?? "info" },
    genReqId: () => crypto.randomUUID(),
  });

  // Core plugins
  await app.register(cors, { origin: process.env.CORS_ORIGIN ?? "http://localhost:3000" });
  await app.register(swagger, {
    openapi: {
      info: { title: "Podcast Production Platform API", version: "1.0.0" },
      servers: [{ url: process.env.API_URL ?? "http://localhost:4000" }],
    },
  });
  await app.register(swaggerUi, { routePrefix: "/docs" });
  await app.register(databasePlugin);
  await app.register(redisPlugin);

  // Health check
  app.get("/health", {
    schema: {
      response: {
        200: {
          type: "object",
          properties: {
            status: { type: "string" },
            database: { type: "string" },
            redis: { type: "string" },
            timestamp: { type: "string" },
          },
        },
      },
    },
  }, async (request, reply) => {
    const dbOk = await app.db.execute(sql`SELECT 1`).then(() => "ok").catch(() => "error");
    const redisOk = await app.redis.ping().then(() => "ok").catch(() => "error");
    return {
      status: dbOk === "ok" && redisOk === "ok" ? "healthy" : "degraded",
      database: dbOk,
      redis: redisOk,
      timestamp: new Date().toISOString(),
    };
  });

  return app;
}
```

```typescript
// apps/api/src/plugins/database.ts
import fp from "fastify-plugin";
import { drizzle } from "drizzle-orm/node-postgres";
import { Pool } from "pg";
import * as schema from "@ppp/db/schema";

export const databasePlugin = fp(async (app) => {
  const pool = new Pool({ connectionString: process.env.DATABASE_URL });
  const db = drizzle(pool, { schema });
  app.decorate("db", db);
  app.addHook("onClose", async () => pool.end());
});
```

**Testing**:
- `Integration: GET /health returns 200 with status "healthy" when all services are up`
- `Integration: GET /health returns "degraded" when database is unreachable`
- `Integration: GET /docs returns Swagger UI HTML`
- `Integration: OpenAPI JSON spec at /docs/json contains correct title and version`
- `Unit: server starts and shuts down cleanly without hanging connections`

---

#### 1.4 — Authentication System

**What**: Implement email/password registration, login, session management, and organisation creation using Lucia Auth.

**Design**:

```typescript
// packages/shared/src/types/auth.ts
export interface RegisterRequest {
  email: string;
  password: string;
  displayName: string;
  organisationName: string;
}

export interface LoginRequest {
  email: string;
  password: string;
}

export interface SessionUser {
  userId: string;
  email: string;
  displayName: string;
  organisationId: string;
  role: "owner" | "admin" | "editor" | "member";
}
```

API endpoints:

| Method | Path | Request Body | Response | Auth |
|--------|------|-------------|----------|------|
| POST | `/api/auth/register` | `RegisterRequest` | `{ user, organisation, sessionToken }` | No |
| POST | `/api/auth/login` | `LoginRequest` | `{ user, sessionToken }` | No |
| POST | `/api/auth/logout` | — | `{ ok: true }` | Yes |
| GET | `/api/auth/me` | — | `SessionUser` | Yes |

Password hashing: Argon2id via `@node-rs/argon2` (recommended by Lucia Auth).

Session storage: PostgreSQL `sessions` table (Lucia adapter).

Auth middleware:
```typescript
// apps/api/src/middleware/auth.ts
export const authMiddleware = fp(async (app) => {
  app.decorateRequest("session", null);
  app.addHook("preHandler", async (request, reply) => {
    const sessionToken = request.headers.authorization?.replace("Bearer ", "")
      ?? request.cookies?.session_token;
    if (!sessionToken) {
      reply.code(401).send({ error: "Unauthorized" });
      return;
    }
    const session = await lucia.validateSession(sessionToken);
    if (!session) {
      reply.code(401).send({ error: "Session expired" });
      return;
    }
    request.session = session;
  });
});
```

Tenant middleware (applied after auth):
```typescript
// apps/api/src/middleware/tenant.ts
export const tenantMiddleware = fp(async (app) => {
  app.addHook("preHandler", async (request, reply) => {
    // Extract organisation from session or X-Organisation-Id header
    const orgId = request.headers["x-organisation-id"] ?? request.session.organisationId;
    request.organisationId = orgId;
  });
});
```

**Testing**:
- `Integration: POST /api/auth/register with valid data -> 201, user and org created, session token returned`
- `Integration: POST /api/auth/register with duplicate email -> 409 Conflict`
- `Integration: POST /api/auth/register with weak password (<8 chars) -> 422 Validation Error`
- `Integration: POST /api/auth/login with correct credentials -> 200, session token returned`
- `Integration: POST /api/auth/login with wrong password -> 401 Unauthorized`
- `Integration: GET /api/auth/me with valid session -> 200, user data returned`
- `Integration: GET /api/auth/me without session -> 401 Unauthorized`
- `Integration: POST /api/auth/logout invalidates session — subsequent /me returns 401`
- `Unit: password hashing produces different hash for same input (salt is unique)`
- `Unit: Argon2id verification succeeds for correct password, fails for incorrect`

---

## Phase 2: Show and Episode Management — Core CRUD

### Purpose

Build the primary content management layer: creating, reading, updating, and deleting shows and episodes. After this phase, users can manage their podcast catalogue through the API, including metadata for RSS 2.0 and iTunes namespace fields. This is the backbone of all subsequent features.

### Tasks

#### 2.1 — Show CRUD API

**What**: Implement full CRUD operations for podcast shows, including iTunes and Podcast 2.0 metadata stored in JSONB columns.

**Design**:

```typescript
// packages/shared/src/types/show.ts
export interface CreateShowRequest {
  title: string;
  slug?: string; // auto-generated from title if omitted
  description?: string;
  language?: string; // BCP 47, default "en"
  author?: string;
  artworkUrl?: string;
  itunes?: {
    category: string;
    subcategory?: string;
    type?: "episodic" | "serial";
    explicit?: boolean;
    ownerName?: string;
    ownerEmail?: string;
  };
}

export interface UpdateShowRequest extends Partial<CreateShowRequest> {
  status?: "draft" | "active" | "paused" | "archived";
  podcast2?: {
    guid?: string;
    medium?: "podcast" | "music" | "video" | "film" | "audiobook";
    locked?: boolean;
    lockOwner?: string;
    funding?: Array<{ url: string; message?: string }>;
    updateFrequency?: string;
  };
}

export interface ShowResponse {
  id: string;
  organisationId: string;
  title: string;
  slug: string;
  description: string | null;
  language: string;
  author: string | null;
  artworkUrl: string | null;
  status: string;
  episodeCount: number;
  latestEpisodeAt: string | null;
  itunes: Record<string, unknown>;
  podcast2: Record<string, unknown>;
  distribution: Record<string, unknown>;
  createdAt: string;
  updatedAt: string;
}

export interface ShowListResponse {
  shows: ShowResponse[];
  total: number;
  page: number;
  pageSize: number;
}
```

API endpoints:

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/shows` | Create a new show |
| GET | `/api/shows` | List shows (paginated, filterable by status) |
| GET | `/api/shows/:showId` | Get show by ID |
| PATCH | `/api/shows/:showId` | Update show |
| DELETE | `/api/shows/:showId` | Soft-delete (set status to "archived") |

Slug generation: `slugify(title)` with collision detection (append `-2`, `-3` if taken within the same organisation).

Validation: Zod schemas validate all request bodies; Fastify schema validation for response serialisation.

**Testing**:
- `Integration: POST /api/shows with valid data -> 201, show created with correct JSONB metadata`
- `Integration: POST /api/shows without title -> 422 validation error`
- `Integration: POST /api/shows with duplicate slug in same org -> auto-appends suffix`
- `Integration: GET /api/shows returns paginated list, default 20 per page`
- `Integration: GET /api/shows?status=active filters correctly`
- `Integration: GET /api/shows/:id returns full show including JSONB fields`
- `Integration: PATCH /api/shows/:id with itunes metadata -> merges into itunes_meta JSONB`
- `Integration: DELETE /api/shows/:id sets status to "archived", does not hard delete`
- `Integration: shows are scoped to organisation — user cannot access another org's shows`
- `Unit: slug generation handles unicode, spaces, and special characters`
- `Unit: slug collision detection appends correct suffix`

---

#### 2.2 — Episode CRUD API

**What**: Implement CRUD operations for episodes within a show, including season/episode numbering and status lifecycle management.

**Design**:

```typescript
// packages/shared/src/types/episode.ts
export type EpisodeStatus = "draft" | "recording" | "processing" | "review" | "scheduled" | "published" | "archived";

export interface CreateEpisodeRequest {
  title: string;
  slug?: string;
  description?: string;
  episodeNumber?: number;
  seasonNumber?: number;
  itunes?: {
    episodeType?: "full" | "trailer" | "bonus";
    explicit?: boolean;
  };
}

export interface UpdateEpisodeRequest extends Partial<CreateEpisodeRequest> {
  status?: EpisodeStatus;
  scheduledAt?: string; // ISO 8601
  artworkUrl?: string;
}

export interface EpisodeResponse {
  id: string;
  showId: string;
  title: string;
  slug: string;
  description: string | null;
  episodeNumber: number | null;
  seasonNumber: number | null;
  durationSeconds: number | null;
  publishedAt: string | null;
  scheduledAt: string | null;
  status: EpisodeStatus;
  artworkUrl: string | null;
  itunes: Record<string, unknown>;
  podcast2: Record<string, unknown>;
  enclosure: Record<string, unknown>;
  aiContent: Record<string, unknown>;
  workflow: Record<string, unknown>;
  createdBy: string | null;
  createdAt: string;
  updatedAt: string;
}
```

Episode status transitions (state machine):
```
draft -> recording -> processing -> review -> scheduled -> published
                                          \-> published (skip schedule)
Any state -> archived
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/shows/:showId/episodes` | Create episode |
| GET | `/api/shows/:showId/episodes` | List episodes (paginated, filterable by status/season) |
| GET | `/api/shows/:showId/episodes/:episodeId` | Get episode |
| PATCH | `/api/shows/:showId/episodes/:episodeId` | Update episode |
| DELETE | `/api/shows/:showId/episodes/:episodeId` | Archive episode |
| POST | `/api/shows/:showId/episodes/:episodeId/publish` | Publish episode (sets publishedAt, status) |

Auto-numbering: if `episodeNumber` is omitted, assign `MAX(episode_number) + 1` within the show (or season, if serial).

**Testing**:
- `Integration: POST /api/shows/:id/episodes creates episode with correct defaults`
- `Integration: episode_number auto-increments when omitted`
- `Integration: GET /api/shows/:id/episodes?status=published returns only published episodes`
- `Integration: GET /api/shows/:id/episodes?season=2 filters by season`
- `Integration: PATCH with status transition draft->review succeeds`
- `Integration: PATCH with invalid status transition published->draft returns 422`
- `Integration: POST .../publish sets publishedAt to current time, status to "published", updates show.episode_count`
- `Integration: episode scoped to show — cannot access episode from wrong show`
- `Unit: status transition validator accepts valid transitions, rejects invalid`
- `Unit: auto-numbering handles concurrent creates (SELECT FOR UPDATE)`

---

#### 2.3 — Guest/Person Management API

**What**: Implement CRUD for guest and contributor records, with the ability to associate persons with episodes.

**Design**:

```typescript
// packages/shared/src/types/person.ts
export interface CreatePersonRequest {
  fullName: string;
  email?: string;
  bio?: string;
  avatarUrl?: string;
  contactInfo?: {
    website?: string;
    twitter?: string;
    linkedin?: string;
    mastodon?: string;
  };
}

export interface EpisodePersonAssignment {
  personId: string;
  role: "host" | "co-host" | "guest" | "editor" | "producer";
  roleGroup?: "cast" | "crew";
  displayName?: string; // override for this appearance
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/persons` | Create person |
| GET | `/api/persons` | List persons (search by name) |
| GET | `/api/persons/:personId` | Get person with appearance history |
| PATCH | `/api/persons/:personId` | Update person |
| POST | `/api/episodes/:episodeId/persons` | Assign person to episode |
| DELETE | `/api/episodes/:episodeId/persons/:personId` | Remove person from episode |

Persons assigned to an episode are stored in `episodes.podcast2_meta.persons[]` (JSONB array) and the person's `appearance_count` and `last_appeared_at` are updated.

**Testing**:
- `Integration: POST /api/persons creates person with JSONB contact_info`
- `Integration: GET /api/persons?search=John returns matching persons (case-insensitive)`
- `Integration: POST /api/episodes/:id/persons assigns person, updates podcast2_meta.persons[]`
- `Integration: assigning same person twice with same role returns 409`
- `Integration: removing person from episode decrements appearance_count`
- `Integration: GET /api/persons/:id includes appearance history (episodes appeared on)`
- `Unit: person search handles partial matches and diacritics`

---

## Phase 3: Audio File Management and Object Storage

### Purpose

Enable uploading, storing, and managing audio files (raw tracks and processed versions). After this phase, users can upload multi-track recordings, track audio file provenance (original to processed), and retrieve files via pre-signed URLs. This is the prerequisite for all audio processing in Phase 4.

### Tasks

#### 3.1 — S3 Storage Abstraction

**What**: Implement an S3-compatible storage service supporting upload, download, pre-signed URLs, and multi-part upload for large audio files.

**Design**:

```typescript
// apps/api/src/services/storage-service.ts
import { S3Client, PutObjectCommand, GetObjectCommand, DeleteObjectCommand } from "@aws-sdk/client-s3";
import { getSignedUrl } from "@aws-sdk/s3-request-presigner";
import { Upload } from "@aws-sdk/lib-storage";

export interface StorageConfig {
  endpoint: string;
  accessKey: string;
  secretKey: string;
  bucket: string;
  region: string;
  publicBaseUrl?: string; // CDN URL prefix
}

export interface UploadResult {
  key: string;
  bucket: string;
  size: number;
  etag: string;
  publicUrl: string;
}

export interface StorageService {
  generateUploadUrl(key: string, contentType: string, expiresIn?: number): Promise<string>;
  generateDownloadUrl(key: string, expiresIn?: number): Promise<string>;
  upload(key: string, body: Buffer | Readable, contentType: string): Promise<UploadResult>;
  delete(key: string): Promise<void>;
  exists(key: string): Promise<boolean>;
  getMetadata(key: string): Promise<{ size: number; contentType: string; lastModified: Date }>;
}
```

Key naming convention: `{organisationId}/{showId}/{episodeId}/{purpose}/{filename}`

Example: `org-uuid/show-uuid/ep-uuid/raw/host-track.wav`

Pre-signed upload flow:
1. Client requests upload URL: `POST /api/audio/upload-url`
2. Server generates pre-signed S3 PUT URL (expires in 1 hour)
3. Client uploads directly to S3
4. Client confirms upload: `POST /api/audio/confirm-upload`
5. Server verifies file exists, creates `audio_files` record

**Testing**:
- `Integration (MinIO): upload file via pre-signed URL, download via pre-signed URL — content matches`
- `Integration (MinIO): delete file, verify exists returns false`
- `Integration (MinIO): multi-part upload for 100MB+ file completes successfully`
- `Unit: key generation follows naming convention`
- `Unit: pre-signed URL expiration is configurable, defaults to 3600 seconds`
- `Unit: storage service rejects keys with path traversal patterns (../)`

---

#### 3.2 — Audio File Upload and Management API

**What**: Implement the API for uploading audio files, tracking metadata, and managing file provenance chains.

**Design**:

```typescript
// packages/shared/src/types/audio.ts
export type FilePurpose = "raw_track" | "processed" | "episode" | "clip" | "trailer";
export type ProcessingStatus = "pending" | "uploading" | "uploaded" | "processing" | "complete" | "failed";

export interface RequestUploadUrlRequest {
  episodeId: string;
  filename: string;
  mimeType: string; // audio/wav, audio/mpeg, audio/x-m4a, audio/flac
  fileSizeBytes: number;
  filePurpose: FilePurpose;
  trackLabel?: string; // "Host", "Guest 1"
}

export interface RequestUploadUrlResponse {
  uploadUrl: string;
  audioFileId: string; // pre-created with status "uploading"
  expiresAt: string;
}

export interface ConfirmUploadRequest {
  audioFileId: string;
}

export interface AudioFileResponse {
  id: string;
  episodeId: string | null;
  filename: string;
  mimeType: string;
  fileSizeBytes: number;
  durationSeconds: number | null;
  codec: string | null;
  filePurpose: FilePurpose;
  trackLabel: string | null;
  isOriginal: boolean;
  sourceFileId: string | null;
  processingStatus: ProcessingStatus;
  audioAnalysis: {
    lufsIntegrated?: number;
    lufsTruePeak?: number;
    sampleRate?: number;
    bitRate?: number;
    channels?: number;
  };
  downloadUrl: string; // pre-signed
  createdAt: string;
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/audio/upload-url` | Get pre-signed upload URL |
| POST | `/api/audio/confirm-upload` | Confirm upload completed |
| GET | `/api/episodes/:episodeId/audio` | List audio files for episode |
| GET | `/api/audio/:fileId` | Get audio file metadata + download URL |
| DELETE | `/api/audio/:fileId` | Delete audio file (storage + record) |

On confirm-upload, the worker probes the file with FFmpeg to extract duration, sample rate, bit rate, channels, and codec, then updates the `audio_files` record.

**Testing**:
- `Integration: full upload flow — request URL, upload to S3, confirm, probe metadata`
- `Integration: upload rejected if mime_type not in allowed list`
- `Integration: file size limit enforced (configurable, default 500MB)`
- `Integration: GET /api/episodes/:id/audio returns all tracks with download URLs`
- `Integration: DELETE removes both S3 object and database record`
- `Integration: confirm-upload with non-existent audioFileId returns 404`
- `Unit: mime type validation accepts audio/wav, audio/mpeg, audio/x-m4a, audio/flac, audio/ogg`
- `Unit: mime type validation rejects video/mp4, application/pdf`

---

## Phase 4: Audio Processing Pipeline

### Purpose

Build the async processing pipeline that transforms raw audio tracks into production-ready podcast episodes. This includes transcoding, noise removal, loudness normalisation (EBU R128 / -16 LUFS), silence trimming, and filler-word removal. After this phase, a user can upload raw tracks and receive a processed, broadcast-ready audio file.

### Tasks

#### 4.1 — BullMQ Worker Infrastructure

**What**: Set up the BullMQ worker process with job routing, retry logic, progress reporting, and dead-letter handling.

**Design**:

```typescript
// apps/worker/src/worker.ts
import { Worker, Queue, QueueEvents } from "bullmq";
import { Redis } from "ioredis";

export interface JobPayload {
  organisationId: string;
  episodeId?: string;
  audioFileId?: string;
  params: Record<string, unknown>;
}

export type JobType =
  | "audio:probe"
  | "audio:transcode"
  | "audio:noise-removal"
  | "audio:loudness-norm"
  | "audio:silence-trim"
  | "audio:filler-removal"
  | "ai:transcribe"
  | "ai:chapter-gen"
  | "ai:show-notes"
  | "ai:clip-extract"
  | "rss:build";

const JOB_CONFIG: Record<JobType, { concurrency: number; attempts: number; backoff: { type: string; delay: number } }> = {
  "audio:probe":         { concurrency: 10, attempts: 3, backoff: { type: "exponential", delay: 1000 } },
  "audio:transcode":     { concurrency: 4,  attempts: 3, backoff: { type: "exponential", delay: 5000 } },
  "audio:noise-removal": { concurrency: 2,  attempts: 2, backoff: { type: "exponential", delay: 5000 } },
  "audio:loudness-norm": { concurrency: 4,  attempts: 3, backoff: { type: "exponential", delay: 2000 } },
  "audio:silence-trim":  { concurrency: 4,  attempts: 2, backoff: { type: "exponential", delay: 2000 } },
  "audio:filler-removal":{ concurrency: 2,  attempts: 2, backoff: { type: "exponential", delay: 5000 } },
  "ai:transcribe":       { concurrency: 2,  attempts: 3, backoff: { type: "exponential", delay: 10000 } },
  "ai:chapter-gen":      { concurrency: 4,  attempts: 2, backoff: { type: "exponential", delay: 5000 } },
  "ai:show-notes":       { concurrency: 4,  attempts: 2, backoff: { type: "exponential", delay: 5000 } },
  "ai:clip-extract":     { concurrency: 2,  attempts: 2, backoff: { type: "exponential", delay: 5000 } },
  "rss:build":           { concurrency: 8,  attempts: 3, backoff: { type: "exponential", delay: 1000 } },
};
```

Each job updates the `processing_jobs` table with status, timestamps, and error messages. Job progress is reported via BullMQ's `updateProgress` method.

Processing pipeline (orchestrated via BullMQ flow):
```
upload confirmed
  -> audio:probe
    -> audio:noise-removal
      -> audio:loudness-norm
        -> audio:silence-trim
          -> audio:transcode (to MP3 192kbps)
            -> episode status updated to "processing complete"
```

**Testing**:
- `Integration: enqueue job -> worker picks it up within 1 second`
- `Integration: failed job retries with exponential backoff`
- `Integration: after max retries, job moves to failed state with error message`
- `Integration: job progress updates are recorded in processing_jobs table`
- `Integration: concurrent job limit is respected per job type`
- `Unit: job routing maps job type to correct processor function`
- `Unit: dead-letter handler records failure in processing_jobs table`

---

#### 4.2 — FFmpeg Audio Processing Processors

**What**: Implement the individual audio processing jobs: probe, transcode, noise removal, loudness normalisation, and silence trimming.

**Design**:

```typescript
// apps/worker/src/lib/ffmpeg.ts
export interface ProbeResult {
  durationSeconds: number;
  sampleRate: number;
  bitRate: number;
  channels: number;
  codec: string;
  lufsIntegrated: number;
  lufsTruePeak: number;
  lufsRange: number;
}

export interface TranscodeOptions {
  outputCodec: "mp3" | "aac" | "flac";
  bitRate: number; // 128000, 192000, 256000, 320000
  sampleRate: number; // 44100, 48000
  channels: 1 | 2;
}

export interface LoudnessNormOptions {
  targetLufs: number; // default -16.0 (EBU R128 for podcasts)
  truePeakLimit: number; // default -1.0 dBTP
}

export async function probe(inputPath: string): Promise<ProbeResult>;
export async function transcode(inputPath: string, outputPath: string, options: TranscodeOptions): Promise<void>;
export async function normalise(inputPath: string, outputPath: string, options: LoudnessNormOptions): Promise<ProbeResult>;
export async function removeNoise(inputPath: string, outputPath: string, noiseProfile?: string): Promise<void>;
export async function trimSilence(inputPath: string, outputPath: string, thresholdDb?: number, minDurationMs?: number): Promise<void>;
```

FFmpeg commands used:
- **Probe**: `ffprobe -v quiet -print_format json -show_format -show_streams`
- **Loudness measurement**: `ffmpeg -i input -af loudnorm=print_format=json -f null -` (two-pass)
- **Loudness normalisation**: `ffmpeg -i input -af loudnorm=I={target}:TP={peak}:LRA=11:measured_I={measured}:measured_TP={measuredPeak}:measured_LRA={measuredLRA}:measured_thresh={thresh} output` (second pass with measured values)
- **Noise removal**: `ffmpeg -i input -af afftdn=nf=-25 output`
- **Silence trimming**: `ffmpeg -i input -af silenceremove=start_periods=1:start_silence={min}:start_threshold={thresh}dB:stop_periods=-1:stop_silence={min}:stop_threshold={thresh}dB output`
- **Transcode**: `ffmpeg -i input -codec:a {codec} -b:a {bitrate} -ar {sampleRate} -ac {channels} output`

Each processor:
1. Downloads input file from S3 to a temp directory
2. Runs FFmpeg command
3. Uploads output file to S3
4. Creates new `audio_files` record with `source_file_id` pointing to input
5. Updates `processing_jobs` record with output params
6. Cleans up temp files

**Testing**:
- `Integration (FFmpeg): probe WAV file -> correct duration, sample rate, channels`
- `Integration (FFmpeg): probe MP3 file -> correct bit rate and codec`
- `Integration (FFmpeg): transcode WAV -> MP3 at 192kbps -> output is valid MP3 with correct bit rate`
- `Integration (FFmpeg): loudness normalise -> output LUFS within 0.5 of target -16.0`
- `Integration (FFmpeg): silence trim on file with leading/trailing silence -> shorter duration`
- `Integration (FFmpeg): noise removal on noisy file -> noise floor reduced`
- `Fixture: use 10-second test WAV files committed to fixtures/audio/`
- `Unit: TranscodeOptions validates bit rate is in allowed set`
- `Unit: LoudnessNormOptions defaults to -16.0 LUFS and -1.0 dBTP`

---

#### 4.3 — One-Click Post-Production Pipeline

**What**: Implement the orchestrated post-production pipeline that chains noise removal, loudness normalisation, silence trimming, and transcoding into a single automated workflow triggered by a single API call.

**Design**:

```typescript
// packages/shared/src/types/processing.ts
export interface PostProductionRequest {
  episodeId: string;
  audioFileIds: string[]; // raw track IDs to process
  options?: {
    noiseRemoval?: boolean; // default true
    loudnessTarget?: number; // default -16.0 LUFS
    silenceTrim?: boolean; // default true
    silenceThresholdDb?: number; // default -40
    outputCodec?: "mp3" | "aac"; // default "mp3"
    outputBitRate?: number; // default 192000
    mixdown?: boolean; // default true (mix multi-track to stereo)
  };
}

export interface PostProductionStatus {
  episodeId: string;
  status: "queued" | "processing" | "complete" | "failed";
  steps: Array<{
    name: string;
    status: "pending" | "running" | "complete" | "failed" | "skipped";
    progress?: number; // 0-100
    error?: string;
  }>;
  outputFileId?: string;
}
```

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/episodes/:episodeId/process` | Start post-production pipeline |
| GET | `/api/episodes/:episodeId/process/status` | Get pipeline status |

The pipeline creates a BullMQ flow (directed acyclic graph of jobs):

```
[for each track]:
  noise-removal -> loudness-norm -> silence-trim
[after all tracks complete]:
  mix-down (if multi-track) -> transcode -> update episode enclosure
```

Episode `workflow` JSONB is updated at each step:
```json
{
  "stages_completed": ["uploaded", "noise_removed", "normalized", "trimmed", "mixed", "transcoded"],
  "current_stage": "transcoded",
  "processing_jobs": ["job-uuid-1", "job-uuid-2"],
  "last_processed_at": "2026-05-25T10:00:00Z"
}
```

**Testing**:
- `Integration: POST /api/episodes/:id/process with 2 raw tracks -> pipeline completes, final MP3 created`
- `Integration: pipeline status endpoint shows correct step progression`
- `Integration: if noise removal fails, pipeline marks episode status as "failed" with error`
- `Integration: episode workflow JSONB updated at each step`
- `Integration: final output file has LUFS within tolerance of target`
- `Integration: final output has correct codec and bit rate`
- `Fixture: use committed multi-track WAV fixtures`
- `Unit: pipeline skips noise removal when options.noiseRemoval is false`
- `Unit: mono tracks mixed to stereo in mixdown step`

---

## Phase 5: Transcription and AI Content Generation

### Purpose

Add automatic transcription (via Whisper) and AI-generated content: show notes, chapter markers, and social media clip identification. After this phase, uploading and processing an episode automatically generates a searchable transcript, chapter markers, show notes, and suggested social clips.

### Tasks

#### 5.1 — Transcription Service

**What**: Implement automatic speech-to-text transcription using Whisper, producing WebVTT and plain text output stored in the transcripts table.

**Design**:

```typescript
// apps/worker/src/processors/transcribe.ts
export interface TranscriptionJobPayload {
  organisationId: string;
  episodeId: string;
  audioFileId: string;
  language?: string; // BCP 47; auto-detect if omitted
  model?: string; // "whisper-large-v3" (default)
}

export interface TranscriptionResult {
  transcriptId: string;
  format: "vtt";
  language: string;
  wordCount: number;
  confidenceScore: number;
  durationMs: number; // processing time
  vttContent: string;
  plainTextContent: string;
  storageKey: string;
  publicUrl: string;
}
```

Whisper integration (provider-agnostic adapter):
```typescript
// apps/worker/src/lib/whisper.ts
export interface WhisperProvider {
  transcribe(audioPath: string, options: { language?: string; model?: string }): Promise<WhisperResult>;
}

export interface WhisperResult {
  text: string;
  segments: Array<{
    start: number; // seconds
    end: number;
    text: string;
    confidence: number;
  }>;
  language: string;
  duration: number;
}

// OpenAI Whisper API provider (cloud)
export class OpenAIWhisperProvider implements WhisperProvider { ... }
// Local faster-whisper provider (self-hosted)
export class LocalWhisperProvider implements WhisperProvider { ... }
```

WebVTT generation from segments:
```typescript
function segmentsToVTT(segments: WhisperResult["segments"]): string {
  // WEBVTT\n\n
  // 00:00:00.000 --> 00:00:05.200\nHello and welcome to the show.\n\n
  // ...
}
```

The transcript is stored in both the `transcripts` table (for full-text search) and referenced in `episodes.podcast2_meta.transcript` (for RSS generation).

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/episodes/:episodeId/transcribe` | Trigger transcription |
| GET | `/api/episodes/:episodeId/transcript` | Get transcript (VTT or plain text) |
| PUT | `/api/episodes/:episodeId/transcript` | Replace transcript (manual edit) |

**Testing**:
- `Integration (mocked Whisper): transcription job produces valid WebVTT output`
- `Integration (mocked Whisper): transcript stored in database with full-text search index`
- `Integration: full-text search query against transcript content returns matching episodes`
- `Integration: GET /api/episodes/:id/transcript returns VTT content with correct MIME type`
- `Integration: PUT /api/episodes/:id/transcript replaces content and updates search index`
- `Integration: episode podcast2_meta.transcript updated with public URL`
- `Fixture: use committed VTT fixtures for testing parser and renderer`
- `Unit: segmentsToVTT produces valid WebVTT format with correct timestamps`
- `Unit: confidence score is average of segment confidences`

---

#### 5.2 — AI Chapter Generation

**What**: Use an LLM to analyse the transcript and generate chapter markers in JSON Chapters format (per Podcast 2.0 spec).

**Design**:

```typescript
// apps/worker/src/processors/chapter-gen.ts
export interface ChapterGenPayload {
  organisationId: string;
  episodeId: string;
  transcriptId: string;
}

// Output matches the Podcast 2.0 JSON Chapters Format
// https://github.com/Podcastindex-org/podcast-namespace/blob/main/chapters/jsonChapters.md
export interface JsonChapters {
  version: "1.2.0";
  chapters: Array<{
    startTime: number; // seconds
    title: string;
    img?: string;
    url?: string;
    toc?: boolean; // show in table of contents
  }>;
}
```

LLM prompt structure:
```typescript
const CHAPTER_GEN_SYSTEM_PROMPT = `You are an expert podcast editor. Given a transcript, identify the major topic transitions and generate chapter markers.

Rules:
- Generate 4-12 chapters per episode
- First chapter starts at 0 seconds
- Chapter titles should be concise (3-8 words)
- Identify natural topic shifts, not arbitrary time intervals
- Include an "Intro" and "Outro" chapter if the episode has them

Output JSON matching the Podcast 2.0 JSON Chapters Format.`;
```

The chapters JSON file is uploaded to S3 and referenced in `episodes.podcast2_meta.chapters`.

**Testing**:
- `Integration (mocked LLM): chapter generation produces valid JSON Chapters format`
- `Integration: chapters JSON uploaded to S3, URL stored in podcast2_meta`
- `Integration: generated chapters have startTime in ascending order`
- `Integration: first chapter starts at 0`
- `Unit: JSON Chapters output validates against the Podcast 2.0 spec schema`
- `Unit: chapter count is between 4 and 12`
- `Unit: handles transcripts of varying lengths (5 min to 3 hours)`

---

#### 5.3 — AI Show Notes Generation

**What**: Generate episode show notes (summary, key points, keywords) from the transcript using an LLM.

**Design**:

```typescript
// apps/worker/src/processors/show-notes-gen.ts
export interface ShowNotesPayload {
  organisationId: string;
  episodeId: string;
  transcriptId: string;
}

export interface GeneratedShowNotes {
  summary: string; // 2-3 paragraph summary
  keyPoints: string[]; // 5-10 bullet points
  keywords: string[]; // 5-15 tags
  actionItems?: string[]; // if any calls-to-action mentioned
  guestBio?: string; // if guest is identified
  model: string;
  generatedAt: string;
  approved: boolean;
}
```

Show notes are stored in `episodes.ai_content.show_notes` (JSONB). The summary is also used to update `episodes.description` if it is currently empty (with `approved: false` until human review).

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/episodes/:episodeId/generate-show-notes` | Trigger show notes generation |
| GET | `/api/episodes/:episodeId/show-notes` | Get generated show notes |
| PATCH | `/api/episodes/:episodeId/show-notes` | Edit and approve show notes |

**Testing**:
- `Integration (mocked LLM): show notes contain summary, keyPoints, and keywords`
- `Integration: show notes stored in episode ai_content JSONB`
- `Integration: PATCH endpoint allows editing summary and setting approved=true`
- `Integration: if episode.description is empty, it is populated with summary (approved=false)`
- `Unit: show notes generation handles empty transcript gracefully`
- `Unit: keywords are deduplicated and lowercased`

---

#### 5.4 — AI Social Clip Identification

**What**: Analyse the transcript to identify the most shareable 30-90 second moments and generate clip metadata for social media export.

**Design**:

```typescript
// apps/worker/src/processors/clip-extract.ts
export interface ClipExtractPayload {
  organisationId: string;
  episodeId: string;
  transcriptId: string;
  maxClips?: number; // default 5
}

export interface SuggestedClip {
  startTimeMs: number;
  endTimeMs: number;
  title: string;
  description: string;
  shareabilityScore: number; // 0.0-1.0
  topicTags: string[];
  suggestedFormats: Array<"vertical" | "square" | "landscape">;
  suggestedPlatforms: Array<"tiktok" | "youtube_shorts" | "instagram_reels" | "twitter">;
  transcriptExcerpt: string;
}
```

The LLM identifies moments based on:
- Surprising or counterintuitive statements
- Concise, quotable insights
- Emotional peaks (humour, passion, controversy)
- Self-contained topics (do not require surrounding context)

Suggested clips are stored in `episodes.ai_content.clips[]` with `status: "suggested"`.

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/episodes/:episodeId/clips` | List suggested clips |
| PATCH | `/api/episodes/:episodeId/clips/:clipIndex` | Approve, reject, or edit a clip |
| POST | `/api/episodes/:episodeId/clips/:clipIndex/export` | Export clip to audio/video file (Phase 7) |

**Testing**:
- `Integration (mocked LLM): clip extraction returns 3-5 clips with valid time ranges`
- `Integration: clips stored in episode ai_content JSONB`
- `Integration: clips are sorted by shareability score descending`
- `Integration: PATCH allows approving/rejecting clips`
- `Unit: clip time ranges are within episode duration`
- `Unit: clip durations are between 15 and 120 seconds`
- `Unit: shareability scores are between 0.0 and 1.0`

---

## Phase 6: RSS Feed Generation and Distribution

### Purpose

Generate standards-compliant RSS feeds (RSS 2.0 + iTunes namespace + Podcast 2.0 namespace) and distribute them to podcast directories. After this phase, a published episode is accessible in Apple Podcasts, Spotify, and other directories through the generated RSS feed.

### Tasks

#### 6.1 — RSS Feed Builder

**What**: Generate valid RSS 2.0 feeds with iTunes and Podcast 2.0 namespace extensions from show and episode data.

**Design**:

```typescript
// apps/api/src/services/rss-service.ts
export interface RSSBuildOptions {
  showId: string;
  maxEpisodes?: number; // default: all published
  includeTranscripts?: boolean; // default: true
  includeChapters?: boolean; // default: true
  includePersons?: boolean; // default: true
  includeValue?: boolean; // default: true
}

export async function buildRSSFeed(options: RSSBuildOptions): Promise<string>;
```

The generated RSS XML includes:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<rss version="2.0"
     xmlns:itunes="http://www.itunes.com/dtds/podcast-1.0.dtd"
     xmlns:podcast="https://podcastindex.org/namespace/1.0"
     xmlns:atom="http://www.w3.org/2005/Atom"
     xmlns:content="http://purl.org/rss/1.0/modules/content/">
  <channel>
    <title>{show.title}</title>
    <atom:link href="{feedUrl}" rel="self" type="application/rss+xml"/>
    <link>{show.websiteUrl}</link>
    <description>{show.description}</description>
    <language>{show.language}</language>
    <itunes:category text="{show.itunes.category}">
      <itunes:category text="{show.itunes.subcategory}"/>
    </itunes:category>
    <itunes:author>{show.author}</itunes:author>
    <itunes:image href="{show.artworkUrl}"/>
    <itunes:explicit>{show.itunes.explicit}</itunes:explicit>
    <itunes:type>{show.itunes.type}</itunes:type>
    <podcast:guid>{show.podcast2.guid}</podcast:guid>
    <podcast:locked owner="{show.podcast2.lockOwner}">{show.podcast2.locked}</podcast:locked>
    <podcast:funding url="{url}">{message}</podcast:funding>
    <podcast:value ...>
      <podcast:valueRecipient .../>
    </podcast:value>
    <!-- items -->
    <item>
      <title>{episode.title}</title>
      <enclosure url="{enclosure.url}" length="{enclosure.length}" type="{enclosure.type}"/>
      <guid isPermaLink="false">{episode.id}</guid>
      <pubDate>{episode.publishedAt RFC2822}</pubDate>
      <itunes:duration>{HH:MM:SS}</itunes:duration>
      <itunes:episodeType>{episode.itunes.episodeType}</itunes:episodeType>
      <podcast:transcript url="{url}" type="text/vtt" language="{lang}"/>
      <podcast:chapters url="{url}" type="application/json+chapters"/>
      <podcast:soundbite startTime="{s}" duration="{d}">{title}</podcast:soundbite>
      <podcast:person role="{role}" group="{group}" img="{img}" href="{href}">{name}</podcast:person>
    </item>
  </channel>
</rss>
```

RSS feed is served at: `GET /feed/:showSlug.xml`

Feed is rebuilt and cached in Redis (TTL: 5 minutes) on access if stale, or eagerly rebuilt when an episode is published.

**Testing**:
- `Integration: generated RSS validates against PSP-1 podcast RSS specification`
- `Integration: feed contains correct iTunes namespace elements`
- `Integration: feed contains Podcast 2.0 transcript, chapters, person, soundbite tags`
- `Integration: feed contains atom:link self-reference`
- `Integration: episodes ordered by pubDate descending`
- `Integration: unpublished episodes excluded from feed`
- `Integration: feed cached in Redis, served from cache on subsequent requests`
- `Integration: publishing an episode invalidates feed cache`
- `Fixture: compare generated RSS against known-good fixture feed`
- `Unit: pubDate formatted in RFC 2822 format`
- `Unit: duration formatted as HH:MM:SS`
- `Unit: CDATA wrapping for HTML description content`

---

#### 6.2 — Distribution Target Management

**What**: Track podcast directory submissions (Apple Podcasts, Spotify, YouTube) and provide one-click submission guidance.

**Design**:

Distribution targets are stored in `shows.distribution` JSONB:

```typescript
// packages/shared/src/types/distribution.ts
export interface DistributionTarget {
  platform: "apple_podcasts" | "spotify" | "youtube" | "amazon_music" | "google_podcasts" | "podcast_index";
  status: "not_submitted" | "pending" | "approved" | "rejected";
  platformId?: string;
  externalUrl?: string;
  submittedAt?: string;
  approvedAt?: string;
  notes?: string;
}

export interface SubmitToDirectoryRequest {
  platform: string;
}

export interface SubmitToDirectoryResponse {
  platform: string;
  feedUrl: string;
  submissionUrl: string; // URL to paste the feed into
  instructions: string; // human-readable steps
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/shows/:showId/distribution` | List all distribution targets with status |
| POST | `/api/shows/:showId/distribution/submit` | Get submission instructions for a platform |
| PATCH | `/api/shows/:showId/distribution/:platform` | Update platform status (after manual verification) |

Podcast Index submission is automated via their API (using HMAC-SHA1 authentication per the Podcast Index API spec).

**Testing**:
- `Integration: GET distribution returns all platforms with correct default statuses`
- `Integration: POST submit returns feed URL and platform-specific submission instructions`
- `Integration: PATCH updates distribution JSONB in show record`
- `Integration (mocked): Podcast Index API submission sends correct HMAC-signed request`
- `Unit: submission instructions are correct for each supported platform`
- `Unit: feed URL uses the correct show slug`

---

## Phase 7: Frontend — Dashboard and Episode Management

### Purpose

Build the Next.js web application providing a dashboard for managing shows, episodes, and production workflows. After this phase, users have a complete web UI for the core podcast production workflow: creating shows, uploading episodes, monitoring processing, reviewing AI-generated content, and publishing.

### Tasks

#### 7.1 — Authentication UI and Layout

**What**: Implement the login, registration, and dashboard layout pages using Next.js App Router and shadcn/ui.

**Design**:

Page structure:
```
src/app/
├── (auth)/
│   ├── login/page.tsx
│   ├── register/page.tsx
│   └── layout.tsx              # Centered card layout
├── (dashboard)/
│   ├── layout.tsx              # Sidebar nav, header with user menu
│   ├── page.tsx                # Dashboard home (recent episodes, quick stats)
│   ├── shows/
│   │   ├── page.tsx            # Show list
│   │   ├── new/page.tsx        # Create show form
│   │   └── [showId]/
│   │       ├── page.tsx        # Show detail
│   │       ├── settings/page.tsx
│   │       └── episodes/
│   │           ├── page.tsx    # Episode list
│   │           ├── new/page.tsx
│   │           └── [episodeId]/
│   │               ├── page.tsx        # Episode detail/editor
│   │               ├── process/page.tsx # Processing status
│   │               └── publish/page.tsx # Pre-publish review
│   ├── analytics/page.tsx
│   └── settings/page.tsx
```

Dashboard layout sidebar navigation:
```typescript
const NAV_ITEMS = [
  { label: "Dashboard", href: "/", icon: Home },
  { label: "Shows", href: "/shows", icon: Mic },
  { label: "Analytics", href: "/analytics", icon: BarChart },
  { label: "Settings", href: "/settings", icon: Settings },
];
```

**Testing**:
- `E2E (Playwright): register new account -> redirected to dashboard`
- `E2E: login with valid credentials -> dashboard loads with sidebar`
- `E2E: login with invalid credentials -> error message displayed`
- `E2E: sidebar navigation works, active state highlights current page`
- `E2E: logout -> redirected to login page`
- `Unit (React): layout component renders sidebar and content area`

---

#### 7.2 — Show Management UI

**What**: Build pages for creating, listing, and editing shows with iTunes category selection and metadata forms.

**Design**:

Create show form fields:
```typescript
interface CreateShowForm {
  title: string;           // required
  description: string;     // textarea, optional
  language: string;        // select, BCP 47 codes
  author: string;          // optional
  artwork: File;           // image upload, 3000x3000 min
  itunesCategory: string;  // select from Apple's category list
  itunesSubcategory: string;
  explicit: boolean;       // toggle
  showType: "episodic" | "serial";
}
```

iTunes categories constant (from Apple Podcasts spec):
```typescript
// packages/shared/src/constants/itunes-categories.ts
export const ITUNES_CATEGORIES: Record<string, string[]> = {
  "Arts": ["Books", "Design", "Fashion & Beauty", "Food", "Performing Arts", "Visual Arts"],
  "Business": ["Careers", "Entrepreneurship", "Investing", "Management", "Marketing", "Non-Profit"],
  "Comedy": ["Comedy Interviews", "Improv", "Stand-Up"],
  "Education": ["Courses", "How To", "Language Learning", "Self-Improvement"],
  "Technology": [],
  // ... full list per Apple spec
};
```

**Testing**:
- `E2E: create show with all fields -> show appears in list`
- `E2E: artwork upload validates minimum dimensions`
- `E2E: iTunes category select populates subcategories`
- `E2E: edit show -> changes reflected in show detail page`
- `E2E: archive show -> removed from active list, visible in archived filter`
- `Unit: category select component renders all Apple Podcasts categories`

---

#### 7.3 — Episode Editor and Upload UI

**What**: Build the episode creation page with multi-track audio upload, drag-and-drop, and processing status monitoring.

**Design**:

Episode editor page sections:
1. **Metadata**: Title, description, episode/season number, artwork
2. **Audio Tracks**: Multi-file drag-and-drop upload with track labelling (Host, Guest 1, etc.)
3. **Processing**: One-click post-production button, real-time status via WebSocket/SSE
4. **AI Content**: Tabs for transcript, chapters, show notes, suggested clips (read-only until generated)
5. **Publish**: Pre-publish checklist, schedule date picker, publish button

Audio upload component:
```typescript
interface AudioUploadProps {
  episodeId: string;
  onUploadComplete: (audioFile: AudioFileResponse) => void;
}
// Uses pre-signed URL flow:
// 1. Request upload URL from API
// 2. Upload directly to S3 with progress bar
// 3. Confirm upload
// 4. Display track with waveform preview
```

Processing status uses Server-Sent Events (SSE):
```typescript
// apps/api/src/routes/episodes/process-status.ts
// GET /api/episodes/:episodeId/process/stream
// Content-Type: text/event-stream
// data: {"step":"noise_removal","status":"running","progress":45}
// data: {"step":"noise_removal","status":"complete"}
// data: {"step":"loudness_norm","status":"running","progress":10}
```

**Testing**:
- `E2E: drag-and-drop audio file -> upload progress shown, track appears in list`
- `E2E: label tracks as Host/Guest -> labels persisted`
- `E2E: click "Process" -> processing status updates in real-time`
- `E2E: after processing complete, transcript tab populates`
- `E2E: show notes and chapters tabs show AI-generated content`
- `E2E: clip suggestions displayed with approve/reject buttons`
- `Unit: upload progress bar updates correctly`
- `Unit: SSE client reconnects on connection drop`

---

#### 7.4 — Episode Publishing Flow

**What**: Build the pre-publish review page with checklist validation and the publish action.

**Design**:

Pre-publish checklist:
```typescript
interface PublishChecklist {
  hasTitle: boolean;
  hasDescription: boolean;
  hasAudioFile: boolean;
  hasEnclosure: boolean; // processed MP3 available
  hasTranscript: boolean;
  hasChapters: boolean;
  hasShowNotes: boolean;
  isReviewed: boolean; // show notes approved
  artworkSet: boolean;
}
```

All items except `hasTranscript`, `hasChapters`, and `hasShowNotes` are required for publishing. The optional items show warnings rather than blocking publish.

**Testing**:
- `E2E: publish checklist shows red/green status for each item`
- `E2E: cannot publish without required items (button disabled, tooltip explains why)`
- `E2E: clicking publish -> episode status changes, appears in RSS feed`
- `E2E: schedule publish -> scheduled date saved, episode publishes at scheduled time`
- `Unit: checklist component correctly derives status from episode data`

---

## Phase 8: Public Podcast Website

### Purpose

Generate a public-facing podcast website for each show, with episode pages, embedded audio player, transcript display, and Schema.org structured data for SEO. After this phase, every show has a browsable website that serves as both a listener destination and an SEO asset.

### Tasks

#### 8.1 — Public Show and Episode Pages

**What**: Build server-rendered public pages for shows and episodes with audio player, transcript, and chapter navigation.

**Design**:

Routes:
```
src/app/(public)/
├── [showSlug]/
│   ├── page.tsx              # Show landing page with episode list
│   └── [episodeSlug]/
│       └── page.tsx          # Episode page with player, transcript, chapters
```

Episode page components:
- **Audio player**: wavesurfer.js with waveform display, play/pause, seek, speed control
- **Chapter navigator**: clickable chapter list that seeks the player to the chapter start time
- **Transcript viewer**: scrolling transcript that highlights current segment during playback
- **Show notes**: rendered markdown from AI-generated show notes
- **Guest cards**: profile cards for each person tagged on the episode

Schema.org structured data (per `standards.md`):
```typescript
// Embedded in <head> as JSON-LD
const episodeSchema = {
  "@context": "https://schema.org",
  "@type": "PodcastEpisode",
  "name": episode.title,
  "description": episode.description,
  "datePublished": episode.publishedAt,
  "duration": `PT${episode.durationSeconds}S`,
  "associatedMedia": {
    "@type": "MediaObject",
    "contentUrl": episode.enclosure.url,
    "encodingFormat": episode.enclosure.type,
  },
  "partOfSeries": {
    "@type": "PodcastSeries",
    "name": show.title,
    "url": showUrl,
  },
};
```

**Testing**:
- `E2E: public show page loads with episode list, artwork, and description`
- `E2E: public episode page loads with audio player, transcript, and chapters`
- `E2E: clicking chapter seeks player to correct time`
- `E2E: transcript highlights sync with audio playback`
- `Integration: Schema.org JSON-LD is valid and present in page source`
- `Integration: OG meta tags set correctly for social sharing`
- `Unit: audio player component renders waveform from audio URL`

---

#### 8.2 — RSS Feed Endpoint and Sitemap

**What**: Serve the RSS feed at a public URL and generate a sitemap for search engine indexing.

**Design**:

Public feed URL: `GET /:showSlug/feed.xml`

Sitemap: `GET /sitemap.xml` — dynamically generated, includes all show and episode pages.

```xml
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  <url>
    <loc>https://example.com/the-dev-podcast</loc>
    <lastmod>2026-05-25</lastmod>
    <changefreq>weekly</changefreq>
  </url>
  <url>
    <loc>https://example.com/the-dev-podcast/episode-42-ai-in-production</loc>
    <lastmod>2026-05-20</lastmod>
  </url>
</urlset>
```

**Testing**:
- `Integration: GET /:showSlug/feed.xml returns valid RSS with correct Content-Type (application/rss+xml)`
- `Integration: GET /sitemap.xml includes all published shows and episodes`
- `Integration: sitemap only includes published episodes (not drafts)`
- `Unit: sitemap lastmod uses episode publishedAt date`

---

## Phase 9: Analytics and Listener Tracking

### Purpose

Implement download/stream tracking, listener analytics, and a dashboard for viewing episode performance. Analytics events use privacy-respecting data collection (hashed IPs, no PII storage) compliant with GDPR Article 32.

### Tasks

#### 9.1 — Analytics Event Collection

**What**: Track episode downloads and streams via the audio file endpoint, storing privacy-respecting analytics events.

**Design**:

Analytics are captured when the audio enclosure URL is requested. The enclosure URL is a redirect endpoint:

```
GET /api/track/:episodeId/:filename
-> 302 redirect to actual S3 CDN URL
-> analytics event recorded asynchronously
```

```typescript
// packages/shared/src/types/analytics.ts
export interface AnalyticsEvent {
  episodeId: string;
  showId: string;
  eventType: "download" | "stream" | "partial_play";
  countryCode?: string; // ISO 3166-1 alpha-2, from IP geolocation
  region?: string;
  city?: string;
  deviceType?: string; // parsed from User-Agent
  appName?: string; // apple_podcasts, spotify, overcast (from User-Agent)
  referrer?: string;
  ipHash: string; // SHA-256(ip + daily_salt) — rotated daily for GDPR
}
```

IP hashing uses a daily-rotated salt to prevent long-term tracking while allowing same-day deduplication per GDPR Article 32 guidance.

User-Agent parsing to identify podcast apps:
```typescript
const APP_PATTERNS: Record<string, RegExp> = {
  apple_podcasts: /AppleCoreMedia|Podcasts\//,
  spotify: /Spotify\//,
  overcast: /Overcast\//,
  pocket_casts: /PocketCasts\//,
  castro: /Castro\//,
  google_podcasts: /GoogleChirp/,
};
```

**Testing**:
- `Integration: GET /api/track/:episodeId/episode.mp3 returns 302 redirect`
- `Integration: analytics event created with correct episode and show IDs`
- `Integration: IP is hashed, not stored raw`
- `Integration: IP hash uses daily-rotated salt (different hash for same IP on different days)`
- `Integration: User-Agent correctly parsed to identify podcast app`
- `Integration: duplicate download from same IP hash within 24h is deduplicated`
- `Unit: app detection regex matches known User-Agent strings`
- `Unit: country code derived from IP geolocation (mocked MaxMind)`

---

#### 9.2 — Analytics Aggregation and Dashboard API

**What**: Aggregate raw analytics events into daily rollups and provide API endpoints for the analytics dashboard.

**Design**:

Daily aggregation job (runs nightly via BullMQ scheduled job):
```typescript
// apps/worker/src/processors/analytics-rollup.ts
export interface DailyStats {
  downloads: number;
  streams: number;
  uniqueListeners: number;
  avgListenPct: number;
  countries: Record<string, number>; // { "US": 250, "GB": 80 }
  apps: Record<string, number>; // { "apple_podcasts": 200, "spotify": 150 }
  devices: Record<string, number>; // { "mobile": 300, "desktop": 100 }
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/shows/:showId/analytics` | Show-level analytics (date range) |
| GET | `/api/episodes/:episodeId/analytics` | Episode-level analytics (date range) |
| GET | `/api/shows/:showId/analytics/top-episodes` | Top episodes by downloads |

Query parameters: `startDate`, `endDate`, `granularity` (daily, weekly, monthly).

**Testing**:
- `Integration: aggregation job correctly sums events into daily rollup`
- `Integration: unique listeners counted by distinct IP hash per day`
- `Integration: GET analytics endpoint returns correct totals for date range`
- `Integration: top-episodes endpoint returns episodes sorted by download count`
- `Unit: date range validation rejects future dates and ranges > 1 year`
- `Unit: granularity "weekly" groups daily stats into ISO weeks`

---

## Phase 10: API Tokens, Webhooks, and External Integrations

### Purpose

Enable third-party integrations via API tokens, webhook notifications, and the public REST API. After this phase, external tools can programmatically manage shows and episodes, and events (episode published, processing complete) trigger webhook notifications.

### Tasks

#### 10.1 — API Token Management

**What**: Implement personal API token creation, scoping, and authentication for the public API.

**Design**:

```typescript
// packages/shared/src/types/api-token.ts
export type ApiScope = "shows:read" | "shows:write" | "episodes:read" | "episodes:write" |
  "audio:read" | "audio:write" | "analytics:read" | "webhooks:manage";

export interface CreateApiTokenRequest {
  name: string;
  scopes: ApiScope[];
  expiresIn?: number; // days, default: never
}

export interface ApiTokenResponse {
  id: string;
  name: string;
  scopes: ApiScope[];
  token: string; // only returned once, on creation
  expiresAt: string | null;
  lastUsedAt: string | null;
  createdAt: string;
}
```

Token format: `ppp_` prefix + 40 random hex characters. Stored as SHA-256 hash in database.

API authentication: `Authorization: Bearer ppp_abc123...` header.

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/tokens` | Create API token |
| GET | `/api/tokens` | List tokens (without token value) |
| DELETE | `/api/tokens/:tokenId` | Revoke token |

**Testing**:
- `Integration: create token -> token returned, hash stored in database`
- `Integration: authenticate with valid token -> request succeeds`
- `Integration: authenticate with revoked token -> 401`
- `Integration: token with shows:read scope -> can GET shows, cannot POST shows`
- `Integration: token last_used_at updated on each authenticated request`
- `Unit: token format matches ppp_ prefix pattern`
- `Unit: token hash is SHA-256 of raw token`

---

#### 10.2 — Webhook System

**What**: Implement webhook registration and reliable event delivery with retry logic.

**Design**:

```typescript
// packages/shared/src/types/webhook.ts
export type WebhookEvent =
  | "episode.published"
  | "episode.updated"
  | "episode.archived"
  | "processing.started"
  | "processing.completed"
  | "processing.failed"
  | "transcript.generated"
  | "show_notes.generated";

export interface CreateWebhookRequest {
  url: string;
  events: WebhookEvent[];
  secret: string; // for HMAC signature verification
}

export interface WebhookDelivery {
  event: WebhookEvent;
  payload: Record<string, unknown>;
  timestamp: string;
  signature: string; // HMAC-SHA256(secret, JSON.stringify(payload))
}
```

Webhook delivery:
1. Event occurs (e.g., episode published)
2. BullMQ job enqueued for each registered webhook matching the event
3. HTTP POST to webhook URL with JSON payload and `X-PPP-Signature` header
4. Retry 3 times with exponential backoff (1s, 5s, 25s) on non-2xx responses
5. After max retries, webhook marked as `failing`; after 10 consecutive failures, `disabled`

**Testing**:
- `Integration: register webhook -> POST /api/webhooks returns 201`
- `Integration: publishing episode triggers webhook delivery to registered URL`
- `Integration (mocked HTTP): webhook payload contains correct event data and HMAC signature`
- `Integration (mocked HTTP): failed delivery retries with exponential backoff`
- `Integration (mocked HTTP): 10 consecutive failures disables webhook`
- `Unit: HMAC-SHA256 signature matches expected value`
- `Unit: webhook URL validation rejects non-HTTPS URLs in production`

---

## Phase 11: Podcast 2.0 Advanced Features

### Purpose

Implement advanced Podcast 2.0 namespace features: Value4Value payment tags, live item support, and enhanced metadata. After this phase, shows fully support the modern podcasting ecosystem including Lightning Network micropayments and live streaming metadata.

### Tasks

#### 11.1 — Value4Value Payment Configuration

**What**: Enable show owners to configure Value4Value payment splits using the `<podcast:value>` tag (Bitcoin Lightning Network per the Podcast 2.0 namespace spec).

**Design**:

Value configuration stored in `shows.podcast2_meta.value`:

```typescript
// packages/shared/src/types/value.ts
export interface ValueConfig {
  type: "lightning";
  method: "keysend";
  suggested?: number; // suggested sats per minute
  recipients: Array<{
    name: string;
    type: "wallet" | "node";
    address: string;
    customKey?: number;
    customValue?: string;
    split: number; // percentage, all must sum to 100
    fee: boolean;
  }>;
}
```

| Method | Path | Description |
|--------|------|-------------|
| GET | `/api/shows/:showId/value` | Get current Value4Value config |
| PUT | `/api/shows/:showId/value` | Set Value4Value config |
| DELETE | `/api/shows/:showId/value` | Remove Value4Value config |

Validation: recipient splits must sum to 100. At least one recipient with `fee: false`.

**Testing**:
- `Integration: PUT value config -> stored in podcast2_meta, appears in RSS feed`
- `Integration: RSS feed contains valid <podcast:value> and <podcast:valueRecipient> tags`
- `Integration: splits not summing to 100 returns 422`
- `Unit: value config validation accepts valid Lightning keysend addresses`

---

#### 11.2 — Podcast 2.0 Metadata Enhancements

**What**: Support location tags, podroll, season metadata, and alternate enclosures per the Podcast 2.0 namespace spec.

**Design**:

Additional metadata stored in JSONB columns:

```typescript
export interface PodcastLocation {
  geo?: string; // geo:37.7749,-122.4194
  osm?: string; // R123456
  name: string;
}

export interface AlternateEnclosure {
  type: string; // audio/opus, video/mp4
  length: number;
  bitrate?: number;
  height?: number; // for video
  sources: Array<{ uri: string }>;
}
```

| Method | Path | Description |
|--------|------|-------------|
| PATCH | `/api/episodes/:id/location` | Set episode location |
| PUT | `/api/episodes/:id/alternate-enclosures` | Set alternate enclosures |
| PUT | `/api/shows/:id/podroll` | Set show podroll |

**Testing**:
- `Integration: location tag appears in RSS <podcast:location> element`
- `Integration: alternate enclosures appear as <podcast:alternateEnclosure> in RSS`
- `Integration: podroll appears as <podcast:podroll> in show RSS`
- `Unit: geo URI validation accepts valid coordinates, rejects invalid`

---

## Phase 12: Self-Hosted Deployment and Production Hardening

### Purpose

Package the platform for self-hosted deployment with Docker, add production security hardening (rate limiting, OWASP API Security, CORS), and implement backup/restore. After this phase, the platform can be deployed to any Docker-capable server with a single `docker compose up`.

### Tasks

#### 12.1 — Production Docker Configuration

**What**: Create optimised multi-stage Docker builds for the API, worker, and web applications, with a production Docker Compose file.

**Design**:

```dockerfile
# Dockerfile (multi-stage)
FROM node:22-alpine AS base
RUN corepack enable pnpm

FROM base AS deps
WORKDIR /app
COPY package.json pnpm-lock.yaml pnpm-workspace.yaml ./
COPY packages/db/package.json packages/db/
COPY packages/shared/package.json packages/shared/
COPY apps/api/package.json apps/api/
COPY apps/worker/package.json apps/worker/
RUN pnpm install --frozen-lockfile

FROM base AS builder
WORKDIR /app
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm build

FROM base AS api
WORKDIR /app
COPY --from=builder /app/apps/api/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
EXPOSE 4000
CMD ["node", "dist/server.js"]

FROM base AS worker
WORKDIR /app
COPY --from=builder /app/apps/worker/dist ./dist
COPY --from=deps /app/node_modules ./node_modules
# FFmpeg required for audio processing
RUN apk add --no-cache ffmpeg
CMD ["node", "dist/worker.js"]
```

`docker-compose.production.yml`:
```yaml
services:
  api:
    build: { context: ., target: api }
    environment:
      DATABASE_URL: postgresql://ppp:${DB_PASSWORD}@postgres:5432/ppp
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
    ports: ["4000:4000"]
    depends_on: [postgres, redis, minio]
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "wget", "-q", "--spider", "http://localhost:4000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
  worker:
    build: { context: ., target: worker }
    environment:
      DATABASE_URL: postgresql://ppp:${DB_PASSWORD}@postgres:5432/ppp
      REDIS_URL: redis://redis:6379
      S3_ENDPOINT: http://minio:9000
    depends_on: [postgres, redis, minio]
    restart: unless-stopped
  web:
    build: { context: ., dockerfile: apps/web/Dockerfile }
    ports: ["3000:3000"]
    depends_on: [api]
    restart: unless-stopped
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ppp
      POSTGRES_USER: ppp
      POSTGRES_PASSWORD: ${DB_PASSWORD}
    volumes: ["pgdata:/var/lib/postgresql/data"]
    restart: unless-stopped
  redis:
    image: redis:7-alpine
    command: redis-server --appendonly yes
    volumes: ["redisdata:/data"]
    restart: unless-stopped
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    environment:
      MINIO_ROOT_USER: ${S3_ACCESS_KEY}
      MINIO_ROOT_PASSWORD: ${S3_SECRET_KEY}
    volumes: ["miniodata:/data"]
    restart: unless-stopped
volumes:
  pgdata:
  redisdata:
  miniodata:
```

**Testing**:
- `Integration: docker compose -f docker-compose.production.yml build succeeds`
- `Integration: docker compose up starts all services, health checks pass`
- `Integration: API responds on port 4000, web responds on port 3000`
- `E2E: full workflow (register, create show, upload audio, process, publish) works in Docker`
- `Unit: Docker image size is under 500MB for API, under 800MB for worker (includes FFmpeg)`

---

#### 12.2 — Security Hardening

**What**: Implement rate limiting, CORS configuration, input validation, and OWASP API Security Top 10 protections.

**Design**:

Security measures per OWASP API Security Top 10 (2023):

| OWASP Risk | Mitigation |
|------------|------------|
| API1: Broken Object Level Authorization | Organisation-scoped queries with tenant middleware; all queries include `organisation_id` filter |
| API2: Broken Authentication | Argon2id password hashing; session expiration (7 days); rate-limited login (5 attempts/15 min) |
| API3: Broken Object Property Level Authorization | Zod schema validation strips unknown properties; response serialisation excludes internal fields |
| API4: Unrestricted Resource Consumption | Rate limiting (100 req/min per user); file upload size limits (500MB); audio processing queue limits |
| API5: Broken Function Level Authorization | Role-based access control (owner, admin, editor, member); admin-only routes for org settings |
| API7: Server Side Request Forgery | Webhook URLs validated against allowlist; no internal network URLs; URL scheme restricted to HTTPS |

Rate limiting configuration:
```typescript
// apps/api/src/middleware/rate-limit.ts
import rateLimit from "@fastify/rate-limit";

export const rateLimitConfig = {
  global: { max: 100, timeWindow: "1 minute" },
  auth: { max: 5, timeWindow: "15 minutes" }, // login/register
  upload: { max: 10, timeWindow: "1 hour" },
  aiGeneration: { max: 20, timeWindow: "1 hour" },
};
```

**Testing**:
- `Integration: rate-limited endpoint returns 429 after exceeding limit`
- `Integration: tenant isolation — user A cannot access user B's shows (returns 404, not 403)`
- `Integration: unknown properties in request body are stripped (not passed to database)`
- `Integration: webhook URL validation rejects http://, localhost, and RFC 1918 addresses`
- `Integration: file upload exceeding size limit returns 413`
- `Unit: CORS configuration only allows configured origins`
- `Unit: session tokens expire after configured TTL`

---

#### 12.3 — Database Backup and Restore

**What**: Implement automated PostgreSQL backup and restore utilities for self-hosted deployments.

**Design**:

```typescript
// apps/api/src/services/backup-service.ts
export interface BackupService {
  createBackup(): Promise<{ filename: string; size: number; createdAt: string }>;
  restoreBackup(filename: string): Promise<void>;
  listBackups(): Promise<Array<{ filename: string; size: number; createdAt: string }>>;
  deleteBackup(filename: string): Promise<void>;
}
```

Backup implementation:
- Uses `pg_dump` for database backup (custom format for compression)
- Backup files stored in S3 under `backups/` prefix
- Scheduled backup via BullMQ repeatable job (daily at 02:00 UTC)
- Retention: keep last 30 daily backups, delete older

| Method | Path | Description |
|--------|------|-------------|
| POST | `/api/admin/backups` | Create manual backup |
| GET | `/api/admin/backups` | List available backups |
| POST | `/api/admin/backups/:filename/restore` | Restore from backup |

Admin-only endpoints (requires `owner` role).

**Testing**:
- `Integration: create backup -> backup file exists in S3 with correct format`
- `Integration: restore backup -> database state matches backup contents`
- `Integration: list backups returns correct metadata`
- `Integration: backup retention deletes backups older than 30 days`
- `Unit: backup filename format includes ISO date and organisationId`
- `Unit: non-owner users receive 403 on admin endpoints`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation                    ─── required by everything
    │
Phase 2: Show & Episode CRUD          ─── requires Phase 1
    │
Phase 3: Audio File Management         ─── requires Phase 2
    │
Phase 4: Audio Processing Pipeline     ─── requires Phase 3
    │
Phase 5: Transcription & AI Content   ─── requires Phase 4
    │       │
    │       ├── Phase 6: RSS & Distribution    ─── requires Phase 2 + Phase 5
    │       │
    │       └── Phase 9: Analytics             ─── requires Phase 2 (can parallel with 6, 7, 8)
    │
Phase 7: Frontend Dashboard           ─── requires Phase 2 (can parallel with 5, 6)
    │
Phase 8: Public Podcast Website        ─── requires Phase 6 + Phase 7
    │
Phase 10: API Tokens & Webhooks       ─── requires Phase 2 (can parallel with 5-9)
    │
Phase 11: Podcast 2.0 Advanced        ─── requires Phase 6
    │
Phase 12: Deployment & Hardening      ─── requires all previous phases
```

### Parallelism Opportunities

- **Phases 6, 7, 9, 10** can be developed concurrently after Phase 5 (or Phase 4 for 7 and 10)
- **Phase 8** depends on both Phase 6 (RSS) and Phase 7 (frontend framework)
- **Phase 11** depends on Phase 6 (RSS generation) but is otherwise independent
- **Phase 12** is the final integration phase and requires all other phases

---

## Definition of Done (per phase)

1. All tasks implemented with code matching the design specifications.
2. All unit tests pass (`pnpm test`).
3. All integration tests pass (with Docker services running).
4. Biome linting and formatting pass (`pnpm lint`).
5. TypeScript compilation succeeds with no errors (`pnpm build`).
6. Docker build succeeds for all affected services.
7. Feature works end-to-end (manual verification or E2E test).
8. Database migrations created and applied cleanly.
9. New API endpoints appear in OpenAPI spec at `/docs`.
10. New environment variables documented in `.env.example`.
11. No regressions — existing tests continue to pass.
12. Processing jobs (if any) complete successfully with test fixtures.
