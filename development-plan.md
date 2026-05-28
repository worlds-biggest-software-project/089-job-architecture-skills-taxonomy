# Job Architecture & Skills Taxonomy — Development Plan

> Project: #089 · Created: 2026-05-25

---

## Technology Decisions

### Decision 1: Data Model — Hybrid Relational + JSONB (Model 3) with Graph Extension Path

**Rationale:** Model 3 (Hybrid Relational + JSONB) is selected as the primary data model for the following reasons:

- **Fastest path to MVP.** ~10-12 core tables versus ~45-50 for fully normalised Model 1. JSONB columns absorb the variability of multi-industry, multi-region job profile attributes without schema migrations.
- **Multi-taxonomy flexibility.** Lightcast, O\*NET, ESCO, and SFIA identifiers are relational columns for joins; taxonomy-specific metadata lives in JSONB. New taxonomy sources can be added without DDL.
- **Generated columns bridge the gap.** Frequently-queried JSONB paths (e.g., `demand_trend`) are promoted to indexed generated columns, giving relational-speed queries where needed.
- **Graph extension path (from Model 2).** A `skill_relationship` table (source_skill_id, target_skill_id, relationship_type, weight) is added in Phase 5 to support skills similarity, adjacency, and prerequisite queries without the full graph_node/graph_edge infrastructure. If graph traversal needs grow, the architecture can migrate to PostgreSQL Apache AGE or Neo4j.
- **Event audit from Model 4 adopted selectively.** Rather than full event sourcing (high implementation cost), a lightweight `audit_event` table captures taxonomy changes, AI proposals, and skill assessment history. This satisfies ISO 30414 audit trail requirements without CQRS projection infrastructure.

**Rejected alternatives:**
- Model 1 (Normalized Relational): Too many tables, too many migrations, slow to iterate during MVP.
- Model 2 (Graph-Relational Hybrid): Graph layer adds query complexity the team doesn't need until Phase 5+. The simpler `skill_relationship` table covers early needs.
- Model 4 (Event-Sourced): Implementation complexity is disproportionate to MVP requirements. Audit trail needs are met by the lightweight audit table.

### Decision 2: Tech Stack

| Layer | Choice | Rationale |
|-------|--------|-----------|
| **Language** | TypeScript (Node.js) | Shared language front-to-back; largest OSS contributor pool; strong ecosystem for API development |
| **API Framework** | Fastify + tRPC | Fastify for raw HTTP performance and OpenAPI plugin; tRPC for type-safe internal client when paired with a TypeScript frontend |
| **Database** | PostgreSQL 16+ | JSONB + GIN indexes, `pg_trgm` for fuzzy skill search, `ltree` extension reserved for future hierarchy queries, row-level security for multi-tenancy |
| **ORM / Query** | Drizzle ORM | Type-safe schema-as-code, lightweight, excellent PostgreSQL JSONB support, generates migrations |
| **Frontend** | Next.js 15 (App Router) | React Server Components for taxonomy browsing, server actions for mutations, built-in API routes for simple deployments |
| **UI Components** | shadcn/ui + Tailwind CSS | Accessible, composable, no vendor lock-in; theming for white-label enterprise deployments |
| **Authentication** | NextAuth.js v5 + OAuth 2.0 | Supports OIDC providers (Okta, Azure AD, Google) for enterprise SSO; built-in session management |
| **Background Jobs** | BullMQ (Redis) | Taxonomy import, HRIS sync, AI inference, and export jobs run as background workers |
| **AI Integration** | Anthropic Claude API (primary) + OpenAI-compatible fallback | Skills inference, role profiling, taxonomy maintenance agent; provider-agnostic interface |
| **Search** | PostgreSQL full-text + pg_trgm (MVP) → Typesense (scale) | Fuzzy skill search with trigram indexes is sufficient for 34,000 skills; Typesense added if sub-10ms latency is needed at scale |
| **File Storage** | S3-compatible (MinIO self-hosted / AWS S3) | CSV/HR-XML export artifacts, bulk import files |
| **Containerisation** | Docker Compose (dev) / Kubernetes Helm chart (prod) | Self-hostable target; Docker Compose for local development |
| **CI/CD** | GitHub Actions | Lint, test, build, deploy pipeline; automated migration checks |
| **Monorepo** | Turborepo | Shared packages (types, validators, db schema) between API, frontend, and worker |

### Decision 3: Project Structure

```
job-architecture-skills-taxonomy/
├── apps/
│   ├── web/                    # Next.js 15 frontend
│   │   ├── src/
│   │   │   ├── app/            # App Router pages
│   │   │   ├── components/     # UI components (shadcn/ui based)
│   │   │   ├── lib/            # Client utilities
│   │   │   └── hooks/          # React hooks
│   │   └── ...
│   ├── api/                    # Fastify API server
│   │   ├── src/
│   │   │   ├── routes/         # REST API routes
│   │   │   ├── services/       # Business logic
│   │   │   ├── middleware/     # Auth, tenant resolution, RLS
│   │   │   └── plugins/        # Fastify plugins
│   │   └── ...
│   └── worker/                 # BullMQ background workers
│       ├── src/
│       │   ├── jobs/           # Job handlers (import, sync, AI, export)
│       │   └── processors/    # Job-specific processing logic
│       └── ...
├── packages/
│   ├── db/                     # Drizzle schema, migrations, seed data
│   │   ├── schema/             # Table definitions
│   │   ├── migrations/         # SQL migrations
│   │   └── seed/               # Taxonomy seed data (Lightcast, O*NET)
│   ├── types/                  # Shared TypeScript types
│   ├── validators/             # Zod schemas for API validation
│   └── taxonomy-loader/        # ETL utilities for Lightcast/O*NET/ESCO import
├── docker-compose.yml
├── turbo.json
└── package.json
```

---

## Phase Dependency Graph

```
Phase 1 ──→ Phase 2 ──→ Phase 3 ──→ Phase 5 ──→ Phase 7 ──→ Phase 9
  │            │            │                        │
  │            │            └──→ Phase 4             └──→ Phase 8
  │            │                   │
  │            └──→ Phase 6 ◄──────┘
  │
  └──→ (Phase 10 can start after Phase 2)

Legend:
  ──→  "must complete before"
  Phase 10 (Deployment & DevOps) can begin in parallel after Phase 2
```

| Phase | Depends On | Can Run In Parallel With |
|-------|-----------|-------------------------|
| 1. Foundation & Schema | — | — |
| 2. Skills Taxonomy Engine | 1 | — |
| 3. Job Architecture Builder | 2 | — |
| 4. Employee Profiles & Gap Analysis | 3 | — |
| 5. Skills Relationships & Intelligence | 3 | 4 |
| 6. HRIS Integration Layer | 2, 4 | 5 |
| 7. AI-Assisted Role Profiling | 5 | 6 |
| 8. Career Pathways & Visualisation | 7 | — |
| 9. Self-Maintaining Taxonomy Agent | 7 | 8 |
| 10. Deployment, DevOps & Hardening | 2 | 3-9 |

---

## Phase 1: Foundation & Schema

**Goal:** Establish the monorepo, database schema, authentication, and multi-tenant infrastructure so all subsequent phases build on a stable base.

**Definition of Done:** A developer can clone the repo, run `docker compose up`, hit the health endpoint, and see an authenticated multi-tenant API with an empty database that passes all migration checks.

### Task 1.1: Monorepo Scaffold

**What:** Initialise the Turborepo monorepo with `apps/web`, `apps/api`, `apps/worker`, and `packages/db`, `packages/types`, `packages/validators`. Configure TypeScript, ESLint, Prettier, and Husky pre-commit hooks.

**Design:**
```bash
# Root turbo.json
{
  "pipeline": {
    "build": { "dependsOn": ["^build"] },
    "dev": { "cache": false, "persistent": true },
    "lint": {},
    "test": {},
    "db:migrate": { "cache": false }
  }
}
```
```typescript
// packages/types/src/index.ts
export type TenantId = string & { readonly __brand: 'TenantId' };
export type SkillId = string & { readonly __brand: 'SkillId' };
export type JobProfileId = string & { readonly __brand: 'JobProfileId' };
export type EmployeeId = string & { readonly __brand: 'EmployeeId' };
```

**Testing:**
- `turbo build` completes without errors across all workspaces
- `turbo lint` passes with zero warnings
- Each workspace's `tsconfig.json` resolves shared package imports correctly
- `packages/types` is importable from `apps/api` and `apps/web`

### Task 1.2: Database Schema — Core Tables

**What:** Implement the Drizzle ORM schema for the core tables: `tenant`, `skill`, `job_family`, `job_level`, `job_profile`, `career_pathway`, `employee`, `hris_connection`, `data_job`, `app_user`, `audit_event`. Create the initial migration.

**Design:**
```typescript
// packages/db/schema/tenant.ts
import { pgTable, uuid, text, char, jsonb, timestamp, boolean } from 'drizzle-orm/pg-core';

export const tenant = pgTable('tenant', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  slug: text('slug').notNull().unique(),
  countryCode: char('country_code', { length: 2 }),
  industry: text('industry'),
  settings: jsonb('settings').notNull().default('{}'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});

// packages/db/schema/skill.ts
export const skill = pgTable('skill', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  name: text('name').notNull(),
  skillType: text('skill_type').notNull(), // 'specialised' | 'common' | 'software' | 'certification'
  category: text('category'),
  subcategory: text('subcategory'),
  source: text('source').notNull().default('lightcast'),
  isActive: boolean('is_active').notNull().default(true),
  lightcastId: text('lightcast_id'),
  onetElementId: text('onet_element_id'),
  escoUri: text('esco_uri'),
  sfiaCode: text('sfia_code'),
  attributes: jsonb('attributes').notNull().default('{}'),
  isEmerging: boolean('is_emerging').notNull().default(false),
  deprecatedAt: timestamp('deprecated_at', { withTimezone: true }),
  aiConfidence: text('ai_confidence'), // numeric stored as text for Drizzle compatibility
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
  updatedAt: timestamp('updated_at', { withTimezone: true }).notNull().defaultNow(),
});
```

**Testing:**
- `drizzle-kit generate` produces a valid SQL migration
- `drizzle-kit push` applies the migration to a clean PostgreSQL 16 database without errors
- All tables have correct indexes (verify via `\d+ table_name` in psql)
- Row-level security policies enforce tenant isolation: inserting a row with `tenant_id = A` while `app.current_tenant_id = B` is blocked
- Foreign key constraints are enforced: inserting a `skill` with a non-existent `tenant_id` fails

### Task 1.3: Fastify API Server with Auth & Tenant Resolution

**What:** Set up the Fastify server with OpenAPI 3.1 generation, NextAuth.js v5 session validation, and tenant resolution middleware that sets `app.current_tenant_id` for RLS.

**Design:**
```typescript
// apps/api/src/middleware/tenant.ts
import { FastifyRequest } from 'fastify';

export async function resolveTenant(request: FastifyRequest) {
  const tenantSlug = request.headers['x-tenant-slug'] as string;
  if (!tenantSlug) throw { statusCode: 400, message: 'Missing X-Tenant-Slug header' };

  const tenant = await request.server.db
    .select()
    .from(tenantTable)
    .where(eq(tenantTable.slug, tenantSlug))
    .limit(1);

  if (!tenant.length) throw { statusCode: 404, message: 'Tenant not found' };

  // Set PostgreSQL session variable for RLS
  await request.server.db.execute(
    sql`SET LOCAL app.current_tenant_id = ${tenant[0].id}`
  );

  request.tenantId = tenant[0].id;
}
```

**Testing:**
- `GET /health` returns `200` with `{ status: 'ok', version: '0.1.0' }`
- `GET /api/docs` returns a valid OpenAPI 3.1 JSON document
- Requests without `X-Tenant-Slug` return `400`
- Requests with an invalid tenant slug return `404`
- Authenticated requests with a valid tenant slug resolve correctly and set the RLS context
- Unauthenticated requests to protected routes return `401`

### Task 1.4: Next.js Frontend Shell

**What:** Set up the Next.js 15 app with App Router, shadcn/ui, NextAuth.js sign-in flow, and a sidebar layout with navigation stubs for: Dashboard, Taxonomy, Job Architecture, Employees, Gap Analysis, Integrations, Settings.

**Design:**
```typescript
// apps/web/src/app/layout.tsx
export default function RootLayout({ children }: { children: React.ReactNode }) {
  return (
    <html lang="en">
      <body>
        <SessionProvider>
          <TenantProvider>
            <div className="flex h-screen">
              <Sidebar />
              <main className="flex-1 overflow-auto p-6">{children}</main>
            </div>
          </TenantProvider>
        </SessionProvider>
      </body>
    </html>
  );
}
```

**Testing:**
- `next dev` starts without errors
- Sign-in flow redirects to the configured OAuth provider and returns a valid session
- Sidebar navigation renders all seven sections
- Each section route renders a placeholder page without 404
- Responsive layout: sidebar collapses to a hamburger menu on mobile viewports
- Lighthouse accessibility score >= 90 on the shell layout

### Task 1.5: Docker Compose Development Environment

**What:** Create a `docker-compose.yml` with PostgreSQL 16, Redis 7, MinIO, and the three app services (web, api, worker). Include a `Makefile` with `make dev`, `make test`, `make migrate`, `make seed` targets.

**Design:**
```yaml
# docker-compose.yml
services:
  postgres:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: jast
      POSTGRES_USER: jast
      POSTGRES_PASSWORD: jast_dev
    ports: ["5432:5432"]
    volumes: ["pgdata:/var/lib/postgresql/data"]
  redis:
    image: redis:7-alpine
    ports: ["6379:6379"]
  minio:
    image: minio/minio:latest
    command: server /data --console-address ":9001"
    ports: ["9000:9000", "9001:9001"]
  api:
    build: { context: ., dockerfile: apps/api/Dockerfile }
    depends_on: [postgres, redis]
  web:
    build: { context: ., dockerfile: apps/web/Dockerfile }
    depends_on: [api]
  worker:
    build: { context: ., dockerfile: apps/worker/Dockerfile }
    depends_on: [postgres, redis]
```

**Testing:**
- `docker compose up` brings up all services with health checks passing
- `make migrate` applies all migrations to the containerised PostgreSQL
- `make test` runs the test suite inside containers
- Tearing down and re-running `docker compose up` produces a clean state
- Services restart automatically if a container crashes

---

## Phase 2: Skills Taxonomy Engine

**Goal:** Import, store, search, and manage the open skills taxonomy (Lightcast + O\*NET + ESCO) with a browsable UI and API.

**Definition of Done:** A user can browse 34,000+ Lightcast skills by category, search with fuzzy matching, view O\*NET crosswalk mappings, and an admin can import/refresh taxonomy data via background jobs.

### Task 2.1: Taxonomy Loader — Lightcast Open Skills Import

**What:** Build the `packages/taxonomy-loader` ETL module that fetches the Lightcast Open Skills API, transforms the three-tier hierarchy (category > subcategory > skill) into the `skill` table schema, and runs as a BullMQ job.

**Design:**
```typescript
// packages/taxonomy-loader/src/lightcast.ts
export async function importLightcastSkills(tenantId: string, db: DrizzleClient) {
  const token = await getLightcastOAuthToken();
  const skills = await fetchAllLightcastSkills(token);

  // Batch upsert: 500 skills per transaction
  for (const batch of chunk(skills, 500)) {
    await db.transaction(async (tx) => {
      for (const skill of batch) {
        await tx.insert(skillTable).values({
          tenantId,
          name: skill.name,
          skillType: mapSkillType(skill.type), // 'Specialized' → 'specialised'
          category: skill.category?.name,
          subcategory: skill.subcategory?.name,
          source: 'lightcast',
          lightcastId: skill.id,
          attributes: {
            lightcast: {
              categoryId: skill.category?.id,
              subcategoryId: skill.subcategory?.id,
              type: skill.type,
              infoUrl: skill.infoUrl,
            },
          },
        }).onConflictDoUpdate({
          target: [skillTable.tenantId, skillTable.lightcastId],
          set: { name: skill.name, category: skill.category?.name, updatedAt: new Date() },
        });
      }
    });
  }
}
```

**Testing:**
- Import job processes all 34,000+ Lightcast skills without errors
- All 31 top-level categories are represented in the `category` column
- Skill types are correctly mapped: `specialised`, `common`, `software`
- Re-running the import is idempotent (upsert, no duplicates)
- Import completes within 60 seconds for the full taxonomy
- A `data_job` record is created with status `completed` and `records_processed` count

### Task 2.2: Taxonomy Loader — O\*NET and ESCO Import

**What:** Add O\*NET occupation import (1,000+ occupations with SOC codes, skills, abilities, tasks) and ESCO concept import (occupations and skills with multilingual labels) to the taxonomy loader. Store crosswalk mappings in the `skill.attributes` JSONB.

**Design:**
```typescript
// packages/taxonomy-loader/src/onet.ts
export async function importOnetOccupations(tenantId: string, db: DrizzleClient) {
  const occupations = await fetchOnetOccupations(apiKey);
  for (const occ of occupations) {
    await db.insert(skillTable).values({
      tenantId,
      name: occ.title,
      skillType: 'specialised',
      category: 'O*NET Occupation',
      source: 'onet',
      onetElementId: occ.code,
      attributes: {
        onet: {
          socCode: occ.code,
          jobZone: occ.jobZone,
          description: occ.description,
          tasks: occ.tasks,
          skills: occ.skills,       // O*NET's own skill descriptors
          abilities: occ.abilities,
        },
      },
    }).onConflictDoUpdate({
      target: [skillTable.tenantId, skillTable.onetElementId],
      set: { name: occ.title, attributes: sql`excluded.attributes`, updatedAt: new Date() },
    });
  }
}
```

**Testing:**
- O\*NET import loads 1,000+ occupations with valid SOC codes
- ESCO import loads concepts with URIs as `esco_uri`
- Crosswalk between Lightcast skills and O\*NET elements is stored in `attributes.onet_crosswalk`
- Searching for "software developer" returns results from both Lightcast and O\*NET sources
- ESCO multilingual labels are stored in `attributes.esco.translations` for at least English, French, German, Spanish

### Task 2.3: Skills Search API

**What:** Build the skills search API endpoint that supports fuzzy text search (via `pg_trgm`), filtering by category/type/source, and pagination. Returns skills with their taxonomy metadata and crosswalk identifiers.

**Design:**
```typescript
// apps/api/src/routes/skills.ts
app.get('/api/v1/skills', {
  schema: {
    querystring: z.object({
      q: z.string().optional(),
      category: z.string().optional(),
      skillType: z.enum(['specialised', 'common', 'software', 'certification']).optional(),
      source: z.enum(['lightcast', 'onet', 'esco', 'sfia', 'custom']).optional(),
      isActive: z.boolean().optional().default(true),
      page: z.number().int().min(1).optional().default(1),
      pageSize: z.number().int().min(1).max(100).optional().default(25),
    }),
    response: { 200: SkillListResponseSchema },
  },
  handler: async (request, reply) => {
    const { q, category, skillType, source, isActive, page, pageSize } = request.query;
    let query = db.select().from(skillTable).where(eq(skillTable.tenantId, request.tenantId));

    if (q) {
      // pg_trgm similarity search with ranking
      query = query.where(sql`similarity(${skillTable.name}, ${q}) > 0.2`)
                   .orderBy(sql`similarity(${skillTable.name}, ${q}) DESC`);
    }
    if (category) query = query.where(eq(skillTable.category, category));
    if (skillType) query = query.where(eq(skillTable.skillType, skillType));
    if (source) query = query.where(eq(skillTable.source, source));
    query = query.where(eq(skillTable.isActive, isActive));

    const offset = (page - 1) * pageSize;
    const results = await query.limit(pageSize).offset(offset);
    return { data: results, page, pageSize, total: results.length };
  },
});
```

**Testing:**
- `GET /api/v1/skills?q=python` returns Python-related skills ranked by relevance
- `GET /api/v1/skills?q=pythn` (typo) still returns Python skills via trigram similarity
- `GET /api/v1/skills?category=Cloud%20%26%20Infrastructure` returns only skills in that category
- `GET /api/v1/skills?source=onet` returns only O\*NET-sourced entries
- Pagination: `page=1&pageSize=10` returns 10 results with correct offset
- Response time < 200ms for any query against 34,000+ skills (verified with `pg_trgm` GIN index)
- Empty search `q=xyznonexistent` returns empty array, not an error

### Task 2.4: Taxonomy Browser UI

**What:** Build the taxonomy browser page in the Next.js frontend: a category tree sidebar, skill list with search/filter, and skill detail panel showing name, type, category, description, and crosswalk identifiers (Lightcast ID, O\*NET SOC code, ESCO URI).

**Design:**
```typescript
// apps/web/src/app/taxonomy/page.tsx
export default async function TaxonomyPage({ searchParams }: { searchParams: SearchParams }) {
  const categories = await fetchCategories();
  const skills = await fetchSkills(searchParams);

  return (
    <div className="flex gap-4">
      <CategoryTree categories={categories} selected={searchParams.category} />
      <div className="flex-1">
        <SkillSearchBar defaultValue={searchParams.q} />
        <SkillFilters skillType={searchParams.skillType} source={searchParams.source} />
        <SkillList skills={skills.data} />
        <Pagination page={skills.page} pageSize={skills.pageSize} total={skills.total} />
      </div>
      {searchParams.skillId && <SkillDetailPanel skillId={searchParams.skillId} />}
    </div>
  );
}
```

**Testing:**
- Category tree displays all 31 Lightcast categories in alphabetical order
- Clicking a category filters the skill list to only that category's skills
- Search input triggers debounced API call after 300ms of inactivity
- Skill detail panel shows all crosswalk identifiers when available
- Loading state shows skeleton UI while data fetches
- Empty state shows "No skills found" message with suggestion to adjust filters
- Keyboard navigation: Tab cycles through categories, Enter selects

### Task 2.5: Taxonomy Admin — Custom Skills

**What:** Allow tenant admins to create custom skills that coexist with imported taxonomy skills. Custom skills have `source = 'custom'` and can be tagged, categorised, and linked to existing taxonomy entries.

**Design:**
```typescript
// apps/api/src/routes/skills.ts — POST endpoint
app.post('/api/v1/skills', {
  schema: {
    body: z.object({
      name: z.string().min(2).max(200),
      skillType: z.enum(['specialised', 'common', 'software', 'certification']),
      category: z.string().optional(),
      subcategory: z.string().optional(),
      description: z.string().optional(),
      attributes: z.record(z.unknown()).optional(),
    }),
    response: { 201: SkillResponseSchema },
  },
  handler: async (request, reply) => {
    const existing = await db.select().from(skillTable)
      .where(and(eq(skillTable.tenantId, request.tenantId), ilike(skillTable.name, request.body.name)));
    if (existing.length) return reply.status(409).send({ error: 'Skill already exists', existing: existing[0] });

    const [created] = await db.insert(skillTable).values({
      tenantId: request.tenantId,
      ...request.body,
      source: 'custom',
    }).returning();

    await db.insert(auditEventTable).values({
      tenantId: request.tenantId,
      entityType: 'skill',
      entityId: created.id,
      eventType: 'skill_created',
      payload: request.body,
      userId: request.userId,
    });

    return reply.status(201).send(created);
  },
});
```

**Testing:**
- Admin can create a custom skill with name, type, and category
- Duplicate skill names within the same tenant are rejected with `409`
- Custom skills appear in search results alongside imported skills
- Custom skills have `source = 'custom'` and no external taxonomy IDs
- Non-admin users receive `403` when attempting to create skills
- Audit event is recorded for the creation
- Custom skills can be updated and soft-deleted (is_active = false)

---

## Phase 3: Job Architecture Builder

**Goal:** Enable HR admins to define job families, grade/level hierarchies, and job profiles with skill requirements — the structural core of any job architecture project.

**Definition of Done:** An admin can create a complete job architecture: job families, levels with pay bands, and job profiles with mapped skills at specified proficiency levels. The architecture is exportable to CSV and JSON.

### Task 3.1: Job Family & Level Management API

**What:** CRUD endpoints for job families (with optional parent for sub-families) and job levels (with track: IC/management/executive, level number, and pay band attributes in JSONB).

**Design:**
```typescript
// apps/api/src/routes/job-families.ts
app.post('/api/v1/job-families', {
  schema: {
    body: z.object({
      name: z.string().min(2).max(200),
      code: z.string().optional(),
      parentId: z.string().uuid().optional(),
      description: z.string().optional(),
      attributes: z.record(z.unknown()).optional(),
    }),
  },
  handler: async (request, reply) => {
    // Validate parent exists if provided
    if (request.body.parentId) {
      const parent = await db.select().from(jobFamilyTable)
        .where(and(eq(jobFamilyTable.id, request.body.parentId), eq(jobFamilyTable.tenantId, request.tenantId)));
      if (!parent.length) return reply.status(404).send({ error: 'Parent job family not found' });
    }
    const [family] = await db.insert(jobFamilyTable).values({
      tenantId: request.tenantId,
      ...request.body,
    }).returning();
    return reply.status(201).send(family);
  },
});
```

**Testing:**
- Create a job family with name "Engineering" and code "ENG"
- Create a sub-family "Backend Engineering" with parentId pointing to "Engineering"
- Create levels: IC1 through IC7 (individual_contributor) and M1 through M4 (management)
- Each level has pay band data in JSONB: `{ "pay_band": { "currency": "USD", "min": 80000, "mid": 100000, "max": 120000 } }`
- Deleting a parent family with children returns `409` (must delete children first)
- Duplicate family names within the same tenant return `409`
- Level numbers must be unique within a track for the same tenant

### Task 3.2: Job Profile Builder API

**What:** CRUD endpoints for job profiles, including skill requirements stored as a JSONB array on the profile. Each skill requirement specifies the skill ID, required proficiency level, and importance (required/preferred/nice_to_have).

**Design:**
```typescript
// apps/api/src/routes/job-profiles.ts
app.post('/api/v1/job-profiles', {
  schema: {
    body: z.object({
      title: z.string().min(2).max(300),
      code: z.string().optional(),
      jobFamilyId: z.string().uuid(),
      jobLevelId: z.string().uuid(),
      summary: z.string().optional(),
      description: z.string().optional(),
      onetSocCode: z.string().optional(),
      requiredSkills: z.array(z.object({
        skillId: z.string().uuid(),
        requiredLevel: z.number().int().min(1).max(7),
        importance: z.enum(['required', 'preferred', 'nice_to_have']),
      })).optional().default([]),
      attributes: z.record(z.unknown()).optional(),
    }),
  },
  handler: async (request, reply) => {
    // Validate job family and level exist
    // Validate all skill IDs exist
    // Enrich requiredSkills with skill names for denormalized storage
    const enrichedSkills = await enrichSkillReferences(request.body.requiredSkills, request.tenantId);
    const [profile] = await db.insert(jobProfileTable).values({
      tenantId: request.tenantId,
      ...request.body,
      requiredSkills: enrichedSkills,
      status: 'draft',
    }).returning();
    return reply.status(201).send(profile);
  },
});
```

**Testing:**
- Create a job profile "Senior Backend Engineer" in family "Engineering" at level IC5
- Add 8 required skills with varying proficiency levels and importance
- Skill references include denormalized names (for display without join)
- Invalid skill IDs in `requiredSkills` return `400` with details of which IDs are invalid
- Profile status starts as `draft`; can be moved to `active` or `archived`
- Updating a profile records an audit event with before/after diff
- Job profiles are filterable by family, level, status

### Task 3.3: Job Architecture UI — Family/Level/Profile Management

**What:** Build the Job Architecture section of the UI: a tree view of job families and sub-families, a level/grade table editor, and a profile creation form with a skill picker that searches the taxonomy.

**Design:**
```typescript
// apps/web/src/components/job-profile-form.tsx
export function JobProfileForm({ families, levels, onSubmit }: JobProfileFormProps) {
  const [selectedSkills, setSelectedSkills] = useState<SkillRequirement[]>([]);

  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <Input label="Job Title" {...register('title')} />
      <Select label="Job Family" options={families} {...register('jobFamilyId')} />
      <Select label="Level" options={levels} {...register('jobLevelId')} />
      <Textarea label="Summary" {...register('summary')} />
      <Textarea label="Description" {...register('description')} />

      <h3>Required Skills</h3>
      <SkillPicker
        selected={selectedSkills}
        onAdd={(skill, level, importance) =>
          setSelectedSkills([...selectedSkills, { skillId: skill.id, name: skill.name, requiredLevel: level, importance }])
        }
        onRemove={(skillId) =>
          setSelectedSkills(selectedSkills.filter(s => s.skillId !== skillId))
        }
      />
      <SkillRequirementsTable skills={selectedSkills} />
      <Button type="submit">Save Profile</Button>
    </form>
  );
}
```

**Testing:**
- Family tree view renders hierarchically with expand/collapse
- Level table shows all levels sorted by track then level_number
- Pay band editor allows multi-currency input
- Profile form validates required fields before submission
- Skill picker opens a modal with taxonomy search; selecting a skill adds it with a proficiency level slider
- Removing a skill from the profile works via the requirements table's delete action
- Form shows validation errors inline (empty title, no family selected, etc.)

### Task 3.4: Export — CSV, JSON, HR-XML

**What:** Build export functionality for job architecture data: job families, levels, profiles, and skill requirements. Support CSV, JSON, and HR-XML formats. Exports run as background jobs and produce downloadable files stored in S3/MinIO.

**Design:**
```typescript
// apps/worker/src/jobs/export.ts
export async function processExportJob(job: Job<ExportJobData>) {
  const { tenantId, format, scope } = job.data;
  const profiles = await fetchAllProfiles(tenantId);

  let content: string;
  switch (format) {
    case 'csv':
      content = convertToCSV(profiles);
      break;
    case 'json':
      content = JSON.stringify(profiles, null, 2);
      break;
    case 'hr_xml':
      content = convertToHRXML(profiles); // HR Open Standards competency model format
      break;
    case 'schema_org':
      content = convertToSchemaOrgJsonLd(profiles); // Schema.org JobPosting JSON-LD
      break;
  }

  const fileUrl = await uploadToS3(`exports/${tenantId}/${job.id}.${format}`, content);
  await db.update(dataJobTable).set({ status: 'completed', result: { fileUrl, recordCount: profiles.length } })
    .where(eq(dataJobTable.id, job.data.jobId));
}
```

**Testing:**
- CSV export contains headers matching the job profile schema columns
- JSON export is valid JSON parseable by `JSON.parse()`
- HR-XML export validates against the HR Open Standards competency model XSD
- Schema.org export produces valid JSON-LD with `@type: "JobPosting"` and `occupationalCategory` from O\*NET SOC code
- Export of 500 profiles completes within 30 seconds
- Download URL is valid and accessible for 24 hours
- Export with zero profiles produces a valid empty file with headers (CSV) or empty array (JSON)

---

## Phase 4: Employee Profiles & Gap Analysis

**Goal:** Import employee data, capture skill assessments, and compute skills gap analysis comparing employee profiles against their assigned job profile requirements.

**Definition of Done:** An admin can import employees via CSV, employees can self-assess their skills, managers can assess reports, and a gap analysis heat map shows individual and team-level skill gaps.

### Task 4.1: Employee Import (CSV & API)

**What:** CSV upload endpoint and parser that creates employee records, mapping CSV columns to the `employee` table schema. Also a REST API for individual employee CRUD.

**Design:**
```typescript
// apps/api/src/routes/employees.ts
app.post('/api/v1/employees/import', {
  schema: {
    body: z.object({
      fileUrl: z.string().url(), // S3 URL of uploaded CSV
      columnMapping: z.object({
        externalId: z.string(),
        fullName: z.string(),
        email: z.string().optional(),
        department: z.string().optional(),
        jobProfileCode: z.string().optional(),
        location: z.string().optional(),
        hireDate: z.string().optional(),
      }),
    }),
  },
  handler: async (request, reply) => {
    const job = await exportQueue.add('employee-import', {
      tenantId: request.tenantId,
      ...request.body,
    });
    return reply.status(202).send({ jobId: job.id, status: 'processing' });
  },
});
```

**Testing:**
- CSV with 1,000 rows imports all employees with correct field mapping
- Duplicate `external_id` values within the same tenant are handled as upserts (update existing)
- Missing required fields (full_name) cause per-row errors without failing the entire import
- Import result includes `{ imported: 950, updated: 40, errors: 10, error_details: [...] }`
- `job_profile_code` column is matched against existing profiles; unmatched codes are logged as warnings
- Individual employee CRUD endpoints work: create, read, update, soft-delete (status = 'terminated')

### Task 4.2: Skill Assessment API

**What:** Endpoints for recording skill assessments on employee records. Supports self-assessment, manager assessment, peer assessment, and formal assessment types. Each assessment updates the employee's `skill_profile` JSONB array.

**Design:**
```typescript
// apps/api/src/routes/assessments.ts
app.post('/api/v1/employees/:employeeId/assessments', {
  schema: {
    body: z.object({
      skillId: z.string().uuid(),
      proficiencyLevel: z.number().int().min(1).max(7),
      assessmentType: z.enum(['self', 'manager', 'peer', 'assessment', 'inferred']),
      evidenceUrl: z.string().url().optional(),
      notes: z.string().optional(),
    }),
  },
  handler: async (request, reply) => {
    const employee = await getEmployee(request.params.employeeId, request.tenantId);
    const skillProfile = employee.skillProfile as SkillAssessment[];
    const existingIdx = skillProfile.findIndex(s => s.skillId === request.body.skillId);

    const assessment = {
      skillId: request.body.skillId,
      name: await getSkillName(request.body.skillId),
      proficiencyLevel: request.body.proficiencyLevel,
      assessmentType: request.body.assessmentType,
      assessedAt: new Date().toISOString(),
      evidenceUrl: request.body.evidenceUrl,
    };

    if (existingIdx >= 0) {
      // Push current to history, update current
      const existing = skillProfile[existingIdx];
      const history = existing.history || [];
      history.push({ level: existing.proficiencyLevel, assessedAt: existing.assessedAt, type: existing.assessmentType });
      skillProfile[existingIdx] = { ...assessment, history };
    } else {
      skillProfile.push({ ...assessment, history: [] });
    }

    await db.update(employeeTable).set({ skillProfile }).where(eq(employeeTable.id, employee.id));
    // Record audit event
    await recordAuditEvent(request.tenantId, 'employee', employee.id, 'skill_assessed', request.body, request.userId);
    return reply.status(200).send({ skillProfile });
  },
});
```

**Testing:**
- Self-assessment: employee can assess their own skills
- Manager assessment: manager can assess their direct reports' skills
- Peer assessment: users with peer-assess permission can assess colleagues
- Assessment history: re-assessing a skill moves the previous assessment to the `history` array
- Proficiency level outside 1-7 range returns `400`
- Assessing a non-existent skill returns `404`
- Authorization: employees can only self-assess; managers can assess their reports; admins can assess anyone

### Task 4.3: Gap Analysis Engine

**What:** Service that computes skill gaps for an employee by comparing their `skill_profile` against their assigned `job_profile.required_skills`. Returns gaps sorted by severity, grouped by importance level.

**Design:**
```typescript
// apps/api/src/services/gap-analysis.ts
export async function computeGapAnalysis(employeeId: string, tenantId: string): Promise<GapAnalysisResult> {
  const employee = await getEmployee(employeeId, tenantId);
  if (!employee.jobProfileId) throw new Error('Employee not assigned to a job profile');

  const profile = await getJobProfile(employee.jobProfileId);
  const requiredSkills = profile.requiredSkills as SkillRequirement[];
  const employeeSkills = employee.skillProfile as SkillAssessment[];

  const gaps: SkillGap[] = requiredSkills.map(req => {
    const actual = employeeSkills.find(s => s.skillId === req.skillId);
    return {
      skillId: req.skillId,
      skillName: req.name,
      importance: req.importance,
      requiredLevel: req.requiredLevel,
      actualLevel: actual?.proficiencyLevel ?? 0,
      gap: req.requiredLevel - (actual?.proficiencyLevel ?? 0),
      assessmentType: actual?.assessmentType ?? 'not_assessed',
    };
  });

  return {
    employeeId,
    employeeName: employee.fullName,
    jobProfileId: profile.id,
    jobProfileTitle: profile.title,
    gaps: gaps.sort((a, b) => b.gap - a.gap),
    summary: {
      totalRequired: gaps.length,
      met: gaps.filter(g => g.gap <= 0).length,
      gapped: gaps.filter(g => g.gap > 0).length,
      notAssessed: gaps.filter(g => g.assessmentType === 'not_assessed').length,
      avgGap: gaps.reduce((sum, g) => sum + Math.max(0, g.gap), 0) / gaps.length,
    },
  };
}
```

**Testing:**
- Employee with all required skills at or above target level: all gaps <= 0
- Employee with no assessed skills: all gaps equal to required levels, all marked `not_assessed`
- Mixed case: some skills met, some gapped, some not assessed
- Gap is never negative in the summary's `avgGap` (clamped to 0)
- Employee not assigned to a profile returns a clear error message
- Performance: gap analysis for an employee with 50 required skills completes in < 50ms

### Task 4.4: Gap Analysis UI — Individual & Team Heat Maps

**What:** Individual employee gap analysis page showing a skills radar chart and gap table. Team-level heat map showing aggregate skill coverage across a department or the entire organisation.

**Design:**
```typescript
// apps/web/src/app/gap-analysis/[employeeId]/page.tsx
export default async function EmployeeGapPage({ params }: { params: { employeeId: string } }) {
  const analysis = await fetchGapAnalysis(params.employeeId);
  return (
    <div className="space-y-6">
      <EmployeeHeader employee={analysis.employeeName} profile={analysis.jobProfileTitle} />
      <GapSummaryCards summary={analysis.summary} />
      <div className="grid grid-cols-2 gap-4">
        <SkillsRadarChart gaps={analysis.gaps} />
        <GapTable gaps={analysis.gaps} />
      </div>
    </div>
  );
}

// apps/web/src/app/gap-analysis/team/page.tsx
export default async function TeamGapPage({ searchParams }: { searchParams: { department?: string } }) {
  const heatMap = await fetchOrgSkillCoverage(searchParams.department);
  return (
    <div>
      <DepartmentFilter selected={searchParams.department} />
      <SkillHeatMap data={heatMap} />
    </div>
  );
}
```

**Testing:**
- Individual gap page shows radar chart with skill names on axes and required vs. actual levels
- Gap table is sorted by gap severity (largest gaps first)
- Color coding: red (gap >= 3), amber (gap 1-2), green (gap <= 0), grey (not assessed)
- Team heat map aggregates across all employees in the selected department
- Heat map cells show `employees_with_skill / total_employees` ratio
- Clicking a cell in the heat map drills down to the list of employees with/without that skill
- Empty state: department with no employees shows "No data available"

---

## Phase 5: Skills Relationships & Intelligence

**Goal:** Add relationships between skills (related, adjacent, prerequisite) and enable similarity-based skill queries to power intelligent features like "employees with similar skills" and "adjacent skills you could learn."

**Definition of Done:** Skills have typed relationships. The API supports "find related skills" and "find employees with similar skill profiles" queries. A skill detail page shows the skill's relationship graph.

### Task 5.1: Skill Relationship Table

**What:** Add a `skill_relationship` table to model typed, weighted edges between skills. Populate initial relationships from Lightcast related-skills data and O\*NET crosswalks.

**Design:**
```typescript
// packages/db/schema/skill-relationship.ts
export const skillRelationship = pgTable('skill_relationship', {
  id: uuid('id').primaryKey().defaultRandom(),
  tenantId: uuid('tenant_id').notNull().references(() => tenant.id),
  sourceSkillId: uuid('source_skill_id').notNull().references(() => skill.id, { onDelete: 'cascade' }),
  targetSkillId: uuid('target_skill_id').notNull().references(() => skill.id, { onDelete: 'cascade' }),
  relationshipType: text('relationship_type').notNull(),
    // 'related_to', 'prerequisite', 'adjacent_to', 'maps_to', 'similar_to'
  weight: real('weight').notNull().default(1.0),
  source: text('source').notNull().default('lightcast'), // 'lightcast', 'onet', 'esco', 'ai_inferred', 'manual'
  confidence: real('confidence'), // 0.0-1.0 for AI-inferred relationships
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
}, (table) => ({
  uniqueRel: unique().on(table.tenantId, table.sourceSkillId, table.targetSkillId, table.relationshipType),
}));
```

**Testing:**
- Lightcast related-skills data populates `related_to` relationships
- O\*NET crosswalk data populates `maps_to` relationships between O\*NET elements and Lightcast skills
- Relationship is directional: "Python prerequisite of Django" does not imply "Django prerequisite of Python"
- Duplicate relationship (same source, target, type) returns `409`
- Self-referencing relationships (source = target) are rejected
- Index on `(source_skill_id, relationship_type)` makes "find related skills" queries fast

### Task 5.2: Related Skills & Similarity API

**What:** API endpoints for: (a) finding skills related to a given skill up to 2 hops, (b) computing cosine similarity between two employees' skill profiles, (c) finding the top-N employees most similar to a given job profile.

**Design:**
```typescript
// apps/api/src/routes/skills.ts
app.get('/api/v1/skills/:skillId/related', {
  schema: {
    querystring: z.object({
      type: z.enum(['related_to', 'prerequisite', 'adjacent_to', 'maps_to']).optional(),
      maxHops: z.number().int().min(1).max(3).optional().default(1),
      limit: z.number().int().min(1).max(50).optional().default(20),
    }),
  },
  handler: async (request, reply) => {
    const { skillId } = request.params;
    const { type, maxHops, limit } = request.query;
    // 1-hop direct relationships
    let results = await db.select({
      skill: skillTable,
      relationship: skillRelationship,
    }).from(skillRelationship)
      .innerJoin(skillTable, eq(skillTable.id, skillRelationship.targetSkillId))
      .where(and(
        eq(skillRelationship.sourceSkillId, skillId),
        type ? eq(skillRelationship.relationshipType, type) : undefined,
      ))
      .orderBy(desc(skillRelationship.weight))
      .limit(limit);

    // If maxHops > 1, extend with 2nd hop (excluding already found)
    // ...
    return { data: results };
  },
});
```

**Testing:**
- "Python" has `related_to` edges to "Django", "Flask", "Data Science" (from Lightcast)
- 2-hop query from "Python" discovers "REST APIs" via "Flask" → "REST APIs"
- Cosine similarity between two employees with identical skill profiles returns 1.0
- Cosine similarity between two employees with no overlapping skills returns 0.0
- "Find employees matching this job profile" returns employees ranked by skill overlap score
- Performance: related-skills query < 100ms; similarity computation for 2 employees < 200ms

### Task 5.3: Skill Relationship Graph Visualisation

**What:** Add a force-directed graph visualisation to the skill detail page showing the skill's relationships. Nodes are skills, edges are relationships, and edge thickness represents weight/strength.

**Design:**
```typescript
// apps/web/src/components/skill-graph.tsx
// Using D3.js force simulation or react-force-graph
export function SkillRelationshipGraph({ skillId, relationships }: SkillGraphProps) {
  const nodes = extractNodes(relationships);
  const links = extractLinks(relationships);

  return (
    <ForceGraph2D
      graphData={{ nodes, links }}
      nodeLabel="name"
      linkDirectionalArrowLength={3}
      linkWidth={(link) => link.weight * 3}
      linkColor={(link) => relationshipTypeColor(link.type)}
      onNodeClick={(node) => router.push(`/taxonomy?skillId=${node.id}`)}
    />
  );
}
```

**Testing:**
- Graph renders with the selected skill as the central node
- Related skills appear as connected nodes with correct edge types
- Edge thickness reflects relationship weight
- Color legend shows: blue=related, green=adjacent, red=prerequisite, grey=maps_to
- Clicking a node navigates to that skill's detail page
- Graph handles skills with 50+ relationships without performance degradation
- Hover tooltip shows skill name and relationship type

---

## Phase 6: HRIS Integration Layer

**Goal:** Connect to external HRIS systems to sync employee data, job codes, and organisational structure. Start with BambooHR (mid-market target) and Merge unified API (broad coverage).

**Definition of Done:** An admin can connect their BambooHR (or Merge-supported HRIS) instance, configure field mapping, and run a sync that imports employees and maps them to existing job profiles.

### Task 6.1: HRIS Connection Management

**What:** API and UI for managing HRIS connections: create a connection, store encrypted credentials, configure field mapping, test the connection, and view sync history.

**Design:**
```typescript
// apps/api/src/routes/integrations.ts
app.post('/api/v1/integrations/hris', {
  schema: {
    body: z.object({
      provider: z.enum(['bamboohr', 'merge', 'csv']),
      config: z.object({
        apiKey: z.string().optional(),        // BambooHR
        subdomain: z.string().optional(),      // BambooHR
        mergeAccountToken: z.string().optional(), // Merge
      }),
      fieldMapping: z.object({
        externalId: z.string().default('id'),
        fullName: z.string().default('displayName'),
        email: z.string().default('workEmail'),
        department: z.string().default('department'),
        jobTitle: z.string().default('jobTitle'),
        location: z.string().default('location'),
        hireDate: z.string().default('hireDate'),
      }).optional(),
    }),
  },
  handler: async (request, reply) => {
    // Encrypt sensitive config values before storage
    const encryptedConfig = encryptSensitiveFields(request.body.config);
    const [connection] = await db.insert(hrisConnectionTable).values({
      tenantId: request.tenantId,
      provider: request.body.provider,
      config: encryptedConfig,
      fieldMapping: request.body.fieldMapping ?? getDefaultFieldMapping(request.body.provider),
      status: 'pending',
    }).returning();
    return reply.status(201).send({ id: connection.id, provider: connection.provider, status: 'pending' });
  },
});
```

**Testing:**
- Create a BambooHR connection with subdomain and API key
- API key is stored encrypted (not visible in plaintext in the database)
- "Test connection" endpoint validates credentials by making a lightweight API call
- Invalid credentials return a clear error message ("Authentication failed: invalid API key")
- Connection status transitions: pending → active (on successful test) → error (on failed sync)
- Multiple HRIS connections per tenant are supported (e.g., BambooHR for US, Personio for EU)

### Task 6.2: BambooHR Sync Worker

**What:** BullMQ worker that syncs employees from BambooHR's REST API, maps fields according to the connection's `field_mapping`, and upserts employee records. Runs on-demand or on a configurable schedule.

**Design:**
```typescript
// apps/worker/src/jobs/hris-sync.ts
export async function syncBambooHR(job: Job<HrisSyncJobData>) {
  const connection = await getHrisConnection(job.data.connectionId);
  const bamboo = new BambooHRClient(connection.config.subdomain, decrypt(connection.config.apiKey));
  const employees = await bamboo.getEmployeeDirectory();
  const mapping = connection.fieldMapping;

  let synced = 0, errors = 0;
  for (const emp of employees) {
    try {
      const mapped = {
        externalId: emp[mapping.externalId],
        fullName: emp[mapping.fullName],
        email: emp[mapping.email],
        department: emp[mapping.department],
        location: emp[mapping.location],
        hireDate: emp[mapping.hireDate] ? new Date(emp[mapping.hireDate]) : null,
      };
      // Match job profile by title or HRIS job code
      const jobProfile = await matchJobProfile(emp[mapping.jobTitle], connection.tenantId);

      await db.insert(employeeTable).values({
        tenantId: connection.tenantId,
        ...mapped,
        jobProfileId: jobProfile?.id,
        status: emp.status === 'Active' ? 'active' : 'inactive',
      }).onConflictDoUpdate({
        target: [employeeTable.tenantId, employeeTable.externalId],
        set: { ...mapped, jobProfileId: jobProfile?.id, updatedAt: new Date() },
      });
      synced++;
    } catch (e) {
      errors++;
    }
  }

  await updateConnectionSyncStatus(connection.id, synced, errors);
}
```

**Testing:**
- Sync imports all active employees from BambooHR
- Field mapping correctly translates BambooHR field names to local schema
- Job title matching: "Senior Software Engineer" maps to the correct job profile if one exists
- Unmatched job titles leave `job_profile_id` as null (not an error)
- Re-sync updates changed fields and does not create duplicates
- Terminated employees in BambooHR are set to `status = 'inactive'`
- Sync log records: connection_id, start time, end time, records synced, errors

### Task 6.3: Merge Unified API Integration

**What:** Alternative sync path using Merge's unified HRIS API, covering 220+ HRIS platforms through a single normalised data model. This gives broad HRIS coverage without building individual connectors.

**Design:**
```typescript
// apps/worker/src/jobs/hris-sync-merge.ts
export async function syncMerge(job: Job<HrisSyncJobData>) {
  const connection = await getHrisConnection(job.data.connectionId);
  const merge = new MergeClient(process.env.MERGE_API_KEY, decrypt(connection.config.mergeAccountToken));

  // Merge normalises all HRIS data to their standard schema
  const employees = await merge.hris.employees.list({ includeRemoteData: false });

  for (const emp of employees.results) {
    await db.insert(employeeTable).values({
      tenantId: connection.tenantId,
      externalId: emp.id,
      fullName: emp.displayFullName,
      email: emp.workEmail,
      department: emp.team?.name,
      location: emp.workLocation?.name,
      hireDate: emp.startDate ? new Date(emp.startDate) : null,
      status: emp.employmentStatus === 'ACTIVE' ? 'active' : 'inactive',
    }).onConflictDoUpdate({
      target: [employeeTable.tenantId, employeeTable.externalId],
      set: { fullName: emp.displayFullName, email: emp.workEmail, updatedAt: new Date() },
    });
  }
}
```

**Testing:**
- Merge integration syncs employees from a test Workday sandbox
- Merge integration syncs employees from a test BambooHR sandbox
- Normalised field names (displayFullName, workEmail) map correctly
- Pagination: sync handles > 1,000 employees via Merge's cursor-based pagination
- Rate limiting: sync respects Merge's rate limits with exponential backoff
- Webhook: Merge webhook endpoint receives sync completion notifications

---

## Phase 7: AI-Assisted Role Profiling

**Goal:** Given a job title and description, use AI to suggest skills from the taxonomy with proficiency levels. HR practitioners review and accept/modify suggestions.

**Definition of Done:** An admin enters a job title and optional description; the system returns 10-20 suggested skills with proficiency levels; the admin can accept, modify, or reject each suggestion; accepted suggestions populate the job profile's `required_skills`.

### Task 7.1: AI Role Profiling Service

**What:** Service that takes a job title and description, queries the skills taxonomy for context, and calls the Claude API to infer required skills with proficiency levels and importance ratings.

**Design:**
```typescript
// apps/api/src/services/ai-role-profiler.ts
export async function profileRole(
  title: string,
  description: string | undefined,
  tenantId: string,
): Promise<RoleSkillSuggestion[]> {
  // Fetch relevant skills from taxonomy for context
  const candidateSkills = await searchSkills(tenantId, title, { limit: 100 });
  const taxonomyContext = candidateSkills.map(s => `${s.name} (${s.category}, ${s.skillType})`).join('\n');

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 4096,
    system: `You are an expert HR consultant specialising in job architecture and skills frameworks.
Given a job title and description, suggest 10-20 skills from the provided taxonomy that this role requires.
For each skill, provide:
- skill_name: exact name from the taxonomy
- proficiency_level: 1-5 (1=Awareness, 2=Beginner, 3=Intermediate, 4=Advanced, 5=Expert)
- importance: "required", "preferred", or "nice_to_have"
- rationale: one sentence explaining why this skill is needed

Return valid JSON array only.`,
    messages: [{
      role: 'user',
      content: `Job Title: ${title}
${description ? `Description: ${description}` : ''}

Available skills from our taxonomy:
${taxonomyContext}`,
    }],
  });

  const suggestions = JSON.parse(response.content[0].text) as RawSuggestion[];

  // Match suggested skill names back to taxonomy IDs
  return await matchSuggestionsToTaxonomy(suggestions, tenantId);
}
```

**Testing:**
- "Senior Backend Engineer" returns skills like Python, PostgreSQL, System Design, CI/CD, Docker
- Proficiency levels are reasonable: "Python" at 4 (Advanced), "Machine Learning" at 2 (Beginner) for a backend role
- All suggested skill names match actual skills in the taxonomy (fuzzy matching for minor variations)
- Suggestions include a mix of required, preferred, and nice_to_have importance levels
- Rationale field is present and non-empty for every suggestion
- AI response parsing handles malformed JSON gracefully (retry with stricter prompt)
- Rate limiting: maximum 10 AI profiling requests per tenant per minute

### Task 7.2: Human-in-the-Loop Review UI

**What:** UI for reviewing AI skill suggestions for a job profile. Each suggestion is shown as a card with accept/modify/reject actions. Accepted suggestions are added to the profile's `required_skills`. An audit trail records which suggestions were AI-generated.

**Design:**
```typescript
// apps/web/src/components/ai-skill-review.tsx
export function AISkillReview({ suggestions, onAccept, onReject, onModify }: AISkillReviewProps) {
  return (
    <div className="space-y-3">
      <div className="flex items-center gap-2 text-sm text-muted-foreground">
        <SparklesIcon className="h-4 w-4" />
        <span>AI suggested {suggestions.length} skills. Review each suggestion below.</span>
      </div>
      {suggestions.map((suggestion) => (
        <Card key={suggestion.skillId}>
          <CardContent className="flex items-center justify-between p-4">
            <div>
              <p className="font-medium">{suggestion.skillName}</p>
              <p className="text-sm text-muted-foreground">{suggestion.rationale}</p>
              <div className="flex gap-2 mt-1">
                <Badge>{suggestion.importance}</Badge>
                <Badge variant="outline">Level {suggestion.proficiencyLevel}/5</Badge>
                <Badge variant="secondary">{suggestion.confidence}% confidence</Badge>
              </div>
            </div>
            <div className="flex gap-2">
              <Button size="sm" variant="outline" onClick={() => onModify(suggestion)}>Modify</Button>
              <Button size="sm" variant="destructive" onClick={() => onReject(suggestion)}>Reject</Button>
              <Button size="sm" onClick={() => onAccept(suggestion)}>Accept</Button>
            </div>
          </CardContent>
        </Card>
      ))}
    </div>
  );
}
```

**Testing:**
- Clicking "Accept" adds the skill to the job profile's required_skills with `source: 'ai_suggested'` in attributes
- Clicking "Modify" opens a dialog to adjust proficiency level and importance before accepting
- Clicking "Reject" removes the suggestion from the list and records the rejection in audit
- Bulk accept: a "Accept All" button adds all remaining suggestions at once
- Audit trail shows: skill name, AI confidence, accepted/rejected, modified by user, timestamp
- After review, the job profile's required_skills contains both manually-added and AI-suggested skills
- AI badge/icon on skills that were AI-suggested (visible in the profile's skill list)

### Task 7.3: Batch Role Profiling

**What:** Allow admins to select multiple job profiles and run AI profiling on all of them as a batch job. Results are queued for review.

**Design:**
```typescript
// apps/api/src/routes/ai.ts
app.post('/api/v1/ai/batch-profile', {
  schema: {
    body: z.object({
      jobProfileIds: z.array(z.string().uuid()).min(1).max(50),
    }),
  },
  handler: async (request, reply) => {
    const jobs = request.body.jobProfileIds.map(id => ({
      name: 'ai-role-profile',
      data: { tenantId: request.tenantId, jobProfileId: id, userId: request.userId },
    }));
    await aiQueue.addBulk(jobs);
    return reply.status(202).send({
      message: `${jobs.length} profiles queued for AI analysis`,
      jobCount: jobs.length,
    });
  },
});
```

**Testing:**
- Batch of 10 profiles queues 10 background jobs
- Each job produces suggestions stored in a `pending_suggestions` table for review
- Admin dashboard shows "X profiles have pending AI suggestions"
- Rate limiting prevents more than 50 profiles per batch
- Failed AI calls (rate limit, timeout) are retried 3 times with exponential backoff
- Partial failure: 8/10 succeed, 2 fail — admin sees which profiles failed

---

## Phase 8: Career Pathways & Visualisation

**Goal:** Define and visualise career progression paths through the job architecture, showing employees what roles they could move to and what skills they need to develop.

**Definition of Done:** Career pathways are defined between job profiles. An employee can view their possible career paths, see the skill gaps for each target role, and visualise the progression as an interactive diagram.

### Task 8.1: Career Pathway Management

**What:** CRUD API for career pathways between job profiles. Each pathway has a type (promotion, lateral, specialisation), typical duration, and auto-computed skill gaps between source and target roles.

**Design:**
```typescript
// apps/api/src/services/career-pathway.ts
export async function createCareerPathway(
  fromProfileId: string,
  toProfileId: string,
  pathwayType: 'promotion' | 'lateral' | 'specialisation',
  typicalDurationMonths: number | null,
  tenantId: string,
): Promise<CareerPathway> {
  const fromProfile = await getJobProfile(fromProfileId);
  const toProfile = await getJobProfile(toProfileId);

  // Auto-compute skill gaps: skills required by target that aren't required by source (or at lower level)
  const fromSkills = fromProfile.requiredSkills as SkillRequirement[];
  const toSkills = toProfile.requiredSkills as SkillRequirement[];

  const skillGaps = toSkills
    .map(targetSkill => {
      const sourceSkill = fromSkills.find(s => s.skillId === targetSkill.skillId);
      const gapLevels = targetSkill.requiredLevel - (sourceSkill?.requiredLevel ?? 0);
      return gapLevels > 0 ? { skillId: targetSkill.skillId, name: targetSkill.name, gapLevels } : null;
    })
    .filter(Boolean);

  const [pathway] = await db.insert(careerPathwayTable).values({
    tenantId,
    fromProfileId,
    toProfileId,
    pathwayType,
    typicalDurationMonths,
    skillGaps,
  }).returning();

  return pathway;
}
```

**Testing:**
- Promotion pathway from "Software Engineer IC3" to "Senior Software Engineer IC4"
- Lateral pathway from "Backend Engineer" to "Platform Engineer" (same level)
- Skill gaps auto-computed: target requires "Kubernetes L4" but source only requires "Kubernetes L2" → gap of 2
- Skills unique to the target role appear with gap equal to their required level
- Circular pathways (A→B→A) are permitted (lateral moves)
- Duplicate pathways (same from/to) return `409`
- Updating either profile's required_skills triggers re-computation of pathway skill gaps

### Task 8.2: Career Pathway Visualisation

**What:** Interactive career pathway diagram showing the job architecture as a directed graph. Nodes are job profiles (grouped by job family), edges are pathways. Clicking a pathway shows the skill gap details.

**Design:**
```typescript
// apps/web/src/components/career-pathway-diagram.tsx
export function CareerPathwayDiagram({ profiles, pathways, currentProfileId }: CareerDiagramProps) {
  // Group profiles by job family for visual clustering
  const clusters = groupByFamily(profiles);
  const nodes = profiles.map(p => ({
    id: p.id,
    label: p.title,
    level: p.levelNumber,
    family: p.jobFamilyName,
    isCurrent: p.id === currentProfileId,
  }));
  const edges = pathways.map(pw => ({
    from: pw.fromProfileId,
    to: pw.toProfileId,
    type: pw.pathwayType,
    label: pw.typicalDurationMonths ? `~${pw.typicalDurationMonths}mo` : undefined,
    gapCount: pw.skillGaps.length,
  }));

  return (
    <div>
      <DirectedGraph nodes={nodes} edges={edges} clusters={clusters}
        onEdgeClick={(edge) => setSelectedPathway(edge)} />
      {selectedPathway && (
        <PathwayGapPanel pathway={selectedPathway} />
      )}
    </div>
  );
}
```

**Testing:**
- Diagram renders all profiles as nodes with correct level positioning (higher levels at top)
- Promotion pathways are drawn as upward arrows; lateral as horizontal; specialisation as diagonal
- Current employee's profile is highlighted
- Edge labels show estimated duration
- Clicking an edge opens a panel showing the skill gaps for that pathway
- Zoom and pan work on the diagram
- Diagram handles 50+ profiles and 100+ pathways without performance issues
- Mobile view shows a simplified list view instead of the graph

### Task 8.3: Employee Career Explorer

**What:** Employee-facing page where they can explore possible career paths from their current role, see which paths have the smallest skill gaps, and view personalised development recommendations.

**Design:**
```typescript
// apps/web/src/app/career/page.tsx
export default async function CareerExplorerPage() {
  const session = await getServerSession();
  const employee = await getEmployeeByUserId(session.user.id);
  const currentProfile = await getJobProfile(employee.jobProfileId);
  const reachableProfiles = await getReachableProfiles(currentProfile.id, { maxHops: 3 });
  const gapAnalysis = await computeGapAnalysis(employee.id, employee.tenantId);

  return (
    <div className="space-y-6">
      <h1>Your Career Paths</h1>
      <CurrentRoleCard profile={currentProfile} />
      <CareerPathwayDiagram
        profiles={reachableProfiles.profiles}
        pathways={reachableProfiles.pathways}
        currentProfileId={currentProfile.id}
      />
      <h2>Recommended Next Steps</h2>
      <NextStepsList
        pathways={reachableProfiles.pathways.filter(p => p.fromProfileId === currentProfile.id)}
        employeeGaps={gapAnalysis.gaps}
      />
    </div>
  );
}
```

**Testing:**
- Employee sees only pathways reachable from their current role
- Next steps are sorted by feasibility (fewest skill gaps first)
- Each next step shows: target role, pathway type, estimated duration, skill gap count
- Clicking a next step shows detailed skill gaps with "your level" vs. "required level"
- Employee without a job profile sees "Contact HR to set up your career profile"
- Employee with all gaps met for a target role sees a "Ready for promotion" indicator

---

## Phase 9: Self-Maintaining Taxonomy Agent

**Goal:** Build an AI agent that monitors external sources (Lightcast updates, job posting trends) to propose new emerging skills and flag obsolete skills for deprecation, with human approval workflow.

**Definition of Done:** The agent runs on a weekly schedule, identifies 5-20 skill change proposals per run, presents them in a review queue, and approved changes are applied to the taxonomy with full audit trail.

### Task 9.1: Taxonomy Change Detection

**What:** Background job that compares the current taxonomy against the latest Lightcast data, identifies new skills not yet in the taxonomy, skills that have been deprecated upstream, and skills with significantly changed demand signals.

**Design:**
```typescript
// apps/worker/src/jobs/taxonomy-agent.ts
export async function detectTaxonomyChanges(job: Job<TaxonomyAgentJobData>) {
  const { tenantId } = job.data;
  const currentSkills = await getAllSkills(tenantId, 'lightcast');
  const latestLightcast = await fetchAllLightcastSkills(await getLightcastOAuthToken());

  const currentIds = new Set(currentSkills.map(s => s.lightcastId));
  const latestIds = new Set(latestLightcast.map(s => s.id));

  // New skills in Lightcast not in our taxonomy
  const newSkills = latestLightcast.filter(s => !currentIds.has(s.id));

  // Skills removed from Lightcast (potentially obsolete)
  const removedSkills = currentSkills.filter(s => s.lightcastId && !latestIds.has(s.lightcastId));

  // Queue proposals for human review
  for (const skill of newSkills) {
    await db.insert(taxonomyProposalTable).values({
      tenantId,
      proposalType: 'add_skill',
      skillName: skill.name,
      category: skill.category?.name,
      source: 'ai_agent',
      evidence: { lightcastId: skill.id, category: skill.category, type: skill.type },
      status: 'pending',
    });
  }

  for (const skill of removedSkills) {
    await db.insert(taxonomyProposalTable).values({
      tenantId,
      proposalType: 'deprecate_skill',
      skillId: skill.id,
      skillName: skill.name,
      source: 'ai_agent',
      evidence: { reason: 'Removed from Lightcast taxonomy', lightcastId: skill.lightcastId },
      status: 'pending',
    });
  }
}
```

**Testing:**
- Agent detects new skills added to Lightcast since last sync
- Agent detects skills removed from Lightcast and proposes deprecation
- Proposals are created with status `pending` and include evidence
- Running the agent twice does not create duplicate proposals
- Agent job completes within 5 minutes for a full taxonomy comparison
- Agent handles Lightcast API rate limiting gracefully

### Task 9.2: AI-Enhanced Skill Proposals

**What:** Extend the taxonomy agent to use AI to enrich proposals: for new skills, infer category placement, suggest related skills, and estimate importance. For deprecations, suggest replacement skills.

**Design:**
```typescript
// apps/worker/src/jobs/taxonomy-agent-ai.ts
export async function enrichProposalWithAI(proposalId: string) {
  const proposal = await getProposal(proposalId);
  const taxonomyContext = await getRelevantTaxonomyContext(proposal.skillName);

  const response = await anthropic.messages.create({
    model: 'claude-sonnet-4-20250514',
    max_tokens: 2048,
    system: `You are an HR taxonomy expert. Enrich the following skill proposal with:
1. Recommended category and subcategory from the existing taxonomy
2. Up to 5 related skills from the taxonomy
3. Skill type classification (specialised/common/software)
4. Importance assessment (1-10) for modern workforce
5. For deprecations: suggest 1-3 replacement skills
Return valid JSON.`,
    messages: [{ role: 'user', content: `Skill: ${proposal.skillName}\n\nExisting taxonomy categories:\n${taxonomyContext}` }],
  });

  const enrichment = JSON.parse(response.content[0].text);
  await db.update(taxonomyProposalTable).set({
    aiEnrichment: enrichment,
    aiConfidence: enrichment.confidence,
  }).where(eq(taxonomyProposalTable.id, proposalId));
}
```

**Testing:**
- New skill "Prompt Engineering" is enriched with category "Artificial Intelligence", related skills ["Machine Learning", "NLP"]
- Deprecation of "Flash Development" suggests replacement skills ["Web Development", "JavaScript"]
- AI confidence score is between 0.0 and 1.0
- Enrichment handles unknown skills gracefully (returns lower confidence)
- AI enrichment does not override the human review requirement

### Task 9.3: Taxonomy Proposal Review Queue

**What:** Admin UI showing pending taxonomy proposals with AI enrichment. Admins can approve (apply to taxonomy), reject (dismiss), or modify proposals. Approved changes are applied immediately with full audit trail.

**Design:**
```typescript
// apps/web/src/app/taxonomy/proposals/page.tsx
export default async function TaxonomyProposalsPage() {
  const proposals = await fetchPendingProposals();
  return (
    <div>
      <h1>Taxonomy Change Proposals</h1>
      <p className="text-muted-foreground">{proposals.length} pending proposals</p>
      <div className="space-y-4">
        {proposals.map(p => (
          <ProposalCard key={p.id} proposal={p}
            onApprove={() => approveProposal(p.id)}
            onReject={() => rejectProposal(p.id)}
            onModify={() => openModifyDialog(p)} />
        ))}
      </div>
    </div>
  );
}
```

**Testing:**
- Proposals are sorted by AI confidence (highest first)
- "Add skill" proposals show: skill name, suggested category, related skills, AI confidence
- "Deprecate skill" proposals show: skill name, reason, replacement suggestions, affected job profiles count
- Approving an "add skill" proposal creates a new skill in the taxonomy with `source: 'ai_agent'`
- Approving a "deprecate" proposal sets `deprecated_at` and `is_active = false` on the skill
- Rejecting a proposal sets status to `rejected` with optional reason
- Affected job profiles are listed: "This skill is required by 5 active job profiles"
- Bulk approve/reject with checkboxes
- Audit trail shows: proposal, action, user, timestamp

---

## Phase 10: Deployment, DevOps & Hardening

**Goal:** Production-ready deployment with CI/CD, monitoring, security hardening, and documentation. This phase runs in parallel with Phases 3-9.

**Definition of Done:** The application can be deployed to a production environment via CI/CD, with monitoring, alerting, backup, and security controls in place. Documentation covers deployment, API reference, and admin guide.

### Task 10.1: CI/CD Pipeline

**What:** GitHub Actions workflow with: lint, type-check, unit tests, integration tests (against Dockerised PostgreSQL), build, Docker image push, and deployment to staging.

**Design:**
```yaml
# .github/workflows/ci.yml
name: CI
on: [push, pull_request]
jobs:
  lint-and-test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env: { POSTGRES_DB: jast_test, POSTGRES_USER: test, POSTGRES_PASSWORD: test }
        ports: ["5432:5432"]
      redis:
        image: redis:7-alpine
        ports: ["6379:6379"]
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: 20 }
      - run: npm ci
      - run: npx turbo lint
      - run: npx turbo test
      - run: npx turbo build
  docker:
    needs: lint-and-test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - uses: actions/checkout@v4
      - uses: docker/build-push-action@v5
        with:
          push: true
          tags: ghcr.io/${{ github.repository }}:latest
```

**Testing:**
- Push to any branch triggers lint + test + build
- Push to `main` additionally builds and pushes Docker images
- Failed lint or tests block the pipeline
- Integration tests run against a real PostgreSQL instance
- Pipeline completes in < 10 minutes

### Task 10.2: Kubernetes Helm Chart

**What:** Helm chart for deploying the application to Kubernetes with configurable replicas, resource limits, ingress, and secrets management.

**Design:**
```yaml
# helm/values.yaml
replicaCount:
  api: 2
  web: 2
  worker: 1
image:
  repository: ghcr.io/job-architecture-skills-taxonomy
  tag: latest
postgresql:
  enabled: true   # Use embedded or set to false for external
  auth:
    database: jast
redis:
  enabled: true
ingress:
  enabled: true
  className: nginx
  hosts:
    - host: jast.example.com
env:
  DATABASE_URL: ""   # Set via secret
  REDIS_URL: ""
  NEXTAUTH_SECRET: ""
  LIGHTCAST_CLIENT_ID: ""
  LIGHTCAST_CLIENT_SECRET: ""
```

**Testing:**
- `helm template` produces valid Kubernetes manifests
- `helm install` deploys all services to a local k3s cluster
- Health check endpoints respond after deployment
- Scaling `api` replicas from 2 to 4 works without downtime
- Secrets are mounted as environment variables, not in plaintext in manifests
- Database migrations run as a Kubernetes Job before the API starts

### Task 10.3: Monitoring & Alerting

**What:** Integrate Prometheus metrics, structured JSON logging, and health check endpoints. Optional Grafana dashboards for key metrics.

**Design:**
```typescript
// apps/api/src/plugins/metrics.ts
import { fastifyPlugin } from 'fastify-plugin';
import { register, collectDefaultMetrics, Counter, Histogram } from 'prom-client';

collectDefaultMetrics();

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration',
  labelNames: ['method', 'route', 'status_code'],
});

const skillSearchCount = new Counter({
  name: 'skill_search_total',
  help: 'Total skill search queries',
  labelNames: ['tenant_id'],
});

export default fastifyPlugin(async (app) => {
  app.get('/metrics', async (req, reply) => {
    reply.header('Content-Type', register.contentType);
    return register.metrics();
  });

  app.addHook('onResponse', (request, reply, done) => {
    httpRequestDuration.observe(
      { method: request.method, route: request.routeOptions.url, status_code: reply.statusCode },
      reply.elapsedTime / 1000
    );
    done();
  });
});
```

**Testing:**
- `/metrics` returns Prometheus-compatible metrics
- HTTP request duration histogram captures all API requests
- Structured logs include: timestamp, level, request_id, tenant_id, user_id, message
- Health check `/health` returns detailed status (database, Redis, S3 connectivity)
- Grafana dashboard shows: request rate, error rate, p95 latency, active tenants, skills count

### Task 10.4: Security Hardening

**What:** Implement rate limiting, input sanitisation, CORS configuration, Content Security Policy headers, and dependency vulnerability scanning.

**Design:**
```typescript
// apps/api/src/plugins/security.ts
import rateLimit from '@fastify/rate-limit';
import helmet from '@fastify/helmet';
import cors from '@fastify/cors';

export async function registerSecurity(app: FastifyInstance) {
  await app.register(helmet, {
    contentSecurityPolicy: {
      directives: {
        defaultSrc: ["'self'"],
        scriptSrc: ["'self'"],
        styleSrc: ["'self'", "'unsafe-inline'"],
      },
    },
  });

  await app.register(cors, {
    origin: process.env.ALLOWED_ORIGINS?.split(',') || ['http://localhost:3000'],
    credentials: true,
  });

  await app.register(rateLimit, {
    max: 100,
    timeWindow: '1 minute',
    keyGenerator: (request) => request.tenantId || request.ip,
  });
}
```

**Testing:**
- Rate limiting: 101st request within 1 minute returns `429 Too Many Requests`
- CORS: requests from unlisted origins are rejected
- CSP headers are present on all responses
- SQL injection attempt in search query is sanitised (parameterised queries)
- XSS payload in skill name is escaped in API response
- `npm audit` reports no high/critical vulnerabilities
- Tenant A cannot access Tenant B's data through any API endpoint (verify with integration tests)

### Task 10.5: API Documentation & Admin Guide

**What:** Auto-generated OpenAPI 3.1 documentation served at `/api/docs`. Admin guide covering: deployment, configuration, taxonomy import, HRIS integration, and AI configuration.

**Design:**
- OpenAPI spec auto-generated by Fastify's `@fastify/swagger` plugin from route schemas
- Swagger UI served at `/api/docs` for interactive API exploration
- Admin guide as a `/docs` section in the web application (MDX pages)

**Testing:**
- `/api/docs` renders interactive Swagger UI
- Every API endpoint is documented with request/response schemas
- "Try it out" feature works for GET endpoints without authentication
- Admin guide covers all Phase 1-9 features with screenshots
- Configuration reference lists all environment variables with descriptions and defaults

---

## Summary: Definition of Done — Full Project

The project is considered feature-complete when:

1. **Taxonomy:** 34,000+ Lightcast skills, 1,000+ O\*NET occupations, and ESCO concepts are imported, searchable, and browsable with crosswalk mappings.
2. **Job Architecture:** Admins can define job families, grade/level hierarchies with pay bands, and job profiles with skill requirements at specified proficiency levels.
3. **Employees & Gaps:** Employee data is importable via CSV and HRIS sync. Skill assessments produce individual and organisational gap analysis heat maps.
4. **Skills Intelligence:** Skill relationships power "related skills", "adjacent skills", and "similar employees" queries.
5. **HRIS Integration:** BambooHR and Merge (220+ HRIS) connectors sync employee data on schedule.
6. **AI Role Profiling:** AI suggests skills for job profiles with human-in-the-loop review.
7. **Career Pathways:** Career paths are defined, visualised, and personalised for each employee.
8. **Self-Maintaining Taxonomy:** AI agent proposes taxonomy changes from external signals with admin approval.
9. **Production Ready:** CI/CD, Helm chart, monitoring, security hardening, and documentation are complete.
10. **All exports work:** CSV, JSON, HR-XML, and Schema.org JSON-LD.
