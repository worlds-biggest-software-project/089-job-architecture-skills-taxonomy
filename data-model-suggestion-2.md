# Data Model Suggestion 2: Graph-Relational Hybrid

> Project: Job Architecture & Skills Taxonomy · Created: 2026-05-12

## Philosophy

This model uses PostgreSQL as the operational store for CRUD operations but embeds a property-graph layer within it to model the skills ontology, career pathways, and employee-to-skill relationships as a traversable graph. The core insight is that a skills taxonomy is fundamentally a graph problem: skills have parent-child relationships, lateral adjacencies ("Scrum" is related to "Kanban"), prerequisite chains, and contextual modifiers. Job profiles connect to skills, employees connect to skills, and the most valuable queries -- "find employees with skills similar to this role", "what career path gets me from here to there?", "what hidden skills might this employee have?" -- are all graph traversal problems.

This approach is directly inspired by NASA's People Knowledge Graph, where David Meza's team used Neo4j to discover hidden employee skills by modelling O\*NET occupations, skills, and employees as graph nodes and using Doc2Vec similarity algorithms to infer skills from job descriptions and performance evaluations. It is also informed by Gloat's Multi-Ontology Workforce Graph and Beamery's Talent Graph, both of which use graph structures to power skills matching and internal mobility.

The hybrid design avoids the operational complexity of running a separate graph database (Neo4j, Memgraph) by implementing the graph layer using PostgreSQL tables (`graph_node`, `graph_edge`) with the `ltree` extension for hierarchical path queries. For organisations that outgrow this approach, the graph layer can be migrated to a dedicated graph database without changing the relational operational tables.

**Best for:** Organisations that need skills similarity queries, hidden-skill inference, career path traversal, and "find me someone with skills like X" matching -- the core AI-native use cases.

**Trade-offs:**
- Pro: Natural representation of skills ontology relationships (is-a, related-to, prerequisite, adjacent)
- Pro: Efficient career path traversal without deep recursive CTEs
- Pro: Enables skills similarity scoring and hidden-skill inference (the NASA pattern)
- Pro: Multi-taxonomy support -- O\*NET, ESCO, Lightcast, and SFIA nodes coexist in the same graph with crosswalk edges
- Pro: Can migrate graph layer to Neo4j/Memgraph later without changing CRUD tables
- Con: More complex query patterns; requires familiarity with graph traversal concepts
- Con: PostgreSQL graph queries are slower than native graph databases for deep traversals (>5 hops)
- Con: The `ltree` approach has limitations for non-hierarchical relationships; edge table queries add complexity
- Con: Schema is less intuitive for teams without graph database experience

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| O\*NET Content Model | O\*NET occupations, skills, abilities, and tasks are all graph nodes; the six-domain hierarchy is expressed as `IS_A` and `BELONGS_TO` edges |
| ESCO v1.2 | ESCO concepts imported as graph nodes with their URI as the external identifier; ESCO's own broader/narrower relationships map directly to graph edges |
| Lightcast Open Skills | Lightcast skills are graph nodes; category/subcategory hierarchy expressed as `BELONGS_TO` edges; related-skill mappings from the Lightcast API become `RELATED_TO` edges |
| SFIA 9 | SFIA skills and responsibility levels modelled as graph nodes; enables graph queries like "find all SFIA skills related to this Lightcast skill" |
| IEEE 1484.20.1 | Competency definitions stored as node properties; the "is-a" and "part-of" relationships in IEEE's model map to graph edge types |
| Schema.org | Graph nodes for occupations include Schema.org-compatible properties for export |
| NASA People Knowledge Graph | Architecture directly inspired by NASA's approach: employees, skills, and occupations as graph nodes with similarity-scored edges |

---

## Graph Layer Tables

### Nodes and Edges

```sql
-- Enable ltree extension for hierarchical path queries
CREATE EXTENSION IF NOT EXISTS ltree;
CREATE EXTENSION IF NOT EXISTS pg_trgm;

-- Graph nodes: every concept in the system is a node
CREATE TABLE graph_node (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    node_type       TEXT NOT NULL,
    -- Node types:
    --   'skill'           - a skill/competence
    --   'skill_category'  - top-level skill category
    --   'skill_subcategory' - second-level grouping
    --   'occupation'      - O*NET/ESCO occupation
    --   'job_family'      - organisational job family
    --   'job_profile'     - a specific role definition
    --   'job_level'       - a grade/level in the hierarchy
    --   'employee'        - a person
    --   'department'      - organisational unit
    --   'learning_resource' - training/course
    name            TEXT NOT NULL,
    description     TEXT,
    path            LTREE,                   -- hierarchical path for tree queries
    -- e.g. 'skills.specialised.software_engineering.python'
    external_id     TEXT,                    -- ID from external taxonomy (Lightcast, O*NET SOC, ESCO URI)
    source          TEXT,                    -- 'lightcast', 'onet', 'esco', 'sfia', 'custom'
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by node_type. Examples:
    -- For 'skill':      {"skill_type": "specialised", "is_emerging": false}
    -- For 'occupation': {"soc_code": "15-1243.00", "job_zone": 4}
    -- For 'job_profile': {"track": "individual_contributor", "level_number": 5}
    -- For 'employee':   {"email": "...", "hire_date": "2024-01-15", "department": "Engineering"}
    is_active       BOOLEAN NOT NULL DEFAULT TRUE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gn_tenant ON graph_node(tenant_id);
CREATE INDEX idx_gn_type ON graph_node(tenant_id, node_type);
CREATE INDEX idx_gn_path ON graph_node USING gist (path);
CREATE INDEX idx_gn_external ON graph_node(source, external_id);
CREATE INDEX idx_gn_name_trgm ON graph_node USING gin (name gin_trgm_ops);
CREATE INDEX idx_gn_properties ON graph_node USING gin (properties);

-- Graph edges: relationships between nodes
CREATE TABLE graph_edge (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    source_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    target_id       UUID NOT NULL REFERENCES graph_node(id) ON DELETE CASCADE,
    edge_type       TEXT NOT NULL,
    -- Edge types:
    --   'IS_A'            - taxonomic hierarchy (Python IS_A Programming Language)
    --   'BELONGS_TO'      - category membership (skill belongs to subcategory)
    --   'RELATED_TO'      - lateral skill relationship (Scrum RELATED_TO Kanban)
    --   'PREREQUISITE'    - skill prerequisite chain
    --   'ADJACENT_TO'     - adjacent skill (learnable from current skill)
    --   'REQUIRES'        - job profile requires skill
    --   'HAS_SKILL'       - employee has assessed skill
    --   'MAPS_TO'         - crosswalk mapping (Lightcast skill MAPS_TO O*NET element)
    --   'CAREER_PATH'     - progression from one role to another
    --   'REPORTS_TO'      - organisational hierarchy
    --   'SIMILAR_TO'      - AI-inferred similarity
    properties      JSONB NOT NULL DEFAULT '{}',
    -- Properties vary by edge_type. Examples:
    -- For 'REQUIRES':    {"proficiency_level": 4, "importance": "required"}
    -- For 'HAS_SKILL':   {"proficiency_level": 3, "assessment_type": "manager", "assessed_at": "2026-03-15"}
    -- For 'MAPS_TO':     {"match_type": "exact", "confidence": 0.95}
    -- For 'CAREER_PATH': {"pathway_type": "promotion", "typical_months": 24}
    -- For 'SIMILAR_TO':  {"similarity_score": 0.87, "algorithm": "doc2vec"}
    weight          NUMERIC(5,4) DEFAULT 1.0,  -- edge weight for graph algorithms
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_ge_tenant ON graph_edge(tenant_id);
CREATE INDEX idx_ge_source ON graph_edge(source_id);
CREATE INDEX idx_ge_target ON graph_edge(target_id);
CREATE INDEX idx_ge_type ON graph_edge(edge_type);
CREATE INDEX idx_ge_source_type ON graph_edge(source_id, edge_type);
CREATE INDEX idx_ge_target_type ON graph_edge(target_id, edge_type);
CREATE INDEX idx_ge_properties ON graph_edge USING gin (properties);
```

---

## Operational CRUD Tables

These tables support the transactional operations that don't benefit from graph structure.

```sql
-- Tenant and authentication (same as Model 1)
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

-- Proficiency scales (configurable per tenant)
CREATE TABLE proficiency_scale (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    is_default      BOOLEAN NOT NULL DEFAULT FALSE,
    levels          JSONB NOT NULL,
    -- Example: [
    --   {"number": 1, "name": "Awareness", "description": "Basic familiarity"},
    --   {"number": 2, "name": "Beginner", "description": "Can perform with guidance"},
    --   {"number": 3, "name": "Intermediate", "description": "Can perform independently"},
    --   {"number": 4, "name": "Advanced", "description": "Can teach others"},
    --   {"number": 5, "name": "Expert", "description": "Industry-recognised authority"}
    -- ]
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Pay bands (linked to job_level graph nodes)
CREATE TABLE pay_band (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    job_level_node_id UUID NOT NULL REFERENCES graph_node(id),
    currency        CHAR(3) NOT NULL DEFAULT 'USD',
    min_salary      NUMERIC(12,2),
    mid_salary      NUMERIC(12,2),
    max_salary      NUMERIC(12,2),
    effective_from  DATE NOT NULL,
    effective_to    DATE,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- HRIS integration connections
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

-- Audit log for graph mutations
CREATE TABLE graph_audit_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    user_id         UUID,
    action          TEXT NOT NULL,           -- 'node_created', 'node_updated', 'edge_created', 'edge_deleted'
    node_id         UUID,
    edge_id         UUID,
    old_data        JSONB,
    new_data        JSONB,
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_gal_tenant ON graph_audit_log(tenant_id);
CREATE INDEX idx_gal_node ON graph_audit_log(node_id);
CREATE INDEX idx_gal_created ON graph_audit_log(created_at);

-- Access control
CREATE TABLE app_role (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    name            TEXT NOT NULL,
    permissions     TEXT[] NOT NULL DEFAULT '{}',
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE app_user (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL REFERENCES tenant(id),
    email           TEXT NOT NULL,
    role_id         UUID NOT NULL REFERENCES app_role(id),
    employee_node_id UUID REFERENCES graph_node(id),
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);
```

---

## Graph Query Examples

### Find all skills related to a given skill (1-2 hops)

```sql
-- Direct relationships
SELECT
    target.name AS related_skill,
    e.edge_type,
    (e.properties->>'similarity_score')::NUMERIC AS similarity,
    e.weight
FROM graph_edge e
JOIN graph_node target ON target.id = e.target_id
WHERE e.source_id = :skill_node_id
  AND e.edge_type IN ('RELATED_TO', 'ADJACENT_TO', 'SIMILAR_TO')
  AND target.is_active = TRUE
ORDER BY e.weight DESC;

-- Two-hop traversal: skills related to skills related to X
WITH hop1 AS (
    SELECT e.target_id, e.weight AS w1
    FROM graph_edge e
    WHERE e.source_id = :skill_node_id
      AND e.edge_type IN ('RELATED_TO', 'ADJACENT_TO')
),
hop2 AS (
    SELECT e.target_id, h.w1 * e.weight AS combined_weight
    FROM graph_edge e
    JOIN hop1 h ON e.source_id = h.target_id
    WHERE e.edge_type IN ('RELATED_TO', 'ADJACENT_TO')
      AND e.target_id != :skill_node_id  -- exclude self
)
SELECT DISTINCT ON (n.id)
    n.name,
    MAX(hop2.combined_weight) AS relevance
FROM hop2
JOIN graph_node n ON n.id = hop2.target_id
WHERE n.is_active = TRUE
GROUP BY n.id, n.name
ORDER BY n.id, relevance DESC
LIMIT 20;
```

### Career path traversal (shortest path between two roles)

```sql
-- Find all career paths from role A to role B (up to 5 hops)
WITH RECURSIVE career_paths AS (
    -- Start from source role
    SELECT
        ARRAY[e.source_id] AS path_nodes,
        ARRAY[e.target_id] AS next_nodes,
        e.target_id AS current_node,
        1 AS hops,
        (e.properties->>'typical_months')::INT AS total_months
    FROM graph_edge e
    WHERE e.source_id = :from_role_node_id
      AND e.edge_type = 'CAREER_PATH'
    
    UNION ALL
    
    SELECT
        cp.path_nodes || cp.current_node,
        cp.next_nodes || e.target_id,
        e.target_id,
        cp.hops + 1,
        cp.total_months + COALESCE((e.properties->>'typical_months')::INT, 0)
    FROM career_paths cp
    JOIN graph_edge e ON e.source_id = cp.current_node
    WHERE e.edge_type = 'CAREER_PATH'
      AND cp.hops < 5
      AND e.target_id != ALL(cp.path_nodes)  -- prevent cycles
)
SELECT
    hops,
    total_months,
    (SELECT array_agg(n.name ORDER BY ordinality)
     FROM unnest(path_nodes || current_node) WITH ORDINALITY AS u(node_id, ordinality)
     JOIN graph_node n ON n.id = u.node_id) AS path_names
FROM career_paths
WHERE current_node = :to_role_node_id
ORDER BY hops, total_months;
```

### Skills gap analysis using graph

```sql
-- For a given employee, find skills gaps against their assigned role
SELECT
    skill_node.name AS skill_name,
    (req_edge.properties->>'proficiency_level')::INT AS required_level,
    (req_edge.properties->>'importance') AS importance,
    COALESCE((has_edge.properties->>'proficiency_level')::INT, 0) AS actual_level,
    (req_edge.properties->>'proficiency_level')::INT -
        COALESCE((has_edge.properties->>'proficiency_level')::INT, 0) AS gap
FROM graph_edge req_edge
JOIN graph_node skill_node ON skill_node.id = req_edge.target_id
LEFT JOIN graph_edge has_edge ON has_edge.source_id = :employee_node_id
    AND has_edge.target_id = skill_node.id
    AND has_edge.edge_type = 'HAS_SKILL'
WHERE req_edge.source_id = :job_profile_node_id
  AND req_edge.edge_type = 'REQUIRES'
ORDER BY gap DESC;
```

### Find employees with similar skill profiles (NASA pattern)

```sql
-- Cosine-like similarity between two employees based on shared skills
WITH employee_skills AS (
    SELECT
        e.source_id AS employee_id,
        e.target_id AS skill_id,
        (e.properties->>'proficiency_level')::INT AS proficiency
    FROM graph_edge e
    WHERE e.edge_type = 'HAS_SKILL'
      AND e.source_id IN (:employee_a_id, :employee_b_id)
),
skill_vectors AS (
    SELECT
        skill_id,
        MAX(CASE WHEN employee_id = :employee_a_id THEN proficiency ELSE 0 END) AS a_prof,
        MAX(CASE WHEN employee_id = :employee_b_id THEN proficiency ELSE 0 END) AS b_prof
    FROM employee_skills
    GROUP BY skill_id
)
SELECT
    SUM(a_prof * b_prof)::NUMERIC /
    (SQRT(SUM(a_prof * a_prof)) * SQRT(SUM(b_prof * b_prof))) AS cosine_similarity
FROM skill_vectors;
```

### Hierarchy queries using ltree

```sql
-- Find all skills under "Software Engineering" using ltree path
SELECT name, path, properties
FROM graph_node
WHERE path <@ 'skills.specialised.software_engineering'
  AND node_type = 'skill'
  AND tenant_id = :tenant_id
ORDER BY path;

-- Find the full ancestry of a skill
SELECT name, path
FROM graph_node
WHERE 'skills.specialised.software_engineering.python' <@ path
  AND tenant_id = :tenant_id
ORDER BY nlevel(path);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Graph Layer (nodes, edges) | 2 | All taxonomy, job architecture, and employee data as graph nodes/edges |
| Operational (tenants, pay bands, HRIS) | 4 | Non-graph transactional tables |
| Proficiency Scales | 1 | Configurable scales with JSONB levels |
| Audit & History | 1 | Graph mutation audit log |
| Access Control | 2 | RBAC roles and users |
| **Total** | **~10 tables** | Dramatically fewer tables than normalized model; complexity is in graph query patterns |

---

## Key Design Decisions

1. **Two-table graph layer (`graph_node` + `graph_edge`)** — all concepts (skills, occupations, job profiles, employees) and relationships (requires, has-skill, related-to, career-path) are stored in just two tables, enabling any-to-any relationship modelling without schema changes. New entity types or relationship types are added by convention (new `node_type` or `edge_type` values), not by DDL.

2. **JSONB `properties` on both nodes and edges** — variable attributes are stored as JSONB rather than as columns. A skill node's `properties` contains `{"skill_type": "specialised"}` while an employee node's contains `{"email": "...", "hire_date": "..."}`. This eliminates the need for separate tables per entity type while still allowing GIN-indexed queries.

3. **`ltree` paths for hierarchical queries** — the `path` column on `graph_node` stores the full taxonomy path (e.g., `skills.specialised.software_engineering.python`), enabling efficient ancestor/descendant queries using PostgreSQL's `ltree` operators (`<@`, `@>`) without recursive CTEs.

4. **Edge weights for graph algorithms** — every edge has a `weight` column that can be used for shortest-path, PageRank, and similarity algorithms. Career path edges use weight to represent transition probability; skill relationship edges use weight to represent relatedness strength.

5. **Multi-taxonomy coexistence** — O\*NET occupations, ESCO concepts, Lightcast skills, and SFIA competencies all exist as nodes in the same graph. `MAPS_TO` edges with confidence scores connect them, enabling cross-taxonomy traversal queries ("find all ESCO skills related to this Lightcast skill").

6. **NASA-inspired similarity edges** — `SIMILAR_TO` edges with similarity scores can be computed by AI models (Doc2Vec, embeddings) and stored in the graph, enabling "find employees similar to this role" queries without re-computing similarity at query time.

7. **Graph audit log for compliance** — all graph mutations are recorded in `graph_audit_log` with before/after JSONB snapshots, providing an audit trail for regulatory requirements (ISO 30414) without the full complexity of event sourcing.

8. **Migration path to Neo4j** — if PostgreSQL graph performance becomes a bottleneck, the `graph_node`/`graph_edge` tables can be migrated to Neo4j using a straightforward ETL (nodes become labeled nodes, edges become typed relationships, JSONB properties become Neo4j properties). The operational CRUD tables remain in PostgreSQL.

9. **Proficiency scales as JSONB** — since proficiency levels are small, bounded lists that rarely change, they are stored as a JSONB array within the `proficiency_scale` table rather than as a separate table, reducing join complexity.

10. **Bidirectional edge queries** — indexes on both `source_id` and `target_id` ensure that graph traversal is efficient in both directions, which is critical for queries like "what roles require this skill?" (target-to-source) and "what skills does this role require?" (source-to-target).
