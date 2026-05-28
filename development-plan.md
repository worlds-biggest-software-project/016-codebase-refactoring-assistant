# Codebase Refactoring Assistant — Phased Development Plan

> Project: 016-codebase-refactoring-assistant · Created: 2026-05-11
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Primary Language | TypeScript (Node.js 22 LTS) | Polyglot analysis tooling ecosystem strongest in TS/JS; tree-sitter has first-class Node bindings; aligns with target user base of platform/DevEx teams |
| Runtime | Node.js 22 with native ES modules | Native `fetch`, `--watch`, and performance improvements; ESM-first for tree-shaking |
| Web Framework | Fastify 5 | Lower overhead than Express; JSON Schema-based validation; OpenAPI 3.1 auto-generation via `@fastify/swagger` |
| Frontend | Next.js 15 (App Router) + React 19 | SSR for dashboard performance; RSC for data-heavy views (hotspot maps, trend charts); shadcn/ui for component library |
| Database | PostgreSQL 16 | JSONB + GIN indexes for polyglot metrics; recursive CTEs for CWE hierarchy; partitioning for event/audit data |
| Data Model | Hybrid Relational + JSONB (Data Model Suggestion 3) with audit log | Best balance of query performance on core fields (typed columns) and flexibility for language-specific metrics (JSONB); 14 tables vs 33 normalized; audit log covers NIST SSDF without full event-sourcing complexity |
| ORM / Query Builder | Drizzle ORM | Type-safe schema definitions; JSONB column support; migration generation; lighter than Prisma for JSONB-heavy schemas |
| AST Parsing | tree-sitter (via `tree-sitter` npm + `tree-sitter-language-pack`) | 305+ language grammars; CST preserves whitespace/comments for format-preserving refactoring; incremental parsing for editor integration |
| Complexity Metrics | Custom module on tree-sitter CSTs | Cyclomatic and cognitive complexity computed from CST traversal; no dependency on language-specific tooling |
| LLM Integration | Anthropic Claude API (claude-sonnet-4-20250514) via `@anthropic-ai/sdk` | Prompt caching for repeated codebase context; 200K context for large file analysis; tool use for structured refactoring output |
| VCS Integration | `simple-git` npm package + GitHub/GitLab REST APIs | `simple-git` for local git history ingestion; platform APIs for PR creation and webhook handling |
| Task Queue | BullMQ (Redis-backed) | Analysis and transformation jobs run asynchronously; retry/backoff for LLM API calls; rate limiting |
| Testing | Vitest + Playwright | Vitest for unit/integration; Playwright for E2E dashboard tests |
| CI/CD | GitHub Actions | Matrix builds; PR decoration for quality gate results |
| Code Analysis Output | SARIF 2.1.0 | GitHub Advanced Security integration; standard interchange format for findings |
| Authentication | OAuth 2.0 (Authorization Code + PKCE) via GitHub/GitLab | Users authenticate with their VCS provider; no separate credential management |
| API Specification | OpenAPI 3.1 (auto-generated from Fastify schemas) | Machine-readable API contract; client SDK generation |
| Containerization | Docker (multi-stage) + Docker Compose for local dev | Reproducible builds; single-command local environment |
| Monorepo | Turborepo | Shared packages (types, utils) between API and frontend; incremental builds |

### Project Structure

```
codebase-refactoring-assistant/
├── turbo.json
├── package.json
├── docker-compose.yml
├── Dockerfile
├── .github/
│   └── workflows/
│       ├── ci.yml
│       └── release.yml
├── packages/
│   ├── types/                    # Shared TypeScript types
│   │   ├── src/
│   │   │   ├── domain/
│   │   │   │   ├── repository.ts
│   │   │   │   ├── source-file.ts
│   │   │   │   ├── finding.ts
│   │   │   │   ├── hotspot.ts
│   │   │   │   ├── transformation.ts
│   │   │   │   ├── campaign.ts
│   │   │   │   ├── recipe.ts
│   │   │   │   ├── debt-budget.ts
│   │   │   │   └── quality-gate.ts
│   │   │   ├── api/
│   │   │   │   ├── requests.ts
│   │   │   │   └── responses.ts
│   │   │   └── index.ts
│   │   └── package.json
│   └── utils/                    # Shared utilities
│       ├── src/
│       │   ├── sarif.ts
│       │   ├── complexity.ts
│       │   ├── churn.ts
│       │   └── index.ts
│       └── package.json
├── apps/
│   ├── api/                      # Fastify REST API
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── config.ts
│   │   │   ├── db/
│   │   │   │   ├── schema.ts          # Drizzle schema
│   │   │   │   ├── migrations/
│   │   │   │   └── seed.ts
│   │   │   ├── routes/
│   │   │   │   ├── repositories.ts
│   │   │   │   ├── analysis.ts
│   │   │   │   ├── findings.ts
│   │   │   │   ├── hotspots.ts
│   │   │   │   ├── transformations.ts
│   │   │   │   ├── campaigns.ts
│   │   │   │   ├── recipes.ts
│   │   │   │   ├── debt-budgets.ts
│   │   │   │   ├── quality-gates.ts
│   │   │   │   └── webhooks.ts
│   │   │   ├── services/
│   │   │   │   ├── vcs-ingestion.ts
│   │   │   │   ├── analysis-engine.ts
│   │   │   │   ├── complexity-calculator.ts
│   │   │   │   ├── churn-calculator.ts
│   │   │   │   ├── hotspot-detector.ts
│   │   │   │   ├── finding-detector.ts
│   │   │   │   ├── llm-refactoring.ts
│   │   │   │   ├── test-generator.ts
│   │   │   │   ├── test-executor.ts
│   │   │   │   ├── transformation-engine.ts
│   │   │   │   ├── sarif-exporter.ts
│   │   │   │   ├── pr-manager.ts
│   │   │   │   └── debt-budget-planner.ts
│   │   │   ├── workers/
│   │   │   │   ├── analysis.worker.ts
│   │   │   │   ├── transformation.worker.ts
│   │   │   │   └── vcs-ingestion.worker.ts
│   │   │   ├── auth/
│   │   │   │   ├── oauth.ts
│   │   │   │   └── middleware.ts
│   │   │   └── plugins/
│   │   │       ├── swagger.ts
│   │   │       └── auth.ts
│   │   ├── test/
│   │   └── package.json
│   └── web/                      # Next.js frontend
│       ├── src/
│       │   ├── app/
│       │   │   ├── layout.tsx
│       │   │   ├── page.tsx
│       │   │   ├── (auth)/
│       │   │   ├── (dashboard)/
│       │   │   │   ├── repositories/
│       │   │   │   ├── hotspots/
│       │   │   │   ├── findings/
│       │   │   │   ├── transformations/
│       │   │   │   ├── campaigns/
│       │   │   │   ├── debt-budget/
│       │   │   │   └── settings/
│       │   │   └── api/
│       │   ├── components/
│       │   │   ├── hotspot-map.tsx
│       │   │   ├── debt-trend-chart.tsx
│       │   │   ├── finding-list.tsx
│       │   │   ├── diff-viewer.tsx
│       │   │   └── quality-gate-badge.tsx
│       │   └── lib/
│       │       ├── api-client.ts
│       │       └── hooks/
│       ├── test/
│       └── package.json
└── scripts/
    ├── seed-cwe.ts
    ├── seed-owasp.ts
    └── seed-rules.ts
```

---

## Phase 1: Foundation — Project Scaffold, Database, and Core Types

### Purpose
Establish the monorepo structure, database schema, shared type definitions, and development environment so all subsequent phases build on a stable, tested foundation.

### Tasks

#### 1.1 — Monorepo Scaffold with Turborepo
**What**: Initialize the Turborepo monorepo with `packages/types`, `packages/utils`, `apps/api`, and `apps/web` workspaces.
**Design**:
```typescript
// turbo.json
{
  "tasks": {
    "build": { "dependsOn": ["^build"], "outputs": ["dist/**"] },
    "test": { "dependsOn": ["^build"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "typecheck": {}
  }
}

// Root package.json scripts
{
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "test": "turbo test",
    "lint": "turbo lint",
    "typecheck": "turbo typecheck",
    "db:migrate": "cd apps/api && drizzle-kit migrate",
    "db:generate": "cd apps/api && drizzle-kit generate",
    "db:seed": "cd apps/api && tsx src/db/seed.ts"
  }
}
```
**Testing**:
- `scaffold_builds`: `turbo build` completes without errors across all workspaces
- `scaffold_type_resolution`: `packages/types` is importable from both `apps/api` and `apps/web`
- `scaffold_dev_server`: `turbo dev` starts both API (port 3001) and web (port 3000) concurrently

#### 1.2 — Docker Compose Local Environment
**What**: Create Docker Compose configuration for PostgreSQL 16 and Redis 7 with persistent volumes.
**Design**:
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: refactor_assistant
      POSTGRES_USER: refactor
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD:-localdev}
    ports: ["5432:5432"]
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U refactor"]
      interval: 5s
      timeout: 5s
      retries: 5

  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s

volumes:
  pgdata:
  redisdata:
```
**Testing**:
- `docker_postgres_healthy`: `docker compose up -d` starts PostgreSQL; healthcheck passes within 15s
- `docker_redis_healthy`: Redis healthcheck passes; `redis-cli ping` returns PONG
- `docker_data_persists`: Data survives `docker compose down && docker compose up`

#### 1.3 — Database Schema (Hybrid Relational + JSONB)
**What**: Implement the full database schema using Drizzle ORM, covering all 14 tables from Data Model Suggestion 3.
**Design**:
```typescript
// apps/api/src/db/schema.ts
import { pgTable, uuid, varchar, text, integer, numeric, boolean,
         timestamp, date, jsonb, uniqueIndex, index } from 'drizzle-orm/pg-core';

export const organizations = pgTable('organizations', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 100 }).notNull().unique(),
  planTier: varchar('plan_tier', { length: 50 }).notNull().default('free'),
  settings: jsonb('settings').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: varchar('email', { length: 255 }).notNull().unique(),
  displayName: varchar('display_name', { length: 255 }).notNull(),
  authProvider: varchar('auth_provider', { length: 50 }).notNull(),
  authProviderId: varchar('auth_provider_id', { length: 255 }),
  profile: jsonb('profile').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

export const organizationMemberships = pgTable('organization_memberships', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  role: varchar('role', { length: 50 }).notNull().default('member'),
  permissions: jsonb('permissions').notNull().default([]),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_org_memberships_unique').on(table.organizationId, table.userId),
]);

export const repositories = pgTable('repositories', {
  id: uuid('id').primaryKey().defaultRandom(),
  organizationId: uuid('organization_id').notNull().references(() => organizations.id, { onDelete: 'cascade' }),
  name: varchar('name', { length: 255 }).notNull(),
  slug: varchar('slug', { length: 255 }).notNull(),
  vcsProvider: varchar('vcs_provider', { length: 50 }).notNull(),
  vcsUrl: text('vcs_url').notNull(),
  defaultBranch: varchar('default_branch', { length: 255 }).notNull().default('main'),
  isActive: boolean('is_active').notNull().default(true),
  lastAnalyzedAt: timestamp('last_analyzed_at', { withTimezone: true }),
  repoMetadata: jsonb('repo_metadata').notNull().default({}),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_repos_org_slug').on(table.organizationId, table.slug),
]);

export const sourceFiles = pgTable('source_files', {
  id: uuid('id').primaryKey().defaultRandom(),
  repositoryId: uuid('repository_id').notNull().references(() => repositories.id, { onDelete: 'cascade' }),
  filePath: text('file_path').notNull(),
  language: varchar('language', { length: 50 }),
  loc: integer('loc'),
  isDeleted: boolean('is_deleted').notNull().default(false),
  complexityScore: numeric('complexity_score', { precision: 10, scale: 2 }),
  churnScore: numeric('churn_score', { precision: 10, scale: 2 }),
  hotspotScore: numeric('hotspot_score', { precision: 10, scale: 2 }),
  hotspotRank: integer('hotspot_rank'),
  openFindingCount: integer('open_finding_count').notNull().default(0),
  debtMinutes: integer('debt_minutes').notNull().default(0),
  metrics: jsonb('metrics').notNull().default({}),
  vcsStats: jsonb('vcs_stats').notNull().default({}),
  firstSeenAt: timestamp('first_seen_at', { withTimezone: true }).notNull().defaultNow(),
  lastSeenAt: timestamp('last_seen_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => [
  uniqueIndex('idx_files_repo_path').on(table.repositoryId, table.filePath),
  index('idx_files_hotspot').on(table.repositoryId, table.hotspotScore),
]);

export const analysisRuns = pgTable('analysis_runs', {
  id: uuid('id').primaryKey().defaultRandom(),
  repositoryId: uuid('repository_id').notNull().references(() => repositories.id, { onDelete: 'cascade' }),
  triggeredBy: uuid('triggered_by').references(() => users.id),
  triggerType: varchar('trigger_type', { length: 50 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('pending'),
  commitSha: varchar('commit_sha', { length: 40 }),
  branch: varchar('branch', { length: 255 }),
  filesAnalyzed: integer('files_analyzed'),
  findingsTotal: integer('findings_total'),
  findingsNew: integer('findings_new'),
  findingsResolved: integer('findings_resolved'),
  debtTotalMinutes: integer('debt_total_minutes'),
  results: jsonb('results').notNull().default({}),
  errorMessage: text('error_message'),
  startedAt: timestamp('started_at', { withTimezone: true }),
  completedAt: timestamp('completed_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const findings = pgTable('findings', {
  id: uuid('id').primaryKey().defaultRandom(),
  analysisId: uuid('analysis_id').notNull().references(() => analysisRuns.id, { onDelete: 'cascade' }),
  sourceFileId: uuid('source_file_id').notNull().references(() => sourceFiles.id, { onDelete: 'cascade' }),
  repositoryId: uuid('repository_id').notNull().references(() => repositories.id, { onDelete: 'cascade' }),
  ruleKey: varchar('rule_key', { length: 255 }).notNull(),
  ruleName: varchar('rule_name', { length: 500 }),
  severity: varchar('severity', { length: 20 }).notNull(),
  findingType: varchar('finding_type', { length: 50 }).notNull(),
  status: varchar('status', { length: 50 }).notNull().default('open'),
  characteristic: varchar('characteristic', { length: 50 }),
  cweId: integer('cwe_id'),
  owaspId: varchar('owasp_id', { length: 10 }),
  remediationMinutes: integer('remediation_minutes'),
  assignedTo: uuid('assigned_to').references(() => users.id),
  location: jsonb('location').notNull(),
  detail: jsonb('detail').notNull().default({}),
  resolvedAt: timestamp('resolved_at', { withTimezone: true }),
  resolvedBy: uuid('resolved_by').references(() => users.id),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// Remaining tables: refactoringRecipes, refactoringCampaigns, transformations,
// debtBudgets, auditLog, qualityGateProfiles, trendSnapshots
// follow the same Hybrid JSONB pattern from Data Model Suggestion 3
```
**Testing**:
- `schema_migration_clean`: `drizzle-kit migrate` applies all migrations to a fresh database without errors
- `schema_all_tables_exist`: Query `information_schema.tables` confirms all 14 tables exist
- `schema_jsonb_indexes`: GIN indexes on `repo_metadata`, `metrics`, `vcs_stats`, `location`, `detail`, `content`, `context` confirmed via `pg_indexes`
- `schema_foreign_keys`: All FK constraints enforced; inserting a finding with a nonexistent `analysis_id` raises an error

#### 1.4 — Shared Type Definitions
**What**: Define TypeScript domain types in `packages/types` that mirror the database schema and serve as the API contract.
**Design**:
```typescript
// packages/types/src/domain/source-file.ts
export interface SourceFile {
  id: string;
  repositoryId: string;
  filePath: string;
  language: string | null;
  loc: number | null;
  isDeleted: boolean;
  complexityScore: number | null;
  churnScore: number | null;
  hotspotScore: number | null;
  hotspotRank: number | null;
  openFindingCount: number;
  debtMinutes: number;
  metrics: SourceFileMetrics;
  vcsStats: VcsStats;
  firstSeenAt: string;
  lastSeenAt: string;
}

export interface SourceFileMetrics {
  cyclomaticComplexity?: number;
  cognitiveComplexity?: number;
  halstead?: { volume: number; difficulty: number; effort: number };
  maintainabilityIndex?: number;
  dependencies?: string[];
  exports?: string[];
  testCoverage?: number;
  duplicationBlocks?: number;
  frameworkPatterns?: Record<string, unknown>;
}

export interface VcsStats {
  totalCommits?: number;
  uniqueAuthors?: number;
  lastModifiedAt?: string;
  lastAuthor?: string;
  monthlyChurn?: MonthlyChurn[];
  topContributors?: ContributorAffinity[];
  coChangedFiles?: FileCoupling[];
}

export interface MonthlyChurn {
  month: string;        // 'YYYY-MM'
  commits: number;
  linesAdded: number;
  linesDeleted: number;
}

export interface ContributorAffinity {
  email: string;
  commits: number;
  affinity: number;     // 0.0 - 1.0
}

export interface FileCoupling {
  path: string;
  couplingScore: number; // 0.0 - 1.0
}

// packages/types/src/domain/finding.ts
export type Severity = 'blocker' | 'critical' | 'major' | 'minor' | 'info';
export type FindingType = 'bug' | 'vulnerability' | 'code_smell' | 'security_hotspot';
export type FindingStatus = 'open' | 'confirmed' | 'resolved' | 'wont_fix' | 'false_positive';
export type QualityCharacteristic = 'maintainability' | 'reliability' | 'security' | 'performance_efficiency';

export interface Finding {
  id: string;
  analysisId: string;
  sourceFileId: string;
  repositoryId: string;
  ruleKey: string;
  ruleName: string | null;
  severity: Severity;
  findingType: FindingType;
  status: FindingStatus;
  characteristic: QualityCharacteristic | null;
  cweId: number | null;
  owaspId: string | null;
  remediationMinutes: number | null;
  assignedTo: string | null;
  location: SarifPhysicalLocation;
  detail: FindingDetail;
  resolvedAt: string | null;
  resolvedBy: string | null;
  createdAt: string;
  updatedAt: string;
}

export interface SarifPhysicalLocation {
  filePath: string;
  region: {
    startLine: number;
    startColumn?: number;
    endLine?: number;
    endColumn?: number;
  };
  contextRegion?: {
    startLine: number;
    endLine: number;
    snippet?: string;
  };
}

export interface FindingDetail {
  message: string;
  aiExplanation?: string;
  relatedFindings?: string[];
  dataFlow?: DataFlowStep[];
  tags?: string[];
  effortBreakdown?: { analysis: number; coding: number; testing: number; review: number };
}

export interface DataFlowStep {
  step: number;
  location: string;
  description: string;
}
```
**Testing**:
- `types_compile`: `tsc --noEmit` passes in `packages/types` with strict mode
- `types_importable`: Both `apps/api` and `apps/web` can import and use types without errors
- `types_exhaustive`: Every database table has a corresponding TypeScript interface

#### 1.5 — Fastify API Server Scaffold
**What**: Set up the Fastify server with health check, OpenAPI documentation, error handling, and CORS.
**Design**:
```typescript
// apps/api/src/server.ts
import Fastify from 'fastify';
import cors from '@fastify/cors';
import swagger from '@fastify/swagger';
import swaggerUi from '@fastify/swagger-ui';
import { config } from './config.js';

export async function buildServer() {
  const app = Fastify({
    logger: { level: config.logLevel },
    ajv: { customOptions: { allErrors: true } },
  });

  await app.register(cors, { origin: config.corsOrigins });
  await app.register(swagger, {
    openapi: {
      info: {
        title: 'Codebase Refactoring Assistant API',
        version: '0.1.0',
      },
      servers: [{ url: config.apiBaseUrl }],
    },
  });
  await app.register(swaggerUi, { routePrefix: '/docs' });

  app.get('/health', {
    schema: {
      response: {
        200: {
          type: 'object',
          properties: {
            status: { type: 'string', enum: ['ok'] },
            version: { type: 'string' },
            timestamp: { type: 'string', format: 'date-time' },
          },
        },
      },
    },
  }, async () => ({
    status: 'ok' as const,
    version: config.version,
    timestamp: new Date().toISOString(),
  }));

  return app;
}
```
**Testing**:
- `api_health_check`: `GET /health` returns `{ status: "ok" }` with 200
- `api_openapi_doc`: `GET /docs/json` returns valid OpenAPI 3.1 document
- `api_cors_headers`: Response includes `Access-Control-Allow-Origin` for configured origins
- `api_404_handler`: Unknown route returns `{ error: "Not Found" }` with 404

---

## Phase 2: VCS Ingestion and Behavioral Analysis

### Purpose
Ingest git history from repositories and compute churn metrics, file coupling, and complexity scores — the behavioral foundation for hotspot detection.

### Tasks

#### 2.1 — Repository Registration API
**What**: CRUD endpoints for registering repositories with their VCS provider details.
**Design**:
```typescript
// POST /api/v1/repositories
interface CreateRepositoryRequest {
  name: string;
  vcsProvider: 'github' | 'gitlab' | 'bitbucket' | 'azure_devops';
  vcsUrl: string;            // e.g., 'https://github.com/acme/frontend'
  defaultBranch?: string;    // defaults to 'main'
}

interface CreateRepositoryResponse {
  id: string;
  name: string;
  slug: string;
  vcsProvider: string;
  vcsUrl: string;
  defaultBranch: string;
  isActive: boolean;
  repoMetadata: Record<string, unknown>;
  createdAt: string;
}

// GET /api/v1/repositories
// GET /api/v1/repositories/:id
// PATCH /api/v1/repositories/:id
// DELETE /api/v1/repositories/:id

// Error responses:
// 409 Conflict: { error: "Repository with slug 'frontend' already exists" }
// 400 Bad Request: { error: "Invalid VCS URL format" }
```
**Testing**:
- `repo_create_valid`: POST with valid GitHub URL creates repository and returns 201
- `repo_create_duplicate_slug`: Second POST with same org+slug returns 409
- `repo_list_org_scoped`: GET returns only repositories for the authenticated organization
- `repo_update_branch`: PATCH with `{ defaultBranch: "develop" }` updates and returns 200
- `repo_delete_cascades`: DELETE removes repository and all associated source_files

#### 2.2 — Git History Ingestion Service
**What**: Clone or fetch a repository and ingest commit history into computed VCS stats on source_files.
**Design**:
```typescript
// apps/api/src/services/vcs-ingestion.ts
import simpleGit, { SimpleGit } from 'simple-git';

interface VcsIngestionOptions {
  repositoryId: string;
  cloneUrl: string;
  branch: string;
  since?: Date;          // only ingest commits after this date
  maxCommits?: number;   // safety limit, default 50_000
}

interface IngestionResult {
  commitsProcessed: number;
  filesDiscovered: number;
  timeRangeStart: Date;
  timeRangeEnd: Date;
  durationMs: number;
}

export class VcsIngestionService {
  constructor(
    private readonly db: DrizzleDB,
    private readonly workDir: string,  // temp directory for clones
  ) {}

  async ingest(options: VcsIngestionOptions): Promise<IngestionResult>;

  // Internal: parse git log into structured commit data
  // Uses: git log --numstat --format='%H|%aE|%aN|%aI|%s' --no-merges
  private async parseGitLog(git: SimpleGit, since?: Date): Promise<CommitRecord[]>;

  // Internal: upsert source_files and update vcs_stats JSONB
  private async updateFileStats(
    repositoryId: string,
    commits: CommitRecord[],
  ): Promise<void>;
}

interface CommitRecord {
  sha: string;
  authorEmail: string;
  authorName: string;
  authoredAt: Date;
  message: string;
  files: Array<{
    path: string;
    linesAdded: number;
    linesDeleted: number;
    changeType: 'A' | 'M' | 'D' | 'R';
  }>;
}
```
**Testing**:
- `ingest_small_repo`: Ingest a test repo with 50 commits; verify all files discovered and commit count matches
- `ingest_rename_tracking`: File renamed from `old.ts` to `new.ts` updates `source_files` path correctly
- `ingest_incremental`: Second ingestion with `since` only processes new commits
- `ingest_deleted_files`: Deleted files marked `is_deleted = true` in source_files
- `ingest_max_commits`: Ingestion stops at `maxCommits` limit and reports partial result

#### 2.3 — Churn Score Calculator
**What**: Compute normalized churn scores per file from ingested VCS stats, using configurable time windows.
**Design**:
```typescript
// packages/utils/src/churn.ts
interface ChurnConfig {
  windowDays: number;       // default: 90
  decayFactor: number;      // exponential decay for older commits, default: 0.95
  minCommits: number;       // minimum commits to be considered, default: 3
}

interface ChurnResult {
  fileId: string;
  filePath: string;
  rawCommitCount: number;
  weightedCommitCount: number;  // with decay applied
  uniqueAuthors: number;
  linesChurned: number;         // added + deleted
  churnScore: number;           // 0-100 normalized across repo
}

export function calculateChurnScores(
  files: Array<{ id: string; path: string; vcsStats: VcsStats }>,
  config: ChurnConfig,
): ChurnResult[];

// Normalization: min-max scaling across the repository
// churnScore = ((weighted - minWeighted) / (maxWeighted - minWeighted)) * 100
// Files below minCommits threshold get churnScore = 0
```
**Testing**:
- `churn_normalization`: Highest-churn file gets score 100; lowest gets score 0
- `churn_decay`: Recent commits weight more than older commits with decay factor 0.95
- `churn_min_threshold`: Files with fewer than `minCommits` receive churnScore 0
- `churn_single_file_repo`: Single-file repo gets churnScore 100
- `churn_empty_window`: No commits in window returns churnScore 0 for all files

#### 2.4 — Complexity Calculator (tree-sitter based)
**What**: Compute cyclomatic and cognitive complexity for source files using tree-sitter CST traversal.
**Design**:
```typescript
// apps/api/src/services/complexity-calculator.ts
import Parser from 'tree-sitter';

interface ComplexityResult {
  fileId: string;
  filePath: string;
  language: string;
  cyclomaticComplexity: number;
  cognitiveComplexity: number;
  loc: number;
  functions: FunctionComplexity[];
}

interface FunctionComplexity {
  name: string;
  startLine: number;
  endLine: number;
  cyclomaticComplexity: number;
  cognitiveComplexity: number;
}

export class ComplexityCalculator {
  private parsers: Map<string, Parser>;

  constructor();

  async calculateFile(
    filePath: string,
    content: string,
    language: string,
  ): Promise<ComplexityResult>;

  // Cyclomatic: count decision points (if, else if, while, for, case, catch, &&, ||, ?:)
  // Cognitive: nesting-aware complexity (increment per decision + nesting penalty)
  // Supported languages: typescript, javascript, python, java, go, rust, csharp, php
  private traverseForComplexity(
    node: Parser.SyntaxNode,
    language: string,
    nestingLevel: number,
  ): { cyclomatic: number; cognitive: number };
}

// Language-specific decision node types:
const DECISION_NODES: Record<string, string[]> = {
  typescript: ['if_statement', 'while_statement', 'for_statement', 'for_in_statement',
               'switch_case', 'catch_clause', 'ternary_expression',
               'binary_expression'], // filter: && and ||
  python: ['if_statement', 'while_statement', 'for_statement', 'except_clause',
           'conditional_expression', 'boolean_operator'],
  java: ['if_statement', 'while_statement', 'for_statement', 'enhanced_for_statement',
         'switch_case', 'catch_clause', 'ternary_expression', 'binary_expression'],
  go: ['if_statement', 'for_statement', 'select_statement', 'case_clause',
       'binary_expression'],
};
```
**Testing**:
- `complexity_simple_function`: `function add(a, b) { return a + b; }` returns cyclomatic=1, cognitive=0
- `complexity_nested_if`: Three levels of nested `if` returns cognitive > cyclomatic
- `complexity_python_for_comprehension`: Python list comprehension counted as a decision point
- `complexity_java_switch`: Switch with 5 cases returns cyclomatic=5
- `complexity_per_function`: Multi-function file returns per-function breakdown
- `complexity_unknown_language`: Unsupported language throws `UnsupportedLanguageError`

#### 2.5 — File Coupling Detection
**What**: Identify files that change together frequently (temporal coupling) from VCS history.
**Design**:
```typescript
// packages/utils/src/coupling.ts
interface CouplingConfig {
  windowDays: number;            // default: 180
  minCoChanges: number;          // minimum co-changes to report, default: 3
  minCouplingScore: number;      // minimum Jaccard similarity, default: 0.3
  maxResults: number;            // per file, default: 10
}

interface CouplingPair {
  fileAPath: string;
  fileBPath: string;
  coChangeCount: number;
  totalChangesA: number;
  totalChangesB: number;
  couplingScore: number;  // Jaccard: coChanges / (changesA + changesB - coChanges)
}

export function detectFileCoupling(
  commitFileMap: Map<string, Set<string>>,  // commitSha -> set of file paths
  config: CouplingConfig,
): CouplingPair[];
```
**Testing**:
- `coupling_always_together`: Two files changed in every commit get score 1.0
- `coupling_never_together`: Files never co-changed are not included in results
- `coupling_min_threshold`: Pairs below `minCoChanges` are excluded
- `coupling_jaccard_calculation`: 3 co-changes out of 5+4-3=6 total returns score 0.5
- `coupling_symmetry`: Coupling(A,B) equals Coupling(B,A)

---

## Phase 3: Hotspot Detection and Static Analysis

### Purpose
Combine churn and complexity scores to identify hotspots, implement rule-based static analysis with detection rules, and produce the core findings that drive refactoring decisions.

### Tasks

#### 3.1 — Hotspot Detection Engine
**What**: Compute composite hotspot scores from churn and complexity, rank files, and persist results.
**Design**:
```typescript
// apps/api/src/services/hotspot-detector.ts
interface HotspotConfig {
  churnWeight: number;           // default: 0.5
  complexityWeight: number;      // default: 0.5
  minChurnScore: number;         // minimum churn to consider, default: 10
  minComplexityScore: number;    // minimum complexity to consider, default: 5
  topN: number;                  // return top N hotspots, default: 50
}

interface Hotspot {
  fileId: string;
  filePath: string;
  language: string;
  churnScore: number;
  complexityScore: number;
  hotspotScore: number;           // churnWeight * churn + complexityWeight * complexity (normalized 0-100)
  hotspotRank: number;            // 1 = highest priority
  defectProbability: number;      // predicted defect likelihood (0-1), from logistic regression on historical data
  topContributors: ContributorAffinity[];
  coupledFiles: FileCoupling[];
}

export class HotspotDetector {
  constructor(private readonly db: DrizzleDB) {}

  async detectHotspots(
    repositoryId: string,
    config: HotspotConfig,
  ): Promise<Hotspot[]>;

  // Updates source_files.hotspot_score and source_files.hotspot_rank
  async persistHotspots(
    repositoryId: string,
    hotspots: Hotspot[],
  ): Promise<void>;
}
```
**Testing**:
- `hotspot_high_churn_high_complexity`: File with both high churn and complexity ranks #1
- `hotspot_high_churn_low_complexity`: File with high churn but low complexity ranks lower than both-high
- `hotspot_min_thresholds`: Files below both minChurn and minComplexity are excluded
- `hotspot_ranking_stable`: Same inputs produce same ranking (deterministic)
- `hotspot_weight_adjustment`: Setting churnWeight=1.0, complexityWeight=0.0 ranks purely by churn

#### 3.2 — Detection Rule Engine
**What**: Implement a configurable rule engine that detects code quality issues using tree-sitter pattern matching and heuristic analysis.
**Design**:
```typescript
// apps/api/src/services/finding-detector.ts
interface DetectionRule {
  ruleKey: string;                // e.g., 'ts:cognitive-complexity'
  name: string;
  language: string | null;        // null = language-agnostic
  severity: Severity;
  findingType: FindingType;
  characteristic: QualityCharacteristic;
  cweId: number | null;
  owaspId: string | null;
  remediationMinutes: number;
  detector: (file: AnalyzedFile) => DetectedIssue[];
}

interface AnalyzedFile {
  filePath: string;
  content: string;
  language: string;
  cst: Parser.Tree;              // tree-sitter CST
  complexityResult: ComplexityResult;
  vcsStats: VcsStats;
}

interface DetectedIssue {
  ruleKey: string;
  severity: Severity;
  location: SarifPhysicalLocation;
  message: string;
  remediationMinutes: number;
}

// Built-in rules (MVP set):
// ts:cognitive-complexity      - cognitive complexity > 15
// ts:function-length           - function body > 50 lines
// ts:file-length               - file > 500 lines
// ts:duplicate-string-literal  - same string literal used > 3 times
// ts:deeply-nested             - nesting depth > 4 levels
// py:cognitive-complexity      - same thresholds for Python
// java:cognitive-complexity    - same thresholds for Java
// generic:large-file           - file > 1000 lines (any language)
// generic:high-coupling        - coupling score > 0.8 with > 5 files
```
**Testing**:
- `rule_cognitive_complexity`: File with cognitive complexity 20 triggers `ts:cognitive-complexity` at `major` severity
- `rule_function_length`: 60-line function triggers `ts:function-length`
- `rule_below_threshold`: File with complexity 10 does not trigger any rules
- `rule_language_specific`: Python rule does not fire on TypeScript files
- `rule_location_accuracy`: Finding location contains correct startLine and endLine
- `rule_remediation_cost`: Each finding includes estimated remediation minutes

#### 3.3 — Analysis Run Orchestration
**What**: Coordinate a full analysis run: clone/fetch repo, compute complexity, compute churn, detect hotspots, run rules, persist findings.
**Design**:
```typescript
// apps/api/src/services/analysis-engine.ts
export class AnalysisEngine {
  constructor(
    private readonly db: DrizzleDB,
    private readonly vcsIngestion: VcsIngestionService,
    private readonly complexityCalc: ComplexityCalculator,
    private readonly hotspotDetector: HotspotDetector,
    private readonly findingDetector: FindingDetector,
    private readonly sarifExporter: SarifExporter,
  ) {}

  async runAnalysis(analysisId: string): Promise<AnalysisRunResult>;
}

// POST /api/v1/repositories/:repoId/analyses
// Returns: { id: string, status: 'pending' }
// Analysis runs asynchronously via BullMQ worker

// GET /api/v1/analyses/:analysisId
// Returns analysis status and results

// GET /api/v1/analyses/:analysisId/sarif
// Returns SARIF 2.1.0 JSON document of findings

type AnalysisStatus = 'pending' | 'cloning' | 'analyzing' | 'completed' | 'failed';

interface AnalysisRunResult {
  analysisId: string;
  status: 'completed' | 'failed';
  filesAnalyzed: number;
  findingsTotal: number;
  findingsNew: number;
  debtTotalMinutes: number;
  hotspotCount: number;
  durationMs: number;
  errorMessage?: string;
}
```
**Testing**:
- `analysis_full_run`: Trigger analysis on test repo; status transitions: pending -> cloning -> analyzing -> completed
- `analysis_creates_findings`: Completed analysis has findingsTotal > 0 for a repo with known issues
- `analysis_updates_hotspots`: After analysis, source_files.hotspot_score is populated for analyzed files
- `analysis_incremental`: Second analysis on same repo with no changes produces findingsNew = 0
- `analysis_failure_handling`: Analysis with invalid clone URL transitions to 'failed' with error message
- `analysis_sarif_export`: `GET /analyses/:id/sarif` returns valid SARIF 2.1.0 JSON matching the OASIS schema

#### 3.4 — SARIF Export Service
**What**: Export analysis findings as SARIF 2.1.0 JSON documents for GitHub Advanced Security integration.
**Design**:
```typescript
// packages/utils/src/sarif.ts
// Implements: OASIS SARIF 2.1.0 (https://docs.oasis-open.org/sarif/sarif/v2.1.0/sarif-v2.1.0.html)

interface SarifDocument {
  $schema: 'https://raw.githubusercontent.com/oasis-tcs/sarif-spec/master/Schemata/sarif-schema-2.1.0.json';
  version: '2.1.0';
  runs: SarifRun[];
}

interface SarifRun {
  tool: {
    driver: {
      name: 'Codebase Refactoring Assistant';
      version: string;
      informationUri: string;
      rules: SarifReportingDescriptor[];
    };
  };
  results: SarifResult[];
}

export function exportToSarif(
  findings: Finding[],
  rules: Map<string, DetectionRule>,
  toolVersion: string,
): SarifDocument;

// Maps severity to SARIF level:
// blocker/critical -> 'error'
// major -> 'warning'
// minor -> 'note'
// info -> 'none'
```
**Testing**:
- `sarif_valid_schema`: Output validates against the official SARIF 2.1.0 JSON Schema
- `sarif_severity_mapping`: Critical finding maps to SARIF level 'error'
- `sarif_location_mapping`: Finding location maps to SARIF physicalLocation with correct region
- `sarif_rule_descriptors`: Each unique rule appears in `tool.driver.rules`
- `sarif_empty_findings`: Zero findings produces valid SARIF with empty results array

#### 3.5 — Findings and Hotspots API
**What**: REST endpoints for querying findings and hotspots with filtering, sorting, and pagination.
**Design**:
```typescript
// GET /api/v1/repositories/:repoId/findings
interface FindingsQuery {
  status?: FindingStatus;
  severity?: Severity;
  findingType?: FindingType;
  characteristic?: QualityCharacteristic;
  cweId?: number;
  filePath?: string;             // prefix match
  page?: number;                 // default: 1
  pageSize?: number;             // default: 50, max: 200
  sortBy?: 'severity' | 'remediation_minutes' | 'created_at';
  sortOrder?: 'asc' | 'desc';
}

// Response: { data: Finding[], pagination: { page, pageSize, totalCount, totalPages } }

// PATCH /api/v1/findings/:findingId
interface UpdateFindingRequest {
  status?: FindingStatus;
  assignedTo?: string | null;
}

// GET /api/v1/repositories/:repoId/hotspots
// Response: Hotspot[] (ordered by hotspotRank ascending)

// GET /api/v1/repositories/:repoId/hotspots/:fileId
// Response: detailed hotspot with coupling, contributors, and related findings
```
**Testing**:
- `findings_filter_severity`: `?severity=critical` returns only critical findings
- `findings_pagination`: Page 2 with pageSize 10 returns items 11-20
- `findings_sort_remediation`: Sorting by `remediation_minutes desc` returns highest-cost findings first
- `findings_update_status`: PATCH to mark finding as `wont_fix` updates status
- `hotspots_ordered_by_rank`: Hotspot list returns rank 1 first
- `hotspot_detail_includes_coupling`: Single hotspot response includes coupled files and contributors

---

## Phase 4: LLM-Powered Semantic Refactoring

### Purpose
Integrate Claude API for semantic code analysis: generating contextual refactoring explanations, producing refactored code, and creating AI-powered transformation plans from hotspot data.

### Tasks

#### 4.1 — LLM Service with Prompt Caching
**What**: Build a service layer around the Anthropic SDK with prompt caching, retry logic, and structured output.
**Design**:
```typescript
// apps/api/src/services/llm-refactoring.ts
import Anthropic from '@anthropic-ai/sdk';

interface LLMConfig {
  model: string;                 // default: 'claude-sonnet-4-20250514'
  maxTokens: number;             // default: 8192
  temperature: number;           // default: 0.2 for deterministic refactoring
  cacheControl: boolean;         // enable prompt caching, default: true
}

interface RefactoringRequest {
  fileContent: string;
  filePath: string;
  language: string;
  finding: Finding;
  codebaseContext?: string;       // related files, imports, call sites
  hotspotContext?: {
    churnScore: number;
    complexityScore: number;
    topContributors: string[];
    coupledFiles: string[];
  };
}

interface RefactoringResponse {
  explanation: string;            // why this refactoring is appropriate
  refactoredCode: string;        // the complete refactored file content
  diff: string;                  // unified diff
  riskAssessment: {
    riskScore: number;           // 0-100
    riskFactors: string[];
    mitigations: string[];
  };
  affectedImports: string[];
  breakingChanges: boolean;
}

export class LLMRefactoringService {
  private client: Anthropic;
  private systemPromptCacheKey: string;

  constructor(config: LLMConfig);

  async generateRefactoring(request: RefactoringRequest): Promise<RefactoringResponse>;

  // Uses tool_use for structured output:
  // Tool: "propose_refactoring" with JSON schema for RefactoringResponse
  private buildMessages(request: RefactoringRequest): Anthropic.MessageParam[];

  // System prompt includes:
  // - Role: expert code refactoring assistant
  // - Standards: ISO 25010 quality characteristics, CWE mapping
  // - Constraints: preserve public API surface, maintain test compatibility
  // - Format: respond via propose_refactoring tool
  // Cached via cache_control: { type: 'ephemeral' } on system prompt
}
```
**Testing**:
- `llm_generates_refactoring`: Given a complex function, returns valid RefactoringResponse with diff
- `llm_explains_why`: Explanation references the specific finding and business context
- `llm_risk_assessment`: High-complexity refactoring returns riskScore > 50
- `llm_preserves_api`: Refactored code maintains the same export interface
- `llm_cache_hit`: Second call with same system prompt achieves cache hit (verify via response headers)
- `llm_retry_on_429`: Service retries on rate limit errors with exponential backoff

#### 4.2 — Transformation Planning from Hotspots
**What**: Given a hotspot, use LLM to analyze the file and produce a concrete refactoring plan with multiple transformation steps.
**Design**:
```typescript
// POST /api/v1/hotspots/:fileId/plan
interface PlanRequest {
  strategy?: 'decompose' | 'simplify' | 'extract' | 'auto';  // default: 'auto'
}

interface TransformationPlan {
  fileId: string;
  filePath: string;
  summary: string;
  estimatedEffortMinutes: number;
  steps: TransformationStep[];
  prerequisites: string[];        // e.g., "Add tests for function X before refactoring"
  risks: string[];
}

interface TransformationStep {
  order: number;
  title: string;
  description: string;
  affectedLines: { start: number; end: number };
  technique: string;             // 'extract_function', 'decompose_conditional', 'introduce_parameter_object', etc.
  estimatedMinutes: number;
  dependencies: number[];        // step orders that must complete first
}
```
**Testing**:
- `plan_decompose_large_function`: 200-line function produces plan with 'extract_function' steps
- `plan_ordering`: Steps with dependencies come after their prerequisites
- `plan_effort_estimate`: Total estimated minutes is sum of step estimates
- `plan_auto_strategy`: 'auto' selects appropriate strategy based on file characteristics
- `plan_prerequisites`: Plan for a file with no tests includes "add characterization tests" prerequisite

#### 4.3 — Cross-File Impact Analysis
**What**: Before applying a refactoring, identify all files that import, call, or reference the changed symbols.
**Design**:
```typescript
// apps/api/src/services/impact-analysis.ts
interface ImpactAnalysisResult {
  targetFile: string;
  changedSymbols: string[];       // exported names being modified
  impactedFiles: ImpactedFile[];
  apiContractChange: boolean;     // true if public type signatures change
  breakingChangeRisk: 'none' | 'low' | 'medium' | 'high';
}

interface ImpactedFile {
  filePath: string;
  importedSymbols: string[];
  usageLocations: SarifPhysicalLocation[];
  changeRequired: boolean;        // true if downstream changes needed
}

export class ImpactAnalysisService {
  constructor(
    private readonly db: DrizzleDB,
    private readonly complexityCalc: ComplexityCalculator, // for tree-sitter parsing
  ) {}

  async analyzeImpact(
    repositoryId: string,
    targetFilePath: string,
    changedSymbols: string[],
  ): Promise<ImpactAnalysisResult>;

  // Uses tree-sitter to parse import statements across all files
  // in the repository to find usages of changedSymbols
  private async findImporters(
    repoPath: string,
    targetFile: string,
    symbols: string[],
  ): Promise<ImpactedFile[]>;
}
```
**Testing**:
- `impact_direct_importers`: Changing `processPayment` finds files that import it
- `impact_no_importers`: Changing unexported function returns empty impactedFiles
- `impact_re_exports`: File that re-exports the symbol is identified
- `impact_breaking_change`: Changing a function signature marks `apiContractChange: true`
- `impact_cross_directory`: Finds importers across different directory levels

#### 4.4 — Transformation Execution and Persistence
**What**: Create, store, and manage individual transformations with status tracking.
**Design**:
```typescript
// POST /api/v1/transformations
interface CreateTransformationRequest {
  recipeId?: string;
  sourceFileId: string;
  findingId?: string;
  campaignId?: string;
}

// Transformation status flow:
// pending -> generating -> testing -> ready_for_review -> approved -> applied
//                                                      -> rejected
//        -> failed (at any stage)

// GET /api/v1/transformations/:id
// Returns: full transformation with diff, explanation, test results, impact analysis

// POST /api/v1/transformations/:id/approve
// POST /api/v1/transformations/:id/reject
// POST /api/v1/transformations/:id/apply
```
**Testing**:
- `transform_create`: POST creates transformation in 'pending' status
- `transform_status_flow`: Status transitions follow the defined flow; invalid transitions return 409
- `transform_approve_reject`: Approved/rejected transformations record the acting user
- `transform_detail`: GET includes diff, explanation, riskScore, and test results
- `transform_idempotent_apply`: Applying an already-applied transformation returns 409

---

## Phase 5: Test-Anchored Safe Execution

### Purpose
Generate characterization tests before refactoring and validate them after, providing the safety guarantee that transformations are behavior-preserving.

### Tasks

#### 5.1 — Characterization Test Generator
**What**: Use LLM to generate tests that capture the current behavior of a function or file before refactoring.
**Design**:
```typescript
// apps/api/src/services/test-generator.ts
interface TestGenerationRequest {
  fileContent: string;
  filePath: string;
  language: string;
  targetFunctions?: string[];     // specific functions to test; null = all exports
  existingTests?: string;         // content of existing test file, if any
  testFramework: 'jest' | 'vitest' | 'pytest' | 'junit' | 'go_test';
}

interface GeneratedTest {
  testFilePath: string;           // e.g., 'src/payment/__tests__/processor.char.test.ts'
  testContent: string;
  testFramework: string;
  generationMethod: 'ai_generated';
  testCount: number;
  coveredFunctions: string[];
}

export class TestGeneratorService {
  constructor(private readonly llm: LLMRefactoringService) {}

  async generateCharacterizationTests(
    request: TestGenerationRequest,
  ): Promise<GeneratedTest>;

  // System prompt instructs the LLM to:
  // 1. Identify all exported functions and their signatures
  // 2. Generate tests that exercise edge cases, boundary conditions, error paths
  // 3. Use the specified test framework's assertion style
  // 4. Mock external dependencies (database, HTTP, file system)
  // 5. Name tests descriptively: 'should return X when Y'
}
```
**Testing**:
- `testgen_typescript_jest`: Generates valid Jest tests for a TypeScript function
- `testgen_python_pytest`: Generates valid pytest tests for a Python function
- `testgen_covers_exports`: Generated tests cover all exported functions
- `testgen_mocks_externals`: Tests mock HTTP calls and database access
- `testgen_edge_cases`: Tests include null/undefined, empty array, boundary values

#### 5.2 — Test Executor Service
**What**: Execute test suites in isolated containers and capture results (pass/fail, coverage, output).
**Design**:
```typescript
// apps/api/src/services/test-executor.ts
interface TestExecutionRequest {
  repositoryPath: string;
  testFilePath: string;
  testFramework: 'jest' | 'vitest' | 'pytest' | 'junit' | 'go_test';
  phase: 'pre_refactor' | 'post_refactor';
  timeoutMs?: number;            // default: 120_000
}

interface TestExecutionResult {
  phase: 'pre_refactor' | 'post_refactor';
  totalTests: number;
  passed: number;
  failed: number;
  skipped: number;
  coveragePercent: number | null;
  durationMs: number;
  isPassing: boolean;
  logOutput: string;
  failedTests: Array<{
    name: string;
    error: string;
  }>;
}

export class TestExecutorService {
  // Executes tests in a subprocess with:
  // - Working directory set to the repository clone
  // - Timeout enforcement
  // - Coverage collection via framework-specific flags
  // - Structured output parsing from JSON reporter

  async execute(request: TestExecutionRequest): Promise<TestExecutionResult>;

  // Framework-specific commands:
  // jest: npx jest --json --coverage --testPathPattern=<testFile>
  // vitest: npx vitest run --reporter=json --coverage <testFile>
  // pytest: python -m pytest --tb=short --json-report <testFile>
  // junit: mvn test -Dtest=<testClass> -Dsurefire.reportFormat=brief
  // go_test: go test -json -cover ./...
}
```
**Testing**:
- `exec_passing_tests`: Executing passing tests returns `isPassing: true`
- `exec_failing_tests`: Executing failing tests returns `isPassing: false` with failedTests details
- `exec_timeout`: Test exceeding timeout returns failure with 'timeout' error
- `exec_coverage`: Coverage percentage is extracted from framework output
- `exec_log_capture`: Full test output is captured in logOutput

#### 5.3 — Safe Refactoring Pipeline
**What**: Orchestrate the full test-anchored refactoring pipeline: generate tests, run pre-refactor, apply transformation, run post-refactor, compare results.
**Design**:
```typescript
// apps/api/src/services/transformation-engine.ts
interface SafeRefactoringPipeline {
  transformationId: string;
  steps: [
    'generate_characterization_tests',
    'execute_pre_refactor_tests',
    'apply_transformation',
    'execute_post_refactor_tests',
    'compare_results',
    'determine_safety_verdict',
  ];
}

type SafetyVerdict = 'pass' | 'fail' | 'degraded';

interface SafetyComparisonResult {
  verdict: SafetyVerdict;
  preResults: TestExecutionResult;
  postResults: TestExecutionResult;
  testCountDelta: number;        // post - pre (should be >= 0)
  coverageDelta: number;         // post - pre (should be >= 0)
  newFailures: string[];         // test names that passed before but fail after
  explanation: string;
}

// Verdict rules:
// 'pass': all pre-passing tests still pass AND coverage >= pre-coverage
// 'degraded': all tests pass but coverage dropped
// 'fail': any pre-passing test now fails

export class TransformationEngine {
  constructor(
    private readonly llm: LLMRefactoringService,
    private readonly testGen: TestGeneratorService,
    private readonly testExec: TestExecutorService,
    private readonly db: DrizzleDB,
  ) {}

  async executeTransformation(transformationId: string): Promise<SafetyComparisonResult>;
}
```
**Testing**:
- `pipeline_pass`: Behavior-preserving refactoring produces verdict 'pass'
- `pipeline_fail_regression`: Refactoring that breaks a test produces verdict 'fail' with newFailures
- `pipeline_degraded_coverage`: Refactoring that reduces coverage produces verdict 'degraded'
- `pipeline_status_transitions`: Transformation status updates through each pipeline stage
- `pipeline_rollback_on_fail`: Failed transformation reverts the file to its original state
- `pipeline_stores_results`: Both pre and post test results stored in transformation.test_results JSONB

---

## Phase 6: Quality Gates and PR Integration

### Purpose
Implement quality gate profiles that enforce code quality thresholds on pull requests, with inline PR annotations on GitHub and GitLab.

### Tasks

#### 6.1 — Quality Gate Profile Management
**What**: CRUD for quality gate profiles with configurable conditions stored in JSONB.
**Design**:
```typescript
// POST /api/v1/quality-gates
interface CreateQualityGateRequest {
  name: string;
  isDefault?: boolean;
  conditions: QualityGateCondition[];
}

interface QualityGateCondition {
  metric: 'new_critical' | 'new_major' | 'new_issues' | 'coverage_delta' | 'debt_ratio' | 'debt_added_minutes';
  operator: 'GT' | 'LT' | 'GTE' | 'LTE' | 'EQ';
  threshold: number;
  label: string;
}

// GET /api/v1/quality-gates
// GET /api/v1/quality-gates/:id
// PUT /api/v1/quality-gates/:id
// DELETE /api/v1/quality-gates/:id
```
**Testing**:
- `qg_create_with_conditions`: Create profile with 4 conditions; GET returns them correctly
- `qg_default_only_one`: Setting a profile as default unsets the previous default
- `qg_condition_validation`: Invalid metric name returns 400

#### 6.2 — Quality Gate Evaluation Engine
**What**: Evaluate analysis results against a quality gate profile and produce pass/fail/warning verdicts.
**Design**:
```typescript
// apps/api/src/services/quality-gate-evaluator.ts
interface QualityGateResult {
  profileId: string;
  profileName: string;
  overallStatus: 'passed' | 'failed' | 'warning';
  conditions: QualityGateConditionResult[];
}

interface QualityGateConditionResult {
  metric: string;
  operator: string;
  threshold: number;
  actualValue: number;
  status: 'passed' | 'failed';
  label: string;
}

export class QualityGateEvaluator {
  async evaluate(
    analysisId: string,
    profileId: string,
  ): Promise<QualityGateResult>;

  // Metric computation:
  // new_critical: count of new findings with severity='critical' in this analysis
  // new_major: count of new findings with severity='major'
  // new_issues: count of all new findings
  // coverage_delta: coverage difference vs. previous analysis (requires CI integration)
  // debt_ratio: total debt minutes / total LOC * 100
  // debt_added_minutes: sum of remediation_minutes for new findings
}
```
**Testing**:
- `qg_all_pass`: Analysis with 0 new critical and low debt passes all conditions
- `qg_one_fails`: Analysis with 3 new critical issues fails the `new_critical GT 0` condition
- `qg_overall_fail`: If any condition fails, overallStatus is 'failed'
- `qg_metric_calculation`: `debt_ratio` correctly computes as debt_minutes / LOC * 100

#### 6.3 — GitHub PR Integration (Webhooks + Checks API)
**What**: Receive PR webhooks, trigger analysis, post quality gate results as PR checks and inline comments.
**Design**:
```typescript
// apps/api/src/routes/webhooks.ts
// POST /api/v1/webhooks/github
// Handles: pull_request.opened, pull_request.synchronize, pull_request.reopened

// apps/api/src/services/pr-manager.ts
export class PRManager {
  constructor(
    private readonly githubToken: string,
    private readonly analysisEngine: AnalysisEngine,
    private readonly qgEvaluator: QualityGateEvaluator,
  ) {}

  async handlePullRequest(event: GitHubPullRequestEvent): Promise<void>;

  // 1. Create a GitHub Check Run (status: 'in_progress')
  // 2. Trigger analysis on the PR branch
  // 3. Evaluate quality gate
  // 4. Update Check Run with conclusion ('success' | 'failure')
  // 5. Post inline review comments for new findings on changed lines
  // 6. Post summary comment with quality gate results

  async postInlineComments(
    owner: string,
    repo: string,
    prNumber: number,
    findings: Finding[],
    changedFiles: string[],
  ): Promise<void>;

  async postSummaryComment(
    owner: string,
    repo: string,
    prNumber: number,
    qgResult: QualityGateResult,
    analysisResult: AnalysisRunResult,
  ): Promise<void>;
}
```
**Testing**:
- `webhook_signature_validation`: Webhook with invalid HMAC signature returns 401
- `webhook_triggers_analysis`: PR opened webhook creates an analysis run
- `webhook_check_run_created`: GitHub Check Run is created with 'in_progress' status
- `pr_inline_comments`: Findings on changed lines produce inline PR review comments
- `pr_summary_comment`: Summary comment includes quality gate status, finding counts, and hotspot changes
- `pr_check_conclusion`: Passing quality gate sets check conclusion to 'success'

#### 6.4 — GitLab MR Integration
**What**: Mirror the GitHub PR integration for GitLab merge requests using GitLab CI/CD and MR API.
**Design**:
```typescript
// POST /api/v1/webhooks/gitlab
// Handles: merge_request hook events

// GitLab equivalents:
// Check Runs -> Commit Status API + MR Pipeline
// PR review comments -> MR Discussion Notes with position
// Summary comment -> MR Note

export class GitLabPRManager {
  async handleMergeRequest(event: GitLabMergeRequestEvent): Promise<void>;
  async postInlineNotes(projectId: number, mrIid: number, findings: Finding[]): Promise<void>;
  async postSummaryNote(projectId: number, mrIid: number, qgResult: QualityGateResult): Promise<void>;
}
```
**Testing**:
- `gitlab_webhook_triggers_analysis`: MR hook creates analysis run
- `gitlab_inline_notes`: Findings posted as MR discussion notes with correct file/line
- `gitlab_commit_status`: Quality gate result updates commit status

---

## Phase 7: Refactoring Campaigns and Batch Execution

### Purpose
Enable multi-file, multi-repository refactoring campaigns that apply recipes across an organization's codebase at scale.

### Tasks

#### 7.1 — Recipe Management
**What**: CRUD for refactoring recipes with JSONB definitions supporting both rule-based and AI-generated recipes.
**Design**:
```typescript
// POST /api/v1/recipes
interface CreateRecipeRequest {
  name: string;
  recipeType: 'framework_migration' | 'dependency_upgrade' | 'security_fix' | 'code_quality' | 'custom';
  isAiGenerated: boolean;
  definition: RecipeDefinition;
}

interface RecipeDefinition {
  description: string;
  sourceLanguage?: string;
  targetLanguage?: string;
  applicableFrameworks?: string[];
  targetFramework?: string;
  estimatedRisk: 'low' | 'medium' | 'high';
  addressedRules?: string[];      // rule_keys this recipe fixes
  preconditions: RecipePrecondition[];
  transformationSteps: RecipeStep[];
  testStrategy: 'generate_snapshot_tests' | 'generate_unit_tests' | 'use_existing_tests';
  rollbackStrategy: 'git_revert' | 'manual';
}

interface RecipePrecondition {
  type: 'file_pattern' | 'import_present' | 'language_match' | 'framework_version';
  pattern?: string;
  module?: string;
  symbol?: string;
  language?: string;
  version?: string;
}

interface RecipeStep {
  step: string;
  description: string;
}
```
**Testing**:
- `recipe_create`: Create a TypeScript recipe with preconditions; GET returns it
- `recipe_precondition_validation`: Missing required fields in preconditions return 400
- `recipe_list_by_type`: Filter recipes by type returns correct subset
- `recipe_deactivate`: Setting `isActive: false` excludes recipe from listings

#### 7.2 — Campaign Orchestration
**What**: Create and execute refactoring campaigns that apply recipes across multiple repositories.
**Design**:
```typescript
// POST /api/v1/campaigns
interface CreateCampaignRequest {
  name: string;
  description?: string;
  targetRepos: string[];          // repository IDs
  recipeIds: string[];
  filters?: {
    languages?: string[];
    minHotspotScore?: number;
    severityThreshold?: Severity;
  };
  executionMode: 'batch' | 'interactive';
  autoCreatePr: boolean;
  requireTestPass: boolean;
  maxConcurrent?: number;         // default: 5
}

// Campaign status flow:
// draft -> planning -> in_progress -> completed
//                   -> paused -> in_progress
//                              -> cancelled

// POST /api/v1/campaigns/:id/start
// POST /api/v1/campaigns/:id/pause
// POST /api/v1/campaigns/:id/resume
// POST /api/v1/campaigns/:id/cancel
// GET /api/v1/campaigns/:id          (includes progress JSONB)
// GET /api/v1/campaigns/:id/transformations

export class CampaignOrchestrator {
  async planCampaign(campaignId: string): Promise<void>;
  // 1. For each target repo, find files matching recipe preconditions
  // 2. Filter by hotspot score, language, severity
  // 3. Create transformation records for each file × recipe match
  // 4. Update campaign status to 'in_progress'

  async executeCampaign(campaignId: string): Promise<void>;
  // Process transformations in batches of maxConcurrent
  // Each transformation goes through the safe refactoring pipeline (Phase 5)
  // Update campaign.progress JSONB after each transformation completes
}
```
**Testing**:
- `campaign_planning`: Planning a campaign creates transformation records for matching files
- `campaign_execution`: Starting a campaign processes transformations in order
- `campaign_pause_resume`: Pausing stops new transformations; resuming continues from where it paused
- `campaign_progress_tracking`: Campaign progress JSONB updates as transformations complete
- `campaign_max_concurrent`: No more than `maxConcurrent` transformations run simultaneously
- `campaign_auto_pr`: Completed transformations with autoCreatePr=true create PRs

#### 7.3 — Automated PR Creation
**What**: For approved transformations, create pull requests with the refactored code, test results, and explanations.
**Design**:
```typescript
// apps/api/src/services/pr-manager.ts (extended)
export class PRManager {
  async createRefactoringPR(
    transformation: Transformation,
    repositoryId: string,
  ): Promise<{ prUrl: string; prNumber: number }>;

  // Creates:
  // 1. New branch: refactor/<recipe-slug>/<file-name>-<short-id>
  // 2. Commit with the refactored code
  // 3. PR with body containing:
  //    - Finding description
  //    - AI explanation of the refactoring
  //    - Diff preview
  //    - Risk assessment
  //    - Test results (pre/post comparison)
  //    - Quality gate impact
}
```
**Testing**:
- `pr_branch_naming`: Branch follows the naming convention `refactor/<slug>/<name>-<id>`
- `pr_body_includes_explanation`: PR body contains the AI explanation
- `pr_body_includes_test_results`: PR body shows pre/post test comparison
- `pr_body_includes_risk`: PR body shows risk score and factors

---

## Phase 8: Technical Debt Budgeting

### Purpose
Implement active technical debt budget management: allocate refactoring time per sprint, assign tasks to developers based on code familiarity, and track debt reduction against KPIs.

### Tasks

#### 8.1 — Debt Budget Management API
**What**: CRUD for technical debt budgets with sprint-based allocation and task assignment.
**Design**:
```typescript
// POST /api/v1/debt-budgets
interface CreateDebtBudgetRequest {
  repositoryId?: string;         // null = org-wide
  periodStart: string;           // ISO date
  periodEnd: string;
  budgetMinutes: number;
}

// GET /api/v1/debt-budgets
// GET /api/v1/debt-budgets/:id
// PATCH /api/v1/debt-budgets/:id  (adjust budget, add/complete tasks)

// POST /api/v1/debt-budgets/:id/tasks
interface CreateDebtTaskRequest {
  findingId?: string;
  hotspotId?: string;
  assignedTo?: string;           // user ID; null = auto-assign
  estimatedMinutes: number;
}

// PATCH /api/v1/debt-budgets/:id/tasks/:taskId
interface UpdateDebtTaskRequest {
  status?: 'pending' | 'in_progress' | 'completed' | 'skipped';
  actualMinutes?: number;
}
```
**Testing**:
- `budget_create`: Create budget with 480 minutes; GET returns it
- `budget_task_create`: Add task to budget; task appears in budget.tasks JSONB array
- `budget_task_complete`: Completing a task updates `spent_minutes` on the budget
- `budget_overspend_warning`: Completing tasks that exceed budgetMinutes triggers a warning (not a block)
- `budget_period_query`: GET with period filter returns budgets within date range

#### 8.2 — Developer Affinity-Based Auto-Assignment
**What**: Automatically assign debt tasks to developers based on their commit history affinity to the relevant files.
**Design**:
```typescript
// apps/api/src/services/debt-budget-planner.ts
interface AssignmentResult {
  taskId: string;
  assignedTo: string;            // user ID
  assignmentReason: string;      // e.g., "Most commits (45) to this file in last 90 days"
  affinityScore: number;         // 0-1
}

export class DebtBudgetPlanner {
  async autoAssignTasks(budgetId: string): Promise<AssignmentResult[]>;

  // Assignment algorithm:
  // 1. For each unassigned task, get the target file
  // 2. Query source_files.vcs_stats.top_contributors for that file
  // 3. Match contributors to organization members by email
  // 4. Assign to the member with highest affinity score who has remaining capacity
  // 5. Capacity = budget minutes allocated to user - minutes already assigned
  // 6. If no member has affinity, assign to the team lead
}
```
**Testing**:
- `assign_highest_affinity`: Developer with most commits to the file gets assigned
- `assign_capacity_limit`: Developer at capacity gets skipped; next-best developer assigned
- `assign_fallback`: File with no contributor matches assigns to org admin
- `assign_reason_text`: Assignment reason includes commit count and affinity score
- `assign_idempotent`: Running auto-assign twice doesn't reassign already-assigned tasks

#### 8.3 — Debt Reduction Dashboards and Trend Data
**What**: Persist daily trend snapshots and expose APIs for debt reduction visualizations.
**Design**:
```typescript
// Scheduled job: runs daily at midnight UTC
// Captures current state into trend_snapshots table

// GET /api/v1/repositories/:repoId/trends
interface TrendQuery {
  startDate: string;
  endDate: string;
  snapshotType?: 'daily' | 'weekly' | 'monthly';
}

// Response: TrendSnapshot[] with JSONB data per the schema

// GET /api/v1/organizations/:orgId/debt-summary
interface DebtSummaryResponse {
  totalDebtMinutes: number;
  debtByCharacteristic: Record<QualityCharacteristic, number>;
  debtByRepository: Array<{ repoId: string; name: string; debtMinutes: number }>;
  trendDirection: 'improving' | 'stable' | 'degrading';
  budgetUtilization: number;     // percentage of allocated budget spent
  topHotspots: Array<{ filePath: string; repoName: string; hotspotScore: number }>;
}
```
**Testing**:
- `trend_snapshot_daily`: Daily job creates snapshot with correct debt totals
- `trend_query_range`: Date range query returns only snapshots within range
- `trend_direction`: Decreasing debt over 30 days produces trendDirection 'improving'
- `debt_summary_aggregation`: Org-level summary correctly sums across repositories
- `budget_utilization`: 240 spent of 480 budget returns 50% utilization

---

## Phase 9: Web Dashboard (Next.js Frontend)

### Purpose
Build the interactive web dashboard for visualizing hotspots, managing findings, reviewing transformations, and monitoring debt reduction.

### Tasks

#### 9.1 — Authentication and Layout
**What**: OAuth login flow (GitHub/GitLab), protected routes, and dashboard layout with navigation.
**Design**:
```typescript
// apps/web/src/app/(auth)/login/page.tsx
// OAuth redirect to GitHub/GitLab
// Callback handler stores JWT in httpOnly cookie

// apps/web/src/app/(dashboard)/layout.tsx
// Sidebar navigation: Repositories, Hotspots, Findings, Transformations, Campaigns, Debt Budget, Settings
// Top bar: org switcher, user menu, notification bell
// Uses shadcn/ui Sidebar component

// apps/web/src/lib/api-client.ts
// Typed fetch wrapper using packages/types interfaces
// Automatic JWT inclusion from cookie
// Error handling with toast notifications
```
**Testing**:
- `auth_github_flow`: GitHub OAuth redirects correctly and creates session
- `auth_protected_routes`: Unauthenticated access to dashboard redirects to /login
- `layout_navigation`: All sidebar links render and navigate correctly
- `layout_responsive`: Dashboard renders correctly at 1280px and 768px widths

#### 9.2 — Hotspot Visualization
**What**: Interactive hotspot map showing files as sized/colored rectangles based on hotspot score.
**Design**:
```typescript
// apps/web/src/components/hotspot-map.tsx
// Treemap visualization where:
// - Rectangle size = file LOC
// - Rectangle color = hotspot score (green -> yellow -> red gradient)
// - Click opens file detail with findings, coupling, and contributors
// Uses recharts Treemap or D3 treemap layout

// apps/web/src/app/(dashboard)/hotspots/page.tsx
// Server component fetches hotspot data
// Client component renders interactive treemap
// Filters: language, min hotspot score, time window
// Sort: by hotspot score, by churn, by complexity
```
**Testing**:
- `hotspot_map_renders`: Treemap renders with correct number of rectangles
- `hotspot_color_gradient`: Highest score file is red; lowest is green
- `hotspot_click_detail`: Clicking a rectangle opens file detail panel
- `hotspot_filter_language`: Language filter updates treemap to show only matching files
- `hotspot_empty_state`: Repository with no hotspots shows helpful empty state message

#### 9.3 — Findings List and Detail Views
**What**: Paginated, filterable findings list with inline code snippets and finding detail modal.
**Design**:
```typescript
// apps/web/src/app/(dashboard)/findings/page.tsx
// Server component with search params for filters
// DataTable with columns: severity icon, rule, file, line, status, assignee, remediation time
// Filters: severity, type, status, characteristic, CWE, file path
// Bulk actions: assign, mark as wont_fix, mark as false_positive

// apps/web/src/components/finding-detail.tsx
// Modal/drawer showing:
// - Code snippet with highlighted issue lines
// - Rule description and CWE reference
// - AI explanation (if available)
// - Related findings
// - Data flow visualization (for vulnerability findings)
// - Quick actions: generate refactoring, assign, change status
```
**Testing**:
- `findings_list_loads`: Page renders with finding data from API
- `findings_filter_updates_url`: Applying severity filter updates URL search params
- `findings_pagination`: Navigating pages loads correct data
- `findings_bulk_assign`: Selecting multiple findings and assigning updates all
- `finding_detail_code_snippet`: Detail view shows code with highlighted lines

#### 9.4 — Transformation Review and Campaign Dashboard
**What**: Interface for reviewing transformation diffs, approving/rejecting, and monitoring campaign progress.
**Design**:
```typescript
// apps/web/src/app/(dashboard)/transformations/page.tsx
// List of transformations with status badges
// Filter by status, campaign, repository

// apps/web/src/components/diff-viewer.tsx
// Side-by-side or unified diff view using react-diff-viewer-continued
// Shows: before/after code, AI explanation, risk score, test results

// apps/web/src/app/(dashboard)/campaigns/[id]/page.tsx
// Campaign dashboard with:
// - Progress bar (completed / total transformations)
// - Status distribution chart (pie: pending, generating, testing, ready, applied, failed)
// - List of transformations with quick approve/reject actions
// - Debt reduction counter
```
**Testing**:
- `diff_viewer_renders`: Diff viewer shows before/after code correctly
- `transform_approve_button`: Clicking approve calls API and updates status
- `campaign_progress_bar`: Progress bar shows correct percentage
- `campaign_live_update`: Progress updates when transformation completes (polling or WebSocket)

#### 9.5 — Debt Budget Dashboard
**What**: Visualization of debt reduction trends, budget utilization, and task assignment overview.
**Design**:
```typescript
// apps/web/src/app/(dashboard)/debt-budget/page.tsx
// - Current sprint budget card: allocated vs. spent minutes with progress ring
// - Debt trend chart: area chart showing debt over time (from trend_snapshots)
// - Debt by characteristic: stacked bar chart (maintainability, reliability, security, performance)
// - Task list: assigned tasks with status, estimates, and developer avatars
// - Hotspot improvement: before/after hotspot scores for recently refactored files

// apps/web/src/components/debt-trend-chart.tsx
// Recharts AreaChart with:
// - X axis: date range (30/90/180 days)
// - Y axis: debt minutes
// - Stacked areas by quality characteristic
// - Annotations for campaign completions
```
**Testing**:
- `debt_trend_chart_renders`: Chart renders with data from API
- `debt_budget_card`: Shows correct allocated/spent ratio
- `debt_task_list`: Tasks display with correct assignee and status
- `debt_time_range`: Switching between 30/90/180 days updates chart

---

## Phase 10: Polyglot and Cross-Language Support

### Purpose
Extend the platform to handle mixed-language monorepos, cross-language refactoring (e.g., API contract changes across Go services and TypeScript clients), and language-specific recipe libraries.

### Tasks

#### 10.1 — Language Detection and Multi-Language Analysis
**What**: Automatically detect languages in a repository and run appropriate analyzers for each.
**Design**:
```typescript
// apps/api/src/services/language-detector.ts
interface LanguageBreakdown {
  languages: Record<string, { files: number; loc: number; percentage: number }>;
  primaryLanguage: string;
  buildSystems: string[];
  testFrameworks: string[];
  frameworkVersions: Record<string, string>;
}

export class LanguageDetector {
  async detectLanguages(repoPath: string): Promise<LanguageBreakdown>;

  // Detection based on:
  // 1. File extensions (.ts, .py, .java, .go, .rs, .cs, .php)
  // 2. Build files (package.json, pyproject.toml, pom.xml, go.mod, Cargo.toml, *.csproj)
  // 3. Framework markers (next.config.js, django settings, spring application.yml)
  // 4. Test framework markers (jest.config, pytest.ini, build.gradle testImplementation)
}
```
**Testing**:
- `detect_monorepo`: Repository with Go, TypeScript, and Python correctly identifies all three
- `detect_frameworks`: Next.js project identified from next.config.js
- `detect_test_frameworks`: Jest, pytest, JUnit detected from config files
- `detect_primary_language`: Language with most LOC is primaryLanguage

#### 10.2 — Cross-Language Impact Analysis
**What**: Trace API contract changes across language boundaries (e.g., Go gRPC server -> TypeScript client).
**Design**:
```typescript
// apps/api/src/services/cross-language-impact.ts
interface CrossLanguageImpact {
  sourceFile: { path: string; language: string };
  apiContract: {
    type: 'rest' | 'grpc' | 'graphql';
    changedEndpoints: string[];
  };
  impactedConsumers: Array<{
    filePath: string;
    language: string;
    usageType: 'client_call' | 'type_import' | 'schema_reference';
    changeRequired: boolean;
  }>;
}

// Uses LLM to understand API contracts across:
// - OpenAPI specs and their generated clients
// - gRPC proto files and generated stubs
// - GraphQL schemas and client queries
// - Shared type definition files
```
**Testing**:
- `cross_lang_rest_api`: Changing a Go API endpoint identifies affected TypeScript fetch calls
- `cross_lang_proto`: Changing a gRPC proto field identifies affected client stubs
- `cross_lang_graphql`: Changing a GraphQL schema identifies affected query files
- `cross_lang_shared_types`: Changing a shared TypeScript type identifies all consumers

#### 10.3 — Language-Specific Recipe Libraries
**What**: Pre-built recipe collections for common migration patterns per language.
**Design**:
```typescript
// Seed data: built-in recipes
const BUILT_IN_RECIPES: RecipeDefinition[] = [
  // TypeScript
  { name: 'React Class to Hooks', sourceLanguage: 'typescript', recipeType: 'framework_migration', ... },
  { name: 'Next.js Pages to App Router', sourceLanguage: 'typescript', recipeType: 'framework_migration', ... },
  { name: 'CommonJS to ESM', sourceLanguage: 'typescript', recipeType: 'code_quality', ... },

  // Python
  { name: 'Python 2 to 3', sourceLanguage: 'python', recipeType: 'framework_migration', ... },
  { name: 'Django 3 to 5', sourceLanguage: 'python', recipeType: 'framework_migration', ... },
  { name: 'unittest to pytest', sourceLanguage: 'python', recipeType: 'code_quality', ... },

  // Java
  { name: 'JUnit 4 to 5', sourceLanguage: 'java', recipeType: 'framework_migration', ... },
  { name: 'Spring Boot 2 to 3', sourceLanguage: 'java', recipeType: 'framework_migration', ... },
  { name: 'Java 11 to 21', sourceLanguage: 'java', recipeType: 'framework_migration', ... },

  // Go
  { name: 'Go Module Migration', sourceLanguage: 'go', recipeType: 'dependency_upgrade', ... },
  { name: 'Error Wrapping (Go 1.13+)', sourceLanguage: 'go', recipeType: 'code_quality', ... },
];
```
**Testing**:
- `recipes_seeded`: All built-in recipes exist after seed script
- `recipe_precondition_match`: React Class to Hooks recipe matches files importing React Component
- `recipe_language_filter`: Filtering by sourceLanguage='python' returns only Python recipes

---

## Phase 11: Compliance Reporting and Audit

### Purpose
Implement NIST SSDF compliance reporting, audit trail export, and ISO 25010 quality dashboards for regulated enterprise customers.

### Tasks

#### 11.1 — Audit Trail with SSDF Practice Mapping
**What**: Comprehensive audit log entries for all automated transformations, mapped to NIST SP 800-218 practices.
**Design**:
```typescript
// apps/api/src/services/audit-service.ts
interface AuditEntry {
  action: string;                // 'transformation.applied', 'finding.resolved', 'campaign.completed'
  resourceType: string;
  resourceId: string;
  context: {
    ssdpPractice?: string;       // 'PW.5.1', 'PW.7.2', 'RV.1.1'
    repository?: string;
    filePath?: string;
    preTestStatus?: string;
    postTestStatus?: string;
    riskScore?: number;
    commitSha?: string;
    prUrl?: string;
    debtReducedMinutes?: number;
  };
}

// SSDF practice mappings:
// PW.5 (Create Source Code): Automated refactoring with test validation
// PW.7 (Review and/or Analyze Code): Automated analysis runs
// PW.8 (Test Executable Code): Test-anchored execution
// RV.1 (Identify and Confirm Vulnerabilities): Security finding detection

// GET /api/v1/audit-log
// Filters: action, resource_type, date range, actor
// Export: CSV, JSON
```
**Testing**:
- `audit_transformation_logged`: Applying a transformation creates audit entry with SSDF mapping
- `audit_finding_resolution_logged`: Resolving a finding creates audit entry
- `audit_export_csv`: CSV export contains all required columns
- `audit_date_range_filter`: Date range filter returns correct entries

#### 11.2 — ISO 25010 Quality Dashboard
**What**: Map all findings and debt metrics to ISO/IEC 25010:2023 quality characteristics and sub-characteristics.
**Design**:
```typescript
// GET /api/v1/repositories/:repoId/quality-report
interface QualityReport {
  standard: 'ISO/IEC 25010:2023';
  characteristics: Array<{
    code: QualityCharacteristic;
    name: string;
    rating: 'A' | 'B' | 'C' | 'D' | 'E';
    debtMinutes: number;
    findingCount: number;
    subCharacteristics: Array<{
      code: string;
      name: string;
      rating: string;
      debtMinutes: number;
    }>;
  }>;
  overallRating: string;
  complianceDate: string;
}

// Sub-characteristic mappings under maintainability:
// modularity: coupling findings, large file findings
// reusability: duplicate code findings
// analysability: complexity findings, naming violations
// modifiability: hotspot files, tightly coupled files
// testability: untested code, complex test setup
```
**Testing**:
- `quality_report_all_characteristics`: Report includes all 4 ISO 5055 characteristics
- `quality_rating_calculation`: Debt ratio correctly maps to A-E rating (SQALE method)
- `quality_sub_characteristics`: Each characteristic includes its sub-characteristics
- `quality_report_stable`: Same data produces identical report on repeated calls

#### 11.3 — Compliance Export (PDF/HTML)
**What**: Generate downloadable compliance reports in PDF and HTML formats for auditors.
**Design**:
```typescript
// GET /api/v1/repositories/:repoId/compliance-report
// Accept: application/pdf | text/html
// Generates:
// 1. Executive summary with overall quality rating
// 2. ISO 25010 quality breakdown
// 3. NIST SSDF practice compliance evidence
// 4. CWE/OWASP finding summary
// 5. Transformation audit trail
// 6. Trend charts (30/90/180 day)
```
**Testing**:
- `compliance_pdf_generated`: Endpoint returns valid PDF document
- `compliance_html_generated`: Endpoint returns valid HTML document
- `compliance_includes_audit`: Report includes transformation audit trail
- `compliance_includes_trends`: Report includes embedded trend charts

---

## Phase 12: Production Hardening, Performance, and Scalability

### Purpose
Optimize for production workloads: database performance tuning, caching, rate limiting, monitoring, and horizontal scaling.

### Tasks

#### 12.1 — Database Performance Optimization
**What**: Add table partitioning, materialized views, and query optimization for large-scale deployments.
**Design**:
```sql
-- Partition audit_log by month
ALTER TABLE audit_log PARTITION BY RANGE (occurred_at);
CREATE TABLE audit_log_2026_05 PARTITION OF audit_log
  FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- Materialized view for repository-level debt summary
CREATE MATERIALIZED VIEW mv_repo_debt_summary AS
SELECT
  r.id AS repository_id,
  r.name,
  COUNT(f.id) FILTER (WHERE f.status = 'open') AS open_findings,
  SUM(f.remediation_minutes) FILTER (WHERE f.status = 'open') AS total_debt_minutes,
  COUNT(f.id) FILTER (WHERE f.severity = 'critical' AND f.status = 'open') AS critical_count,
  MAX(sf.hotspot_score) AS max_hotspot_score
FROM repositories r
LEFT JOIN findings f ON f.repository_id = r.id
LEFT JOIN source_files sf ON sf.repository_id = r.id
GROUP BY r.id, r.name;

-- Refresh nightly via pg_cron or application job
```
**Testing**:
- `partition_query_performance`: Query on audit_log with date filter uses partition pruning (EXPLAIN)
- `materialized_view_refresh`: View refresh completes within 30 seconds for 1M findings
- `query_plan_optimal`: Key queries (hotspot listing, finding search) use index scans

#### 12.2 — Redis Caching Layer
**What**: Cache frequently-accessed read queries (hotspot lists, debt summaries, quality gate profiles).
**Design**:
```typescript
// apps/api/src/plugins/cache.ts
interface CacheConfig {
  defaultTtlSeconds: number;     // default: 300 (5 min)
  hotspotTtl: number;            // 600 (10 min)
  debtSummaryTtl: number;        // 300
  qualityGateTtl: number;        // 3600 (1 hour)
}

// Cache invalidation:
// - Analysis completion invalidates: hotspots, findings, debt summary for that repo
// - Transformation applied invalidates: findings, debt summary
// - Quality gate update invalidates: quality gate profiles
```
**Testing**:
- `cache_hit`: Second request for hotspots returns cached result (verify via response header)
- `cache_invalidation`: Completing an analysis invalidates hotspot cache for that repo
- `cache_miss_populates`: First request populates cache; verify via Redis GET

#### 12.3 — Rate Limiting and Request Throttling
**What**: Per-user and per-org rate limits for API endpoints, with separate limits for LLM-intensive operations.
**Design**:
```typescript
// apps/api/src/plugins/rate-limit.ts
// Standard API: 1000 requests/minute per user
// Analysis triggers: 10/hour per repository
// LLM operations (refactoring, test generation): 50/hour per org
// Webhook endpoints: 100/minute per org

// Returns 429 with Retry-After header
```
**Testing**:
- `rate_limit_standard`: 1001st request within a minute returns 429
- `rate_limit_llm`: 51st LLM operation within an hour returns 429
- `rate_limit_retry_after`: 429 response includes Retry-After header with correct value

#### 12.4 — Observability (Metrics, Logging, Tracing)
**What**: Structured logging, Prometheus metrics, and distributed tracing for production monitoring.
**Design**:
```typescript
// Metrics (Prometheus):
// refactor_analysis_duration_seconds (histogram)
// refactor_findings_total (counter, labels: severity, type)
// refactor_transformations_total (counter, labels: status, recipe_type)
// refactor_llm_latency_seconds (histogram)
// refactor_llm_cache_hit_rate (gauge)
// refactor_active_campaigns (gauge)
// refactor_debt_minutes_total (gauge, labels: repo, characteristic)

// GET /metrics (Prometheus scrape endpoint)

// Structured logging: JSON format with correlation IDs
// Tracing: OpenTelemetry with Jaeger export
```
**Testing**:
- `metrics_endpoint`: `GET /metrics` returns valid Prometheus exposition format
- `metrics_analysis_duration`: Completing an analysis records a histogram observation
- `metrics_finding_counter`: Detecting findings increments the counter with correct labels
- `tracing_correlation`: Analysis run creates a trace span visible in Jaeger

#### 12.5 — CI/CD Pipeline and Docker Production Build
**What**: GitHub Actions pipeline for testing, building, and publishing Docker images.
**Design**:
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_DB: test_db, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
      redis:
        image: redis:7-alpine
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 22 }
      - run: npm ci
      - run: turbo build
      - run: turbo test
      - run: turbo lint
      - run: turbo typecheck

  docker:
    needs: test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: ${{ github.ref == 'refs/heads/main' }}
          tags: ghcr.io/${{ github.repository }}:latest

# Dockerfile: multi-stage build
# Stage 1: Install dependencies and build
# Stage 2: Production image with only dist/ and node_modules (production)
```
**Testing**:
- `ci_pipeline_passes`: All CI jobs pass on a clean checkout
- `docker_build_succeeds`: Multi-stage Docker build produces a working image
- `docker_image_size`: Production image is under 500MB
- `docker_healthcheck`: Container healthcheck passes within 30 seconds

---

## Phase Summary & Dependencies

```
Phase 1: Foundation
  └─> Phase 2: VCS Ingestion & Behavioral Analysis
       └─> Phase 3: Hotspot Detection & Static Analysis
            ├─> Phase 4: LLM Semantic Refactoring
            │    └─> Phase 5: Test-Anchored Safe Execution
            │         └─> Phase 7: Campaigns & Batch Execution
            ├─> Phase 6: Quality Gates & PR Integration
            └─> Phase 8: Technical Debt Budgeting
                 └─> Phase 9: Web Dashboard
                      └─> Phase 10: Polyglot & Cross-Language
                           └─> Phase 11: Compliance Reporting
                                └─> Phase 12: Production Hardening
```

| Phase | Depends On | Produces |
|-------|-----------|----------|
| 1: Foundation | — | Monorepo, database, types, API scaffold |
| 2: VCS Ingestion | Phase 1 | Git history, churn scores, complexity metrics, file coupling |
| 3: Hotspot Detection | Phase 2 | Hotspots, detection rules, findings, SARIF export, analysis API |
| 4: LLM Refactoring | Phase 3 | Semantic refactoring, transformation plans, impact analysis |
| 5: Test-Anchored Execution | Phase 4 | Characterization tests, safe refactoring pipeline, safety verdicts |
| 6: Quality Gates & PR | Phase 3 | Quality gate profiles, PR checks, inline annotations |
| 7: Campaigns | Phase 5 | Batch transformations, campaign orchestration, automated PRs |
| 8: Debt Budgeting | Phase 3 | Budget allocation, auto-assignment, debt tracking |
| 9: Web Dashboard | Phase 3, 8 | Hotspot map, findings UI, transformation review, debt charts |
| 10: Polyglot | Phase 7 | Multi-language analysis, cross-language impact, recipe libraries |
| 11: Compliance | Phase 8, 10 | NIST SSDF audit, ISO 25010 reports, compliance exports |
| 12: Production | Phase 9, 11 | Caching, rate limiting, observability, CI/CD, Docker |

---

## Definition of Done (per phase)

Each phase is considered complete when:

1. **All tasks implemented**: Every task in the phase has working code merged to the main branch.
2. **All tests passing**: Every named test scenario passes in CI. Unit test coverage for new code >= 80%.
3. **API documentation current**: OpenAPI spec at `/docs` reflects all new endpoints with request/response schemas.
4. **Database migrations clean**: `drizzle-kit migrate` applies without errors on a fresh database. No manual SQL required.
5. **Type safety enforced**: `turbo typecheck` passes with zero errors across all workspaces.
6. **Linting clean**: `turbo lint` passes with zero warnings.
7. **Docker builds**: `docker compose up` starts the full local environment and all health checks pass.
8. **No regressions**: All tests from previous phases continue to pass.
