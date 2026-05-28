# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: Job Architecture & Skills Taxonomy · Created: 2026-05-12

## Philosophy

This model takes a pragmatic middle path: core structural fields that are queried, filtered, and joined frequently are stored as typed relational columns, while variable, domain-specific, or evolving attributes are stored in JSONB columns on the same tables. The key insight is that a job architecture platform must serve organisations across wildly different industries, regions, and regulatory environments. A manufacturing company in Germany needs different job profile attributes than a fintech startup in Singapore. Rather than trying to anticipate every possible field in a normalised schema, JSONB columns absorb the variability while relational columns enforce the structural invariants.

This approach is inspired by how modern SaaS platforms (Stripe, Shopify, Notion) handle extensibility: a solid relational backbone for the things that are always true, with JSONB "metadata" or "properties" columns for everything else. It is also aligned with the HR Open Standards direction, which is moving from rigid XML schemas toward flexible JSON-based interchange formats that can accommodate optional and extension fields.

The hybrid model is particularly well-suited for rapid MVP development. The initial schema can be deployed with a handful of tables, and new attributes can be added to JSONB columns without schema migrations. As patterns stabilise, frequently-queried JSONB fields can be promoted to dedicated columns with generated columns or partial indexes, providing a smooth evolution path from prototype to production.

**Best for:** Rapid MVP development, multi-industry/multi-region deployments where job profile attributes vary significantly, and teams that want to iterate quickly without frequent schema migrations.

**Trade-offs:**
- Pro: Fewest tables (~12-15); fastest time to MVP
- Pro: New attributes can be added without schema migrations (just add to JSONB)
- Pro: Multi-region/multi-industry flexibility: jurisdiction-specific fields live in JSONB
- Pro: Easy to import diverse data sources (CSV, HRIS APIs) with varying schemas
- Pro: Generated columns and GIN indexes provide relational-speed queries on JSONB fields
- Con: JSONB fields are not FK-enforced; application-level validation is required
- Con: Schema documentation must be maintained manually (no DDL self-documents JSONB structure)
- Con: JSONB queries can be slower than typed column queries for complex aggregations
- Con: Risk of "schema drift" if JSONB structure is not governed by application-level schemas

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| O\*NET Content Model | Core occupation identifiers (SOC code, title, job zone) are relational columns; the full 277-descriptor attribute set is stored in `onet_attributes` JSONB |
| ESCO v1.2 | ESCO URI and preferred label are relational columns; multilingual labels (27 languages) stored in `translations` JSONB |
| Lightcast Open Skills | Skill ID, name, and type are relational columns; Lightcast-specific metadata (frequency, demand signals) stored in JSONB |
| SFIA 9 | SFIA skill code and level are relational columns; descriptive attributes stored in JSONB |
| HR Open Standards | JSONB structure follows emerging HR-JSON schema patterns for competency definitions |
| IEEE 1484.20.1 | Competency definition JSONB structure includes IEEE-aligned fields (identifier, title, description, metadata) |
| ISO 30414 | Human capital metrics are computed from relational columns; ISO 30414-specific reporting fields in tenant `settings` JSONB |
| Schema.org | Job profile JSONB includes `schema_org` sub-object with pre-formatted Schema.org JobPosting fields |

---

## Core Tables

### Multi-Tenancy

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;

CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2),                 -- ISO 3166-1 alpha-2
    industry        TEXT,
    settings        JSONB NOT NULL DEFAULT '{}',
    -- Example settings:
    -- {
    --   "proficiency_scale": {
    --     "name": "5-point",
    --     "levels": [
    --       {"number": 1, "name": "Awareness", "description": "Basic familiarity"},
    --       {"number": 2, "name": "Beginner", "description": "Can perform with guidance"},
    --       {"number": 3, "name": "Intermediate", "description": "Independent performance"},
    --       {"number": 4, "name": "Advanced", "description": "Can mentor others"},
    --       {"number": 5, "name": "Expert", "description": "Industry authority"}
    --     ]
    --   },
    --   "job_profile_custom_fields": [
    --     {"key": "flsa_status", "label": "FLSA Status", "type": "select", "options": ["exempt", "non-exempt"]},
    --     {"key": "eeo_category", "label": "EEO Category", "type": "text"}
    --   ],
    --   "skill_assessment_methods": ["self", "manager", "peer", "assessment"],
    --   "default_currency": "USD",
    --   "iso_30414_enabled": true
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### Skills Taxonomy

```sql
-- Unified skill table with JSONB for taxonomy-specific attributes
CREATE TABLE skill (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),

    -- Core relational columns (always present, always queried)
    name            TEXT NOT NULL,
    skill_type      TEXT NOT NULL CHECK (skill_type IN ('specialised', 'common', 'software', 'certification')),
    category        TEXT,                    -- top-level category name
    subcategory     TEXT,                    -- second-level grouping
    source          TEXT NOT NULL DEFAULT 'lightcast',
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,

    -- External taxonomy identifiers (relational for joins)
    lightcast_id    TEXT,                    -- Lightcast skill ID
    onet_element_id TEXT,                    -- O*NET element ID (for knowledge/skill/ability)
    esco_uri        TEXT,                    -- ESCO concept URI
    sfia_code       TEXT,                    -- SFIA skill code (e.g. 'PROG')

    -- JSONB for variable and taxonomy-specific attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example for a Lightcast-sourced skill:
    -- {
    --   "lightcast": {
    --     "category_id": "STK-1234",
    --     "subcategory_id": "STS-5678",
    --     "frequency": 0.85,
    --     "demand_trend": "rising",
    --     "last_updated": "2026-05-01"
    --   },
    --   "onet_crosswalk": {
    --     "element_name": "Programming",
    --     "match_type": "broad",
    --     "confidence": 0.82
    --   },
    --   "esco_crosswalk": {
    --     "preferred_label": "software development",
    --     "match_type": "exact"
    --   },
    --   "related_skills": ["uuid-1", "uuid-2", "uuid-3"],
    --   "prerequisite_skills": ["uuid-4"],
    --   "tags": ["backend", "data-engineering"],
    --   "aliases": ["Python programming", "Python 3", "CPython"]
    -- }

    -- AI-managed fields
    is_emerging     BOOLEAN NOT NULL DEFAULT FALSE,
    deprecated_at   TIMESTAMPTZ,
    ai_confidence   NUMERIC(3,2),            -- confidence of AI-generated skill entries

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_skill_tenant ON skill(tenant_id);
CREATE INDEX idx_skill_type ON skill(tenant_id, skill_type);
CREATE INDEX idx_skill_category ON skill(tenant_id, category);
CREATE INDEX idx_skill_source ON skill(tenant_id, source);
CREATE INDEX idx_skill_lightcast ON skill(lightcast_id) WHERE lightcast_id IS NOT NULL;
CREATE INDEX idx_skill_onet ON skill(onet_element_id) WHERE onet_element_id IS NOT NULL;
CREATE INDEX idx_skill_esco ON skill(esco_uri) WHERE esco_uri IS NOT NULL;
CREATE INDEX idx_skill_name_trgm ON skill USING gin (name gin_trgm_ops);
CREATE INDEX idx_skill_attributes ON skill USING gin (attributes);

-- Generated column example: extract demand_trend for filtering
ALTER TABLE skill ADD COLUMN demand_trend TEXT
    GENERATED ALWAYS AS (attributes->'lightcast'->>'demand_trend') STORED;
CREATE INDEX idx_skill_demand ON skill(tenant_id, demand_trend) WHERE demand_trend IS NOT NULL;
```

### Job Architecture

```sql
-- Job families with optional hierarchy
CREATE TABLE job_family (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    parent_id       UUID REFERENCES job_family(id),
    name            TEXT NOT NULL,
    code            TEXT,
    description     TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "function": "Technology",
    --   "cost_center": "CC-4500",
    --   "headcount_budget": 150
    -- }
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_jf_tenant ON job_family(tenant_id);
CREATE INDEX idx_jf_parent ON job_family(parent_id);

-- Job levels / grades (unified table)
CREATE TABLE job_level (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    level_number    INT NOT NULL,
    track           TEXT NOT NULL CHECK (track IN ('individual_contributor', 'management', 'executive')),
    description     TEXT,
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "pay_band": {"currency": "USD", "min": 120000, "mid": 150000, "max": 180000},
    --   "pay_band_eur": {"currency": "EUR", "min": 110000, "mid": 138000, "max": 165000},
    --   "equity_target_pct": 0.15,
    --   "title_prefix": "Senior",
    --   "sfia_equivalent_level": 5,
    --   "hay_grade": "18"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE INDEX idx_jl_tenant ON job_level(tenant_id);

-- Job profiles (roles) — the core of job architecture
CREATE TABLE job_profile (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_family_id   UUID NOT NULL REFERENCES job_family(id),
    job_level_id    UUID NOT NULL REFERENCES job_level(id),

    -- Core relational columns
    title           TEXT NOT NULL,
    code            TEXT,                    -- HRIS job code
    summary         TEXT,
    description     TEXT,
    status          TEXT NOT NULL DEFAULT 'draft' CHECK (status IN ('draft', 'active', 'archived')),

    -- External taxonomy links
    onet_soc_code   TEXT,                    -- O*NET-SOC code for this role

    -- JSONB for variable role attributes
    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "responsibilities": [
    --     "Design and implement scalable backend services",
    --     "Mentor junior engineers on best practices"
    --   ],
    --   "qualifications": {
    --     "required": ["Bachelor's in CS or equivalent", "5+ years backend experience"],
    --     "preferred": ["Master's degree", "Distributed systems experience"]
    --   },
    --   "location_requirements": {
    --     "remote_eligible": true,
    --     "regions": ["US", "EU"],
    --     "visa_sponsorship": false
    --   },
    --   "compliance": {
    --     "flsa_status": "exempt",
    --     "eeo_category": "2",
    --     "safety_sensitive": false
    --   },
    --   "schema_org": {
    --     "@type": "JobPosting",
    --     "occupationalCategory": "15-1252.00",
    --     "skills": ["Python", "PostgreSQL", "Distributed Systems"]
    --   },
    --   "custom_fields": {
    --     "travel_pct": "10%",
    --     "clearance_required": false
    --   }
    -- }

    -- Skills requirements stored as JSONB array for flexibility
    required_skills JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {"skill_id": "uuid-1", "name": "Python", "required_level": 4, "importance": "required"},
    --   {"skill_id": "uuid-2", "name": "PostgreSQL", "required_level": 3, "importance": "required"},
    --   {"skill_id": "uuid-3", "name": "Kubernetes", "required_level": 2, "importance": "preferred"}
    -- ]

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_jp_tenant ON job_profile(tenant_id);
CREATE INDEX idx_jp_family ON job_profile(job_family_id);
CREATE INDEX idx_jp_level ON job_profile(job_level_id);
CREATE INDEX idx_jp_status ON job_profile(tenant_id, status);
CREATE INDEX idx_jp_attributes ON job_profile USING gin (attributes);
CREATE INDEX idx_jp_required_skills ON job_profile USING gin (required_skills);

-- Career pathways
CREATE TABLE career_pathway (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    from_profile_id UUID NOT NULL REFERENCES job_profile(id),
    to_profile_id   UUID NOT NULL REFERENCES job_profile(id),
    pathway_type    TEXT NOT NULL CHECK (pathway_type IN ('promotion', 'lateral', 'specialisation')),
    typical_duration_months INT,
    skill_gaps      JSONB NOT NULL DEFAULT '[]',
    -- Auto-computed: skills needed for target that source doesn't require
    -- [{"skill_id": "uuid-1", "name": "System Design", "gap_levels": 2}]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (from_profile_id, to_profile_id)
);

CREATE INDEX idx_cp_from ON career_pathway(from_profile_id);
CREATE INDEX idx_cp_to ON career_pathway(to_profile_id);
```

### Employees & Assessments

```sql
-- Employee records
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

    -- Employee skill profile as JSONB
    skill_profile   JSONB NOT NULL DEFAULT '[]',
    -- Example:
    -- [
    --   {
    --     "skill_id": "uuid-1",
    --     "name": "Python",
    --     "proficiency_level": 4,
    --     "assessment_type": "manager",
    --     "assessed_at": "2026-03-15",
    --     "evidence_url": "https://...",
    --     "history": [
    --       {"level": 3, "assessed_at": "2025-06-01", "type": "self"},
    --       {"level": 4, "assessed_at": "2026-03-15", "type": "manager"}
    --     ]
    --   },
    --   {
    --     "skill_id": "uuid-2",
    --     "name": "PostgreSQL",
    --     "proficiency_level": 3,
    --     "assessment_type": "self",
    --     "assessed_at": "2026-01-10"
    --   }
    -- ]

    attributes      JSONB NOT NULL DEFAULT '{}',
    -- Example:
    -- {
    --   "manager_id": "uuid-manager",
    --   "cost_center": "CC-4500",
    --   "certifications": ["AWS Solutions Architect", "PMP"],
    --   "languages": ["en", "de"],
    --   "education": {"degree": "MSc Computer Science", "institution": "TU Munich", "year": 2018}
    -- }

    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_emp_tenant ON employee(tenant_id);
CREATE INDEX idx_emp_profile ON employee(job_profile_id);
CREATE INDEX idx_emp_external ON employee(tenant_id, external_id);
CREATE INDEX idx_emp_status ON employee(tenant_id, status);
CREATE INDEX idx_emp_skills ON employee USING gin (skill_profile);
CREATE INDEX idx_emp_attributes ON employee USING gin (attributes);
```

### Integration

```sql
-- HRIS connections
CREATE TABLE hris_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    config          JSONB NOT NULL DEFAULT '{}',
    field_mapping   JSONB NOT NULL DEFAULT '{}',
    -- Field mapping allows flexible HRIS field -> local field mapping:
    -- {
    --   "employee_id_field": "workdayId",
    --   "name_field": "preferredName",
    --   "email_field": "emailAddress",
    --   "department_field": "supervisoryOrganization",
    --   "custom_mappings": {
    --     "attributes.cost_center": "costCenter",
    --     "attributes.business_unit": "businessUnit"
    --   }
    -- }
    last_sync_at    TIMESTAMPTZ,
    sync_stats      JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Import/export jobs
CREATE TABLE data_job (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_type        TEXT NOT NULL,           -- 'import', 'export'
    data_type       TEXT NOT NULL,           -- 'taxonomy', 'profiles', 'employees', 'gap_analysis'
    format          TEXT NOT NULL,           -- 'csv', 'json', 'hr_xml', 'schema_org'
    status          TEXT NOT NULL DEFAULT 'pending',
    config          JSONB NOT NULL DEFAULT '{}',
    result          JSONB DEFAULT '{}',
    -- Example result:
    -- {"records_processed": 1500, "errors": 3, "file_url": "https://..."}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    completed_at    TIMESTAMPTZ
);

-- Access control
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'viewer' CHECK (role IN ('admin', 'hr_manager', 'editor', 'viewer', 'employee')),
    employee_id     UUID REFERENCES employee(id),
    permissions     JSONB NOT NULL DEFAULT '{}',
    -- Override permissions beyond role defaults:
    -- {"can_export": true, "can_manage_taxonomy": false}
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

---

## JSONB Query Examples

### Find skills by demand trend

```sql
-- Using the generated column
SELECT name, category, demand_trend
FROM skill
WHERE tenant_id = :tenant_id
  AND demand_trend = 'rising'
  AND is_active = TRUE
ORDER BY name;

-- Using JSONB containment directly
SELECT name, category, attributes->'lightcast'->>'frequency' AS frequency
FROM skill
WHERE tenant_id = :tenant_id
  AND attributes @> '{"lightcast": {"demand_trend": "rising"}}'
ORDER BY (attributes->'lightcast'->>'frequency')::NUMERIC DESC;
```

### Skills gap analysis from JSONB profiles

```sql
-- Compare employee skill profile against job profile requirements
WITH required AS (
    SELECT
        (elem->>'skill_id')::UUID AS skill_id,
        elem->>'name' AS skill_name,
        (elem->>'required_level')::INT AS required_level,
        elem->>'importance' AS importance
    FROM job_profile jp,
         jsonb_array_elements(jp.required_skills) AS elem
    WHERE jp.id = :job_profile_id
),
actual AS (
    SELECT
        (elem->>'skill_id')::UUID AS skill_id,
        (elem->>'proficiency_level')::INT AS actual_level
    FROM employee e,
         jsonb_array_elements(e.skill_profile) AS elem
    WHERE e.id = :employee_id
)
SELECT
    r.skill_name,
    r.importance,
    r.required_level,
    COALESCE(a.actual_level, 0) AS actual_level,
    r.required_level - COALESCE(a.actual_level, 0) AS gap
FROM required r
LEFT JOIN actual a ON a.skill_id = r.skill_id
ORDER BY gap DESC, r.importance;
```

### Find employees with a specific skill at a minimum level

```sql
-- Using JSONB array containment with subquery
SELECT e.id, e.full_name, e.department,
       elem->>'name' AS skill_name,
       (elem->>'proficiency_level')::INT AS proficiency
FROM employee e,
     jsonb_array_elements(e.skill_profile) AS elem
WHERE e.tenant_id = :tenant_id
  AND e.status = 'active'
  AND (elem->>'skill_id')::UUID = :target_skill_id
  AND (elem->>'proficiency_level')::INT >= :min_level
ORDER BY (elem->>'proficiency_level')::INT DESC;
```

### Organisational skill heat map

```sql
-- Aggregate skill coverage across the organisation
SELECT
    s.category,
    s.name AS skill_name,
    COUNT(DISTINCT e.id) AS employees_with_skill,
    ROUND(AVG((elem->>'proficiency_level')::NUMERIC), 2) AS avg_proficiency,
    COUNT(DISTINCT CASE WHEN (elem->>'proficiency_level')::INT >= 4 THEN e.id END) AS experts
FROM employee e,
     jsonb_array_elements(e.skill_profile) AS elem
JOIN skill s ON s.id = (elem->>'skill_id')::UUID
WHERE e.tenant_id = :tenant_id
  AND e.status = 'active'
GROUP BY s.category, s.name
ORDER BY employees_with_skill DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Tenancy | 1 | Tenant with settings JSONB (includes proficiency scales, custom fields) |
| Taxonomy | 1 | Unified skill table with JSONB attributes for cross-taxonomy data |
| Job Architecture | 4 | Job families, levels, profiles, career pathways |
| Employees | 1 | Employee records with JSONB skill profiles |
| Integration | 2 | HRIS connections + data import/export jobs |
| Access Control | 1 | Users with role and JSONB permission overrides |
| **Total** | **~10 tables** | Minimal table count; JSONB absorbs variability |

---

## Key Design Decisions

1. **Skill profile as JSONB array on the employee record** — instead of a separate `employee_skill` junction table, the employee's entire skill profile is stored as a JSONB array on the employee record. This eliminates a high-cardinality junction table, makes employee profile reads a single row fetch, and allows assessment history to be nested within each skill entry. The trade-off is that skill-centric queries ("find all employees with skill X") require JSONB array scanning, mitigated by GIN indexes.

2. **Required skills as JSONB on job profiles** — similarly, skill requirements are stored as a JSONB array on `job_profile` rather than in a junction table. This keeps the job profile self-contained and makes it easy to export complete profiles as JSON documents for HR-XML/Schema.org interchange.

3. **Generated columns for frequently-filtered JSONB fields** — PostgreSQL generated columns extract hot JSONB paths into typed, indexable columns (e.g., `demand_trend`). This provides relational query performance for the most common filters while keeping the source of truth in JSONB.

4. **Proficiency scales in tenant settings** — instead of separate `proficiency_scale` and `proficiency_level` tables, the scale is stored in the tenant's `settings` JSONB. This reduces table count and simplifies the API, at the cost of not being able to FK-enforce proficiency levels.

5. **Multi-currency pay bands in JSONB** — the `job_level.attributes` JSONB can contain pay bands for multiple currencies (USD, EUR, GBP) without requiring separate pay band tables per currency. This is critical for global organisations.

6. **Flexible HRIS field mapping** — the `hris_connection.field_mapping` JSONB allows each tenant to map their HRIS field names to the local schema without code changes, supporting the diversity of field naming across Workday, BambooHR, SAP SuccessFactors, etc.

7. **Career pathway skill gaps auto-computed** — the `career_pathway.skill_gaps` JSONB is computed by the application when pathways are created or updated, pre-calculating the skills gap between source and target roles so that career path visualisation queries don't need to join against skill requirements at read time.

8. **Custom fields via tenant configuration** — the `tenant.settings.job_profile_custom_fields` array defines organisation-specific fields (FLSA status, EEO category, clearance level) that the UI renders dynamically. Values are stored in `job_profile.attributes.custom_fields`. This avoids per-tenant schema customisation.

9. **Assessment history embedded in skill profile** — each skill entry in the employee's `skill_profile` JSONB can include a `history` array of past assessments, providing temporal tracking without a separate history table. This is sufficient for most gap analysis use cases but is not as rigorous as the event-sourced model for full audit trail requirements.

10. **Schema.org pre-formatted in attributes** — job profiles include a `schema_org` sub-object in their JSONB that contains pre-formatted Schema.org JobPosting structured data, making SEO-optimised job posting export a simple JSONB extraction rather than a complex transformation query.
