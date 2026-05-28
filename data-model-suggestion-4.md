# Data Model Suggestion 4: Event-Sourced / Audit-First

> Project: Job Architecture & Skills Taxonomy · Created: 2026-05-12

## Philosophy

This model treats every change to the skills taxonomy, job architecture, and employee skill profiles as an immutable event recorded in an append-only event store. The current state of any entity is derived by replaying its event stream. Materialised views (read models) are projected from the event store to serve read-heavy queries like gap analysis, skill coverage heat maps, and career path visualisations. The pattern is CQRS (Command Query Responsibility Segregation): writes go to the event store, reads come from projections.

The core motivation is that a job architecture platform manages data with strong temporal and audit requirements. When an employee's skill assessment changes, the previous assessment is not "wrong" -- it was the correct assessment at the time. When a job profile's skill requirements are updated, the old requirements are still needed for historical compliance reporting. When the skills taxonomy evolves (new skills emerge, old skills are deprecated), the history of that evolution is itself valuable data for workforce planning analytics. Event sourcing captures all of this naturally, because events are never deleted or modified.

This approach is particularly well-suited for the AI-native aspects of the project. The self-maintaining taxonomy agent can emit `SkillProposed`, `SkillApproved`, `SkillDeprecated` events as it monitors external sources. The AI role profiling engine can emit `RoleSkillSuggested` events that are reviewed and accepted or rejected by humans. Every AI action is captured in the event stream, providing full transparency and auditability of AI-driven changes -- a critical trust requirement for HR systems.

**Best for:** Organisations with strict audit trail requirements (ISO 30414, SOX, GDPR), temporal query needs ("what skills were required for this role last year?"), and AI-transparency requirements.

**Trade-offs:**
- Pro: Complete, immutable audit trail of every change -- no data is ever lost
- Pro: Temporal queries are natural ("what was the org's skill profile on 2025-06-01?")
- Pro: Full AI transparency -- every AI suggestion and human approval is an event
- Pro: Event replay enables debugging, compliance investigation, and "what-if" scenario modelling
- Pro: Read models can be optimised independently of write models (CQRS)
- Con: Highest implementation complexity; requires event store, projectors, and read model management
- Con: Eventual consistency between event store and read models may surprise developers
- Con: Event schema evolution requires careful versioning (upcasting)
- Con: More infrastructure to operate (event store + projection engine + read database)
- Con: Simple CRUD operations require more code than in a traditional relational model

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| O\*NET Content Model | O\*NET taxonomy import is a batch of `OnetOccupationImported` events; updates produce `OnetOccupationUpdated` events, preserving version history |
| ESCO v1.2 | ESCO concept imports produce `EscoConceptImported` events with full multilingual labels in the event payload |
| Lightcast Open Skills | Bi-weekly Lightcast taxonomy updates produce `SkillImported`, `SkillUpdated`, `SkillDeprecated` events with diff information |
| SFIA 9 | SFIA skills imported as `SfiaSkillImported` events; version upgrades (SFIA 8 to 9) produce migration events |
| IEEE 1484.20.1 | Event payloads for skill-related events follow IEEE competency definition structure |
| ISO 30414 | Human capital reporting metrics are computed from event projections with full temporal accuracy; "what was our skill coverage on the reporting date?" |
| xAPI (IEEE 9274.1.1) | Learning events can be ingested as `LearningCompleted` events using xAPI statement structure (actor-verb-object) |
| HR Open Standards | Event payloads for competency changes can be serialised to HR-JSON format for system-to-system exchange |
| GDPR | Right-to-erasure implemented via `EmployeeDataErased` events that trigger projection cleanup without corrupting the event stream |

---

## Event Store

### Core Event Table

```sql
-- The event store: single append-only table for all domain events
CREATE TABLE event_store (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    stream_id       UUID NOT NULL,           -- aggregate root ID (the entity this event belongs to)
    stream_type     TEXT NOT NULL,            -- aggregate type: 'skill', 'job_profile', 'employee', 'taxonomy', 'job_family', 'job_level'
    event_type      TEXT NOT NULL,            -- the event name (see Event Catalog below)
    event_version   INT NOT NULL DEFAULT 1,  -- schema version of this event type
    sequence_number BIGINT NOT NULL,         -- per-stream sequence for ordering
    payload         JSONB NOT NULL,          -- event data
    metadata        JSONB NOT NULL DEFAULT '{}',
    -- Metadata example:
    -- {
    --   "user_id": "uuid-user",
    --   "user_email": "admin@example.com",
    --   "source": "ui",              -- 'ui', 'api', 'hris_sync', 'ai_agent', 'bulk_import'
    --   "ai_model": "gpt-4o",        -- if AI-generated
    --   "ai_confidence": 0.92,        -- if AI-generated
    --   "correlation_id": "uuid-req", -- request tracing
    --   "ip_address": "10.0.0.1"
    -- }
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, sequence_number)
);

-- Append-only: no UPDATE or DELETE policies
CREATE INDEX idx_es_tenant ON event_store(tenant_id);
CREATE INDEX idx_es_stream ON event_store(stream_id, sequence_number);
CREATE INDEX idx_es_type ON event_store(event_type);
CREATE INDEX idx_es_created ON event_store(created_at);
CREATE INDEX idx_es_tenant_type ON event_store(tenant_id, stream_type, created_at);

-- Partition by month for performance at scale
-- CREATE TABLE event_store_2026_05 PARTITION OF event_store
--     FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');
```

---

## Event Catalog

### Taxonomy Events

```
SkillCreated            — A new skill added to the taxonomy
SkillUpdated            — Skill attributes modified (name, description, category)
SkillDeprecated         — Skill marked as obsolete
SkillReactivated        — Previously deprecated skill brought back
SkillMerged             — Two skills merged into one (with redirect)
SkillCrosswalkAdded     — Crosswalk mapping added (e.g., Lightcast <-> O*NET)
SkillCrosswalkRemoved   — Crosswalk mapping removed
SkillRelationAdded      — Relationship added (related_to, prerequisite, adjacent)
SkillRelationRemoved    — Relationship removed
SkillCategoryCreated    — New top-level category added
SkillCategoryUpdated    — Category renamed or reorganised

-- AI-specific taxonomy events
SkillProposedByAI       — AI agent proposes a new emerging skill
SkillProposalApproved   — Human approves an AI-proposed skill
SkillProposalRejected   — Human rejects an AI-proposed skill
SkillDeprecationProposedByAI — AI suggests deprecating an obsolete skill
```

### Job Architecture Events

```
JobFamilyCreated        — New job family added
JobFamilyUpdated        — Job family renamed or restructured
JobLevelCreated         — New grade/level added
JobLevelUpdated         — Level attributes changed (description, pay band)
PayBandUpdated          — Compensation band changed for a level
JobProfileCreated       — New job profile (role) defined
JobProfileUpdated       — Role description, requirements, or attributes changed
JobProfileSkillAdded    — Skill requirement added to a job profile
JobProfileSkillRemoved  — Skill requirement removed from a job profile
JobProfileSkillLevelChanged — Required proficiency level changed for a skill
JobProfileActivated     — Profile moved from draft to active
JobProfileArchived      — Profile archived
CareerPathwayCreated    — Career path defined between two profiles
CareerPathwayRemoved    — Career path removed

-- AI-specific job architecture events
RoleSkillSuggestedByAI  — AI suggests adding a skill to a job profile
RoleSuggestionAccepted  — Human accepts AI skill suggestion
RoleSuggestionRejected  — Human rejects AI skill suggestion
```

### Employee Events

```
EmployeeOnboarded       — New employee record created (from HRIS sync or manual)
EmployeeUpdated         — Employee attributes changed (department, location, etc.)
EmployeeAssignedToRole  — Employee assigned to a job profile
EmployeeRoleChanged     — Employee moved to a different job profile
SkillAssessed           — Employee skill assessed (self, manager, peer, or test)
SkillAssessmentUpdated  — Assessment corrected or re-evaluated
CertificationEarned     — Employee earned a certification
CertificationExpired    — Employee certification expired
LearningCompleted       — Employee completed a learning resource
EmployeeTerminated      — Employee left the organisation
EmployeeDataErased      — GDPR right-to-erasure executed (PII removed)
```

### System Events

```
TaxonomyBulkImported    — Batch import from Lightcast/O*NET/ESCO
HrisSyncCompleted       — HRIS synchronisation completed
HrisSyncFailed          — HRIS synchronisation failed
ExportGenerated         — Data export created (CSV, HR-XML, JSON)
SnapshotTaken           — Read model snapshot for recovery
```

---

## Event Payload Examples

```sql
-- SkillCreated event
INSERT INTO event_store (tenant_id, stream_id, stream_type, event_type, sequence_number, payload, metadata)
VALUES (
    :tenant_id,
    :skill_id,           -- stream_id = the skill's UUID
    'skill',
    'SkillCreated',
    1,                   -- first event in this stream
    '{
        "id": "uuid-skill",
        "name": "Kubernetes",
        "skill_type": "specialised",
        "category": "Cloud & Infrastructure",
        "subcategory": "Container Orchestration",
        "source": "lightcast",
        "lightcast_id": "KS-12345",
        "description": "Container orchestration platform for automating deployment, scaling, and management"
    }',
    '{
        "user_id": "uuid-admin",
        "source": "bulk_import",
        "correlation_id": "uuid-import-batch-001"
    }'
);

-- SkillAssessed event
INSERT INTO event_store (tenant_id, stream_id, stream_type, event_type, sequence_number, payload, metadata)
VALUES (
    :tenant_id,
    :employee_id,        -- stream_id = the employee's UUID
    'employee',
    'SkillAssessed',
    42,                  -- 42nd event for this employee
    '{
        "employee_id": "uuid-employee",
        "skill_id": "uuid-kubernetes",
        "skill_name": "Kubernetes",
        "proficiency_level": 3,
        "previous_level": 2,
        "assessment_type": "manager",
        "evidence_url": "https://...",
        "notes": "Completed Kubernetes certification and led cluster migration project"
    }',
    '{
        "user_id": "uuid-manager",
        "source": "ui"
    }'
);

-- SkillProposedByAI event
INSERT INTO event_store (tenant_id, stream_id, stream_type, event_type, sequence_number, payload, metadata)
VALUES (
    :tenant_id,
    :proposed_skill_id,
    'skill',
    'SkillProposedByAI',
    1,
    '{
        "proposed_name": "Prompt Engineering",
        "proposed_category": "Artificial Intelligence",
        "proposed_type": "specialised",
        "evidence": {
            "job_posting_mentions": 12500,
            "growth_rate_90d": 0.45,
            "related_skills": ["Machine Learning", "Natural Language Processing", "LLM Fine-tuning"],
            "source_urls": ["https://lightcast.io/...", "https://linkedin.com/..."]
        },
        "confidence": 0.94
    }',
    '{
        "source": "ai_agent",
        "ai_model": "claude-opus-4-6",
        "ai_confidence": 0.94
    }'
);
```

---

## Read Models (Projections)

These tables are derived from the event store and can be rebuilt at any time by replaying events.

```sql
-- Projection: Current state of all skills
CREATE TABLE rm_skill (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    name            TEXT NOT NULL,
    skill_type      TEXT NOT NULL,
    category        TEXT,
    subcategory     TEXT,
    source          TEXT NOT NULL,
    lightcast_id    TEXT,
    onet_element_id TEXT,
    esco_uri        TEXT,
    sfia_code       TEXT,
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    is_emerging     BOOLEAN NOT NULL DEFAULT FALSE,
    deprecated_at   TIMESTAMPTZ,
    related_skills  UUID[],                  -- array of related skill IDs
    prerequisite_skills UUID[],
    crosswalks      JSONB DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,         -- last processed event sequence
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_skill_tenant ON rm_skill(tenant_id);
CREATE INDEX idx_rm_skill_type ON rm_skill(tenant_id, skill_type);
CREATE INDEX idx_rm_skill_category ON rm_skill(tenant_id, category);
CREATE INDEX idx_rm_skill_name_trgm ON rm_skill USING gin (name gin_trgm_ops);

-- Projection: Current state of all job profiles
CREATE TABLE rm_job_profile (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    job_family_id   UUID NOT NULL,
    job_family_name TEXT NOT NULL,
    job_level_id    UUID NOT NULL,
    job_level_name  TEXT NOT NULL,
    level_number    INT NOT NULL,
    track           TEXT NOT NULL,
    title           TEXT NOT NULL,
    code            TEXT,
    summary         TEXT,
    description     TEXT,
    onet_soc_code   TEXT,
    status          TEXT NOT NULL DEFAULT 'draft',
    required_skills JSONB NOT NULL DEFAULT '[]',
    attributes      JSONB NOT NULL DEFAULT '{}',
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_jp_tenant ON rm_job_profile(tenant_id);
CREATE INDEX idx_rm_jp_family ON rm_job_profile(job_family_id);
CREATE INDEX idx_rm_jp_status ON rm_job_profile(tenant_id, status);
CREATE INDEX idx_rm_jp_skills ON rm_job_profile USING gin (required_skills);

-- Projection: Current employee skill profiles
CREATE TABLE rm_employee (
    id              UUID PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    external_id     TEXT,
    email           TEXT,
    full_name       TEXT NOT NULL,
    job_profile_id  UUID,
    job_profile_title TEXT,
    department      TEXT,
    location        TEXT,
    hire_date       DATE,
    status          TEXT NOT NULL DEFAULT 'active',
    skill_profile   JSONB NOT NULL DEFAULT '[]',
    -- Same structure as Model 3: array of skill assessments with history
    certifications  JSONB NOT NULL DEFAULT '[]',
    last_event_seq  BIGINT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_emp_tenant ON rm_employee(tenant_id);
CREATE INDEX idx_rm_emp_profile ON rm_employee(job_profile_id);
CREATE INDEX idx_rm_emp_status ON rm_employee(tenant_id, status);
CREATE INDEX idx_rm_emp_skills ON rm_employee USING gin (skill_profile);

-- Projection: Skills gap analysis (pre-computed)
CREATE TABLE rm_skill_gap (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    employee_id     UUID NOT NULL,
    employee_name   TEXT NOT NULL,
    job_profile_id  UUID NOT NULL,
    job_profile_title TEXT NOT NULL,
    skill_id        UUID NOT NULL,
    skill_name      TEXT NOT NULL,
    importance      TEXT NOT NULL,
    required_level  INT NOT NULL,
    actual_level    INT NOT NULL DEFAULT 0,
    gap             INT NOT NULL,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_gap_tenant ON rm_skill_gap(tenant_id);
CREATE INDEX idx_rm_gap_employee ON rm_skill_gap(employee_id);
CREATE INDEX idx_rm_gap_profile ON rm_skill_gap(job_profile_id);
CREATE INDEX idx_rm_gap_skill ON rm_skill_gap(skill_id);
CREATE INDEX idx_rm_gap_severity ON rm_skill_gap(tenant_id, gap DESC);

-- Projection: Organisational capability coverage
CREATE TABLE rm_org_capability (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    skill_id        UUID NOT NULL,
    skill_name      TEXT NOT NULL,
    category        TEXT,
    employees_with_skill INT NOT NULL DEFAULT 0,
    experts_count   INT NOT NULL DEFAULT 0,
    avg_proficiency NUMERIC(3,2) NOT NULL DEFAULT 0,
    demand_trend    TEXT,
    risk_level      TEXT,                    -- 'critical', 'warning', 'adequate', 'strong'
    -- 'critical' = zero employees have this skill but it's required by active roles
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_cap_tenant ON rm_org_capability(tenant_id);
CREATE INDEX idx_rm_cap_risk ON rm_org_capability(tenant_id, risk_level);

-- Projection: AI action audit trail
CREATE TABLE rm_ai_actions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    event_id        UUID NOT NULL,
    event_type      TEXT NOT NULL,
    action_type     TEXT NOT NULL,           -- 'propose_skill', 'suggest_role_skill', 'deprecate_skill'
    status          TEXT NOT NULL DEFAULT 'pending',  -- 'pending', 'approved', 'rejected'
    ai_model        TEXT,
    ai_confidence   NUMERIC(3,2),
    proposal_data   JSONB NOT NULL,
    reviewed_by     UUID,
    reviewed_at     TIMESTAMPTZ,
    projected_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_rm_ai_tenant ON rm_ai_actions(tenant_id);
CREATE INDEX idx_rm_ai_status ON rm_ai_actions(tenant_id, status);
```

---

## Operational Tables

```sql
-- Tenant configuration (not event-sourced; operational config)
CREATE TABLE tenant (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name            TEXT NOT NULL,
    slug            TEXT NOT NULL UNIQUE,
    country_code    CHAR(2),
    industry        TEXT,
    settings        JSONB DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projection checkpoints (tracks which events have been projected)
CREATE TABLE projection_checkpoint (
    projection_name TEXT NOT NULL,
    tenant_id       UUID NOT NULL,
    last_event_id   UUID NOT NULL,
    last_sequence   BIGINT NOT NULL,
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (projection_name, tenant_id)
);

-- HRIS connections
CREATE TABLE hris_connection (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    provider        TEXT NOT NULL,
    status          TEXT NOT NULL DEFAULT 'pending',
    config          JSONB NOT NULL DEFAULT '{}',
    last_sync_at    TIMESTAMPTZ,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Access control
CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    role            TEXT NOT NULL DEFAULT 'viewer',
    permissions     JSONB NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

---

## Temporal Query Examples

### What skills were required for a role on a specific date?

```sql
-- Replay job profile events up to a point in time
WITH profile_events AS (
    SELECT
        event_type,
        payload,
        created_at,
        ROW_NUMBER() OVER (
            PARTITION BY stream_id
            ORDER BY sequence_number
        ) AS seq
    FROM event_store
    WHERE stream_id = :job_profile_id
      AND stream_type = 'job_profile'
      AND created_at <= '2025-06-01'::TIMESTAMPTZ
    ORDER BY sequence_number
)
SELECT
    payload->>'skill_name' AS skill_name,
    (payload->>'proficiency_level')::INT AS required_level,
    payload->>'importance' AS importance,
    created_at AS added_at
FROM profile_events
WHERE event_type = 'JobProfileSkillAdded'
  -- Exclude skills that were subsequently removed before the target date
  AND (payload->>'skill_id')::UUID NOT IN (
      SELECT (payload->>'skill_id')::UUID
      FROM profile_events
      WHERE event_type = 'JobProfileSkillRemoved'
  );
```

### How has an employee's skill profile evolved over time?

```sql
-- Timeline of skill assessments for an employee
SELECT
    created_at AS assessed_at,
    payload->>'skill_name' AS skill_name,
    (payload->>'proficiency_level')::INT AS new_level,
    (payload->>'previous_level')::INT AS previous_level,
    payload->>'assessment_type' AS assessment_type,
    metadata->>'source' AS source,
    metadata->>'user_email' AS assessed_by
FROM event_store
WHERE stream_id = :employee_id
  AND stream_type = 'employee'
  AND event_type = 'SkillAssessed'
ORDER BY created_at;
```

### What AI proposals are pending human review?

```sql
SELECT
    id,
    action_type,
    ai_model,
    ai_confidence,
    proposal_data->>'proposed_name' AS proposed_skill,
    proposal_data->'evidence'->>'job_posting_mentions' AS market_mentions,
    projected_at AS proposed_at
FROM rm_ai_actions
WHERE tenant_id = :tenant_id
  AND status = 'pending'
ORDER BY ai_confidence DESC;
```

### Rebuild a read model from scratch

```sql
-- Truncate and rebuild the skill read model from events
TRUNCATE rm_skill;

INSERT INTO rm_skill (id, tenant_id, name, skill_type, category, subcategory, source,
                      lightcast_id, is_active, is_emerging, last_event_seq, projected_at)
SELECT DISTINCT ON (stream_id)
    stream_id AS id,
    tenant_id,
    payload->>'name',
    payload->>'skill_type',
    payload->>'category',
    payload->>'subcategory',
    payload->>'source',
    payload->>'lightcast_id',
    CASE WHEN event_type = 'SkillDeprecated' THEN FALSE ELSE TRUE END,
    COALESCE((payload->>'is_emerging')::BOOLEAN, FALSE),
    sequence_number,
    now()
FROM event_store
WHERE stream_type = 'skill'
  AND event_type IN ('SkillCreated', 'SkillUpdated', 'SkillDeprecated', 'SkillReactivated')
ORDER BY stream_id, sequence_number DESC;

-- Note: A production projector would apply events sequentially, not use this
-- simplified snapshot approach. This is for illustration / disaster recovery.
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event Store | 1 | Single append-only table (partitioned by month) |
| Read Models (Projections) | 6 | Skills, job profiles, employees, skill gaps, org capability, AI actions |
| Operational (tenant, HRIS, auth) | 4 | Non-event-sourced operational tables |
| Projection Management | 1 | Checkpoint tracking for projectors |
| **Total** | **~12 tables** | 1 write table + 6 read tables + 5 operational |

---

## Key Design Decisions

1. **Single event store table** — all domain events across all aggregate types are stored in one table, partitioned by month. This simplifies infrastructure (one table to back up, replicate, and monitor) and enables cross-aggregate temporal queries ("what changed across the entire tenant on date X?").

2. **Stream-per-aggregate** — each entity (skill, job profile, employee) is an event stream identified by its UUID. The `sequence_number` provides strict ordering within a stream, enabling optimistic concurrency control (if the expected sequence number doesn't match, the write is rejected).

3. **Event metadata captures AI provenance** — every event's metadata includes the source (`ui`, `api`, `ai_agent`, `hris_sync`) and, for AI-generated events, the model name and confidence score. This provides a complete audit trail of AI-driven changes, which is critical for HR system trust.

4. **Separate event types for AI proposals vs. approvals** — `SkillProposedByAI` and `SkillProposalApproved`/`Rejected` are distinct events, creating a clear human-in-the-loop audit trail. This pattern satisfies regulatory requirements for AI transparency in HR decisions.

5. **Pre-computed gap analysis projection** — the `rm_skill_gap` read model is updated whenever employee skill assessments or job profile requirements change. Gap analysis queries read from this pre-computed table rather than computing gaps at query time, enabling real-time dashboards at scale.

6. **Capability risk level in projections** — the `rm_org_capability` read model includes a computed `risk_level` field that flags skills with zero internal coverage as "critical", enabling the AI-native organisational capability risk alerting feature described in the project's AI-native opportunity.

7. **GDPR compliance via erasure events** — `EmployeeDataErased` events trigger projectors to remove PII from read models while preserving the event stream's structural integrity. The event itself records that erasure occurred without retaining the erased data.

8. **Projection checkpoints for reliable rebuilds** — the `projection_checkpoint` table tracks the last event processed by each projector, enabling reliable incremental projection updates and full rebuilds from a known position.

9. **Event versioning for schema evolution** — `event_version` on each event enables upcasting: when the event schema changes (e.g., adding a new field to `SkillCreated`), old events are read with their original version and transformed to the current schema by the projector. This avoids the need to migrate historical events.

10. **Temporal queries without bi-temporal complexity** — the event store naturally provides both "what was true at time X" (replay events up to timestamp) and "when was this change recorded" (event `created_at`). This gives bi-temporal semantics without explicitly modelling valid-time and transaction-time columns, because in an event-sourced system, the event timestamp serves both purposes.
