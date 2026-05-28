# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: Job Architecture & Skills Taxonomy · Created: 2026-05-12

## Philosophy

This model follows classic third-normal-form relational design: every concept gets its own table, relationships are expressed through foreign keys and junction tables, and reference data aligns directly with published standards (O\*NET, ESCO, Lightcast, SFIA). The schema is designed so that the database structure itself mirrors the O\*NET Content Model's six-domain hierarchy and the Lightcast three-tier skill taxonomy, making standards crosswalks a first-class data concern rather than an afterthought.

The approach treats the skills taxonomy and job architecture as distinct but tightly coupled subsystems. Taxonomy tables store the canonical skill definitions, categories, and inter-taxonomy crosswalks. Job architecture tables store the organisational structure: job families, levels, grades, pay bands, and role-to-skill mappings. Employee tables link people to assessed skills with proficiency scores. Gap analysis is a computed view joining employee skills against role requirements.

This is the most conservative option and the easiest to reason about for teams experienced with relational databases. It produces the most tables but also the most explicit referential integrity. Every relationship is enforced by the database, and every query can be expressed in standard SQL without needing JSONB operators or graph traversal languages.

**Best for:** Organisations that need strong referential integrity, standards-compliant data exports (HR-XML/JSON, ISO 30414), and straightforward SQL-based reporting.

**Trade-offs:**
- Pro: Strongest referential integrity; every relationship is FK-enforced
- Pro: Direct mapping to O\*NET, ESCO, and Lightcast data structures; crosswalks are explicit tables
- Pro: Easiest to generate standards-compliant exports (HR-XML, Schema.org JobPosting)
- Pro: Most familiar to PostgreSQL-experienced teams; no specialised query patterns needed
- Con: Highest table count (~45-50 tables); schema migrations required for every new concept
- Con: Many-to-many junction tables create verbose queries for common operations
- Con: Taxonomy hierarchy traversal requires recursive CTEs (slower than graph databases)
- Con: Adding jurisdiction-specific or industry-specific fields requires schema changes

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| O\*NET Content Model | Six-domain structure (Worker Characteristics, Worker Requirements, Experience Requirements, Occupational Requirements, Occupation-Specific, Workforce Characteristics) directly maps to table groups; 277 descriptors stored as reference data |
| O\*NET-SOC Codes | `occupation.onet_soc_code` column links every occupation to the O\*NET taxonomy; used in Schema.org JobPosting export |
| ESCO v1.2 | Parallel taxonomy tables (`esco_occupation`, `esco_skill`) with URI-based identifiers; crosswalk table links ESCO to O\*NET and Lightcast |
| Lightcast Open Skills | Three-tier hierarchy (Category > Subcategory > Skill) stored in `lightcast_category`, `lightcast_subcategory`, `skill` tables; CC-BY 4.0 |
| SFIA 9 | Optional extension tables for IT-specific skills; 7 responsibility levels map to `sfia_level` reference table |
| HR Open Standards | Export views generate HR-JSON-compliant competency definitions and position competency models |
| IEEE 1484.20.1 | Competency definition fields (identifier, title, description, definition, metadata) align with the IEEE data model |
| ISO 30414 | Skills coverage, gap severity, and reskilling metrics can be computed from normalised tables for human capital reporting |
| ISO 3166 | `country` and `jurisdiction` reference tables use ISO 3166-1 alpha-2 codes |
| Schema.org | `occupation` and `job_profile` tables include fields that map directly to Schema.org Occupation and JobPosting vocabularies |

---

## Core Taxonomy Tables

### Skill Categories and Skills (Lightcast-aligned)

```sql
-- Top-level skill categories (31 categories in Lightcast)
CREATE TABLE skill_category (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_id     TEXT,                    -- Lightcast category ID
    name            TEXT NOT NULL,
    description     TEXT,
    source          TEXT NOT NULL DEFAULT 'lightcast',  -- 'lightcast', 'custom', 'sfia', 'onet'
    sort_order      INT DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_skill_category_tenant ON skill_category(tenant_id);

-- Subcategories within each category
CREATE TABLE skill_subcategory (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    category_id     UUID NOT NULL REFERENCES skill_category(id),
    external_id     TEXT,
    name            TEXT NOT NULL,
    description     TEXT,
    sort_order      INT DEFAULT 0,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, category_id, name)
);

CREATE INDEX idx_skill_subcategory_category ON skill_subcategory(category_id);

-- Individual skills (32,000+ from Lightcast, plus custom)
CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    subcategory_id  UUID REFERENCES skill_subcategory(id),
    external_id     TEXT,                    -- Lightcast skill ID
    name            TEXT NOT NULL,
    description     TEXT,
    skill_type      TEXT NOT NULL CHECK (skill_type IN ('specialised', 'common', 'software', 'certification')),
    source          TEXT NOT NULL DEFAULT 'lightcast',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    is_emerging     BOOLEAN NOT NULL DEFAULT FALSE,  -- flagged by AI taxonomy agent
    deprecated_at   TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_tenant ON skill(tenant_id);
CREATE INDEX idx_skill_subcategory ON skill(subcategory_id);
CREATE INDEX idx_skill_type ON skill(tenant_id, skill_type);
CREATE INDEX idx_skill_name_trgm ON skill USING gin (name gin_trgm_ops);  -- trigram for fuzzy search
```

### Proficiency Scale

```sql
-- Configurable proficiency scale per tenant
CREATE TABLE proficiency_scale (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,           -- e.g. 'Default 5-point', 'SFIA 7-level'
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE proficiency_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    scale_id        UUID NOT NULL REFERENCES proficiency_scale(id),
    level_number    INT NOT NULL,            -- 1-5, 1-7, etc.
    name            TEXT NOT NULL,           -- 'Novice', 'Beginner', 'Intermediate', 'Advanced', 'Expert'
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (scale_id, level_number)
);
```

### Standards Crosswalks

```sql
-- O*NET occupations (1,000+ occupations)
CREATE TABLE onet_occupation (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    soc_code        TEXT NOT NULL UNIQUE,    -- e.g. '15-1243.00'
    title           TEXT NOT NULL,
    description     TEXT,
    job_zone        INT,                     -- 1-4 (updated from 5 to 4 levels in O*NET 30.2)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- ESCO concepts (occupations and skills)
CREATE TABLE esco_concept (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    uri             TEXT NOT NULL UNIQUE,    -- e.g. 'http://data.europa.eu/esco/occupation/528f90ed-...'
    concept_type    TEXT NOT NULL CHECK (concept_type IN ('occupation', 'skill', 'knowledge')),
    preferred_label TEXT NOT NULL,           -- in default language
    description     TEXT,
    isco_code       TEXT,                    -- ISCO-08 code for occupations
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_esco_concept_type ON esco_concept(concept_type);

-- Cross-taxonomy mapping table
CREATE TABLE taxonomy_crosswalk (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    source_type     TEXT NOT NULL,           -- 'lightcast_skill', 'onet_occupation', 'esco_concept', 'sfia_skill'
    source_id       UUID NOT NULL,
    target_type     TEXT NOT NULL,
    target_id       UUID NOT NULL,
    match_type      TEXT NOT NULL CHECK (match_type IN ('exact', 'broad', 'narrow', 'related')),
    confidence      NUMERIC(3,2),            -- 0.00-1.00 for AI-generated crosswalks
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (source_type, source_id, target_type, target_id)
);

CREATE INDEX idx_crosswalk_source ON taxonomy_crosswalk(source_type, source_id);
CREATE INDEX idx_crosswalk_target ON taxonomy_crosswalk(target_type, target_id);
```

---

## Job Architecture Tables

### Job Families and Levels

```sql
-- Multi-tenant organisation
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2),                 -- ISO 3166-1 alpha-2
    industry        TEXT,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Job families (e.g. Engineering, Marketing, Finance)
CREATE TABLE job_family (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES job_family(id),  -- for sub-families
    name            TEXT NOT NULL,
    code            TEXT,                    -- internal code e.g. 'ENG', 'MKT'
    description     TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_job_family_tenant ON job_family(tenant_id);
CREATE INDEX idx_job_family_parent ON job_family(parent_id);

-- Grade/level hierarchy (e.g. IC1-IC8, M1-M4)
CREATE TABLE job_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,           -- 'IC1', 'IC2', 'Senior', 'Staff', 'Principal'
    level_number    INT NOT NULL,            -- numeric sort order
    track           TEXT NOT NULL CHECK (track IN ('individual_contributor', 'management', 'executive')),
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_job_level_tenant ON job_level(tenant_id);

-- Pay bands linked to levels
CREATE TABLE pay_band (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_level_id    UUID NOT NULL REFERENCES job_level(id),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',  -- ISO 4217
    min_salary      NUMERIC(12,2),
    mid_salary      NUMERIC(12,2),
    max_salary      NUMERIC(12,2),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_pay_band_level ON pay_band(job_level_id);
```

### Job Profiles (Roles)

```sql
-- A job profile defines a specific role within a job family and level
CREATE TABLE job_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_family_id   UUID NOT NULL REFERENCES job_family(id),
    job_level_id    UUID NOT NULL REFERENCES job_level(id),
    title           TEXT NOT NULL,
    code            TEXT,                    -- internal job code for HRIS sync
    summary         TEXT,                    -- short description
    description     TEXT,                    -- full role description
    onet_soc_code   TEXT REFERENCES onet_occupation(soc_code),  -- O*NET mapping
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_job_profile_tenant ON job_profile(tenant_id);
CREATE INDEX idx_job_profile_family ON job_profile(job_family_id);
CREATE INDEX idx_job_profile_level ON job_profile(job_level_id);

-- Skills required for a job profile with target proficiency
CREATE TABLE job_profile_skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    job_profile_id  UUID NOT NULL REFERENCES job_profile(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skill(id),
    proficiency_level_id UUID NOT NULL REFERENCES proficiency_level(id),
    importance      TEXT NOT NULL DEFAULT 'required' CHECK (importance IN ('required', 'preferred', 'nice_to_have')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (job_profile_id, skill_id)
);

CREATE INDEX idx_jps_profile ON job_profile_skill(job_profile_id);
CREATE INDEX idx_jps_skill ON job_profile_skill(skill_id);

-- Career pathways between job profiles
CREATE TABLE career_pathway (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    from_profile_id UUID NOT NULL REFERENCES job_profile(id),
    to_profile_id   UUID NOT NULL REFERENCES job_profile(id),
    pathway_type    TEXT NOT NULL CHECK (pathway_type IN ('promotion', 'lateral', 'specialisation')),
    typical_duration_months INT,
    description     TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (from_profile_id, to_profile_id)
);

CREATE INDEX idx_pathway_from ON career_pathway(from_profile_id);
CREATE INDEX idx_pathway_to ON career_pathway(to_profile_id);
```

---

## Employee & Assessment Tables

```sql
-- Employees (synced from HRIS or imported via CSV)
CREATE TABLE employee (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    external_id     TEXT,                    -- HRIS employee ID
    email           TEXT,
    full_name       TEXT NOT NULL,
    job_profile_id  UUID REFERENCES job_profile(id),
    department      TEXT,
    location        TEXT,
    hire_date       DATE,
    status          TEXT NOT NULL DEFAULT 'active' CHECK (status IN ('active', 'inactive', 'terminated')),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_employee_tenant ON employee(tenant_id);
CREATE INDEX idx_employee_profile ON employee(job_profile_id);
CREATE INDEX idx_employee_external ON employee(tenant_id, external_id);

-- Employee skill assessments
CREATE TABLE employee_skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    employee_id     UUID NOT NULL REFERENCES employee(id) ON DELETE CASCADE,
    skill_id        UUID NOT NULL REFERENCES skill(id),
    proficiency_level_id UUID NOT NULL REFERENCES proficiency_level(id),
    assessment_type TEXT NOT NULL CHECK (assessment_type IN ('self', 'manager', 'peer', 'assessment', 'inferred')),
    assessed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    evidence_url    TEXT,
    notes           TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (employee_id, skill_id, assessment_type)
);

CREATE INDEX idx_employee_skill_employee ON employee_skill(employee_id);
CREATE INDEX idx_employee_skill_skill ON employee_skill(skill_id);
```

---

## Gap Analysis Views

```sql
-- View: Skills gap for each employee against their assigned role
CREATE VIEW v_employee_skill_gap AS
SELECT
    e.id AS employee_id,
    e.full_name,
    e.tenant_id,
    jp.id AS job_profile_id,
    jp.title AS job_profile_title,
    s.id AS skill_id,
    s.name AS skill_name,
    jps.importance,
    pl_required.level_number AS required_level,
    pl_required.name AS required_level_name,
    COALESCE(pl_actual.level_number, 0) AS actual_level,
    COALESCE(pl_actual.name, 'Not assessed') AS actual_level_name,
    COALESCE(pl_required.level_number, 0) - COALESCE(pl_actual.level_number, 0) AS gap
FROM employee e
JOIN job_profile jp ON jp.id = e.job_profile_id
JOIN job_profile_skill jps ON jps.job_profile_id = jp.id
JOIN skill s ON s.id = jps.skill_id
JOIN proficiency_level pl_required ON pl_required.id = jps.proficiency_level_id
LEFT JOIN employee_skill es ON es.employee_id = e.id AND es.skill_id = s.id
LEFT JOIN proficiency_level pl_actual ON pl_actual.id = es.proficiency_level_id
WHERE e.status = 'active';

-- View: Organisational skill coverage heat map
CREATE VIEW v_org_skill_coverage AS
SELECT
    e.tenant_id,
    s.id AS skill_id,
    s.name AS skill_name,
    sc.name AS category_name,
    COUNT(DISTINCT e.id) AS employees_with_skill,
    COUNT(DISTINCT CASE WHEN pl.level_number >= 4 THEN e.id END) AS experts,
    COUNT(DISTINCT CASE WHEN pl.level_number >= 3 THEN e.id END) AS proficient,
    ROUND(AVG(pl.level_number), 2) AS avg_proficiency
FROM employee e
JOIN employee_skill es ON es.employee_id = e.id
JOIN skill s ON s.id = es.skill_id
JOIN proficiency_level pl ON pl.id = es.proficiency_level_id
LEFT JOIN skill_subcategory ssc ON ssc.id = s.subcategory_id
LEFT JOIN skill_category sc ON sc.id = ssc.category_id
WHERE e.status = 'active'
GROUP BY e.tenant_id, s.id, s.name, sc.name;
```

---

## Multi-Tenancy & Access Control

```sql
-- Row-level security for tenant isolation
ALTER TABLE skill ENABLE ROW LEVEL SECURITY;
ALTER TABLE job_family ENABLE ROW LEVEL SECURITY;
ALTER TABLE job_level ENABLE ROW LEVEL SECURITY;
ALTER TABLE job_profile ENABLE ROW LEVEL SECURITY;
ALTER TABLE employee ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation_skill ON skill
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation_job_family ON job_family
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation_job_level ON job_level
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation_job_profile ON job_profile
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);
CREATE POLICY tenant_isolation_employee ON employee
    USING (tenant_id = current_setting('app.current_tenant_id')::UUID);

-- RBAC roles
CREATE TABLE app_role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,           -- 'admin', 'hr_manager', 'employee', 'viewer'
    permissions     TEXT[] NOT NULL DEFAULT '{}',  -- e.g. '{taxonomy:write, profiles:read, employees:read}'
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    role_id         UUID NOT NULL REFERENCES app_role(id),
    employee_id     UUID REFERENCES employee(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

---

## Integration & Export Tables

```sql
-- HRIS integration connections
CREATE TABLE hris_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider        TEXT NOT NULL,           -- 'workday', 'bamboohr', 'sap_sf', 'merge', 'csv'
    status          TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'active', 'error', 'disabled')),
    config          JSONB NOT NULL DEFAULT '{}',  -- connection config (tokens stored encrypted)
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Sync log for HRIS imports
CREATE TABLE hris_sync_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    connection_id   UUID NOT NULL REFERENCES hris_connection(id),
    sync_type       TEXT NOT NULL,           -- 'employees', 'job_codes', 'full'
    status          TEXT NOT NULL CHECK (status IN ('running', 'completed', 'failed')),
    records_synced  INT DEFAULT 0,
    error_message   TEXT,
    started_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

-- Export jobs (HR-XML, CSV, JSON)
CREATE TABLE export_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    export_type     TEXT NOT NULL,           -- 'hr_xml', 'csv', 'json', 'schema_org'
    scope           TEXT NOT NULL,           -- 'taxonomy', 'profiles', 'employees', 'gap_analysis'
    status          TEXT NOT NULL DEFAULT 'pending',
    file_url        TEXT,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);
```

---

## Taxonomy Hierarchy Traversal Example

```sql
-- Recursive CTE to find all sub-families under a parent job family
WITH RECURSIVE family_tree AS (
    SELECT id, name, parent_id, 0 AS depth
    FROM job_family
    WHERE id = '...'  -- root family UUID
    
    UNION ALL
    
    SELECT jf.id, jf.name, jf.parent_id, ft.depth + 1
    FROM job_family jf
    JOIN family_tree ft ON jf.parent_id = ft.id
)
SELECT * FROM family_tree ORDER BY depth, name;

-- Find all skills related to a given skill across taxonomies
SELECT
    s2.name AS related_skill,
    tc.match_type,
    tc.confidence
FROM taxonomy_crosswalk tc
JOIN skill s2 ON s2.id = tc.target_id AND tc.target_type = 'lightcast_skill'
WHERE tc.source_type = 'lightcast_skill'
  AND tc.source_id = '...'  -- source skill UUID
ORDER BY tc.confidence DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Taxonomy (skills, categories, subcategories) | 4 | Lightcast three-tier hierarchy + proficiency scales |
| Standards Reference (O\*NET, ESCO, crosswalks) | 3 | External taxonomy data + cross-mapping |
| Job Architecture (families, levels, profiles, bands) | 6 | Grade/level hierarchy with pay bands and career pathways |
| Employees & Assessments | 2 | Employee records + skill assessments |
| Access Control (tenants, roles, users) | 3 | Multi-tenant with RBAC |
| Integration (HRIS, exports, sync logs) | 3 | HRIS connectors + export jobs |
| **Total** | **~21 core tables** | Plus views for gap analysis and coverage heat maps |

---

## Key Design Decisions

1. **UUID primary keys throughout** — enables distributed ID generation for self-hosted deployments and prevents ID collision when merging tenant data.

2. **tenant_id on every business table with RLS** — PostgreSQL row-level security policies enforce tenant isolation at the database level, not just the application layer; prevents data leaks even in the case of application bugs.

3. **Separate tables for each taxonomy tier** — `skill_category > skill_subcategory > skill` mirrors the Lightcast three-tier hierarchy exactly, making bulk import from the Lightcast API a straightforward ETL operation.

4. **Crosswalk table with confidence scores** — the `taxonomy_crosswalk` table supports both manually-curated exact mappings and AI-generated approximate mappings, with a confidence score to distinguish them.

5. **Configurable proficiency scales** — different organisations use different proficiency models (5-point, 7-level SFIA, 4-level Dreyfus); the `proficiency_scale` and `proficiency_level` tables allow each tenant to configure their own.

6. **Explicit career pathway table** — career paths between job profiles are stored as directed edges with pathway type and typical duration, enabling career path visualisation queries.

7. **Assessment type tracking** — `employee_skill.assessment_type` distinguishes self-assessment from manager assessment from validated assessment, supporting the gap analysis accuracy hierarchy recommended by skills management best practices.

8. **O\*NET SOC code on job profiles** — directly linking job profiles to O\*NET occupational codes enables Schema.org JobPosting export with the `occupationalCategory` property, improving SEO findability.

9. **Pay band temporal validity** — `effective_from` / `effective_to` dates on pay bands support compensation benchmarking across time periods without overwriting historical band data.

10. **HRIS integration as a first-class concern** — dedicated `hris_connection` and `hris_sync_log` tables support the Merge/Knit unified API pattern and provide audit trails for data synchronisation operations.
