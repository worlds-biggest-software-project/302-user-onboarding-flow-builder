# Data Model Suggestion 1: Entity-Centric Normalized Relational

> Project: User Onboarding Flow Builder · Created: 2026-05-22

## Philosophy

Every concept has its own table with tight foreign-key relationships. The flow hierarchy (Flow → Step Group → Step → Action) is represented as separate, joinable entities. Segmentation rules are stored as a recursive condition-tree table rather than raw JSONB, enabling SQL-level querying of rule structure. The Appcues v2 API schema directly informs the resource model: flows, steps, segments, tags, and events map one-to-one to tables.

This approach mirrors how Appcues, Chameleon, and Pendo model their internal data — a canonical relational core with strict referential integrity, suited to teams that want predictable query plans and straightforward analytics joins.

**Best for:** Teams prioritising data integrity, complex cross-entity analytics joins, or operating in compliance environments that require auditable schema structure.

**Trade-offs:**
- High table count — schema is verbose but explicit
- JOIN-heavy queries; mitigate with composite indexes and read replicas
- Segmentation rule queries require recursive CTEs; complex but fast with proper indexing
- Flow versioning adds two extra tables (flow_versions, published_snapshots) but keeps live flows stable
- Easy to extend with new step types without touching existing tables

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| WCAG 2.1 / WAI-ARIA | `steps.aria_role`, `steps.aria_label` fields on step content; a11y metadata stored per step |
| GDPR / CCPA | `tenants.data_residency_region`, `user_journey_events.consent_flags` JSONB; retention TTL per tenant |
| OpenTelemetry | `user_journey_events.otel_trace_id` links inbound OTel spans to journey records |
| OpenAPI 3.1 | Flow, Segment, Tag resources mirror Appcues v2 API fields exactly |
| JSON Schema 2020-12 | Trigger conditions validated against stored JSON Schema before persistence |
| OAuth 2.0 | `api_keys.scopes`, `oauth_clients` table for third-party integrations |
| PostgreSQL RLS | `SET LOCAL app.current_tenant_id = ?` with `USING (tenant_id = current_setting('app.current_tenant_id')::uuid)` on every table |

---

## Multi-Tenant Foundation

```sql
-- Tenant (organization) root
CREATE TABLE tenants (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug                  TEXT NOT NULL UNIQUE,          -- used in subdomain routing
    name                  TEXT NOT NULL,
    plan                  TEXT NOT NULL DEFAULT 'starter', -- starter | growth | enterprise
    data_residency_region TEXT NOT NULL DEFAULT 'us-east-1',
    mau_limit             INTEGER NOT NULL DEFAULT 1000,
    created_at            TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at            TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Projects within a tenant (product/app being instrumented)
CREATE TABLE projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    slug        TEXT NOT NULL,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);

-- Environments: dev / staging / production per project
CREATE TABLE environments (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    project_id   UUID NOT NULL REFERENCES projects(id) ON DELETE CASCADE,
    tenant_id    UUID NOT NULL REFERENCES tenants(id),   -- denormalized for RLS
    name         TEXT NOT NULL,                          -- 'development' | 'staging' | 'production'
    sdk_key      TEXT NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(24), 'base64'),
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_environments_tenant ON environments(tenant_id);

-- API keys (server-side REST API access)
CREATE TABLE api_keys (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name         TEXT NOT NULL,
    key_hash     TEXT NOT NULL UNIQUE,   -- bcrypt hash; raw key shown once at creation
    scopes       TEXT[] NOT NULL DEFAULT '{}',  -- ['flows:read','flows:write','analytics:read']
    expires_at   TIMESTAMPTZ,
    last_used_at TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_api_keys_tenant ON api_keys(tenant_id);

-- Users/members of the platform (product team members, not end-users)
CREATE TABLE members (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email       TEXT NOT NULL,
    name        TEXT NOT NULL,
    role        TEXT NOT NULL DEFAULT 'editor',  -- owner | admin | editor | viewer
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

-- Row Level Security on all tenant-scoped tables
ALTER TABLE projects     ENABLE ROW LEVEL SECURITY;
ALTER TABLE environments ENABLE ROW LEVEL SECURITY;
ALTER TABLE api_keys     ENABLE ROW LEVEL SECURITY;
ALTER TABLE members      ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON projects
    USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
-- (same policy pattern applied to all tenant-scoped tables)
```

---

## Flow Authoring

```sql
-- Flow definition (the master record; never directly executed)
CREATE TABLE flows (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id      UUID NOT NULL REFERENCES tenants(id),
    project_id     UUID NOT NULL REFERENCES projects(id),
    name           TEXT NOT NULL,
    description    TEXT,
    flow_type      TEXT NOT NULL DEFAULT 'tour',  -- tour | checklist | tooltip | banner | survey | announcement
    status         TEXT NOT NULL DEFAULT 'draft', -- draft | published | archived
    frequency      TEXT NOT NULL DEFAULT 'once',  -- once | every_session | every_time
    published_at   TIMESTAMPTZ,
    published_by   UUID REFERENCES members(id),
    created_by     UUID NOT NULL REFERENCES members(id),
    updated_by     UUID REFERENCES members(id),
    created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_flows_tenant_project ON flows(tenant_id, project_id);
CREATE INDEX idx_flows_status ON flows(tenant_id, status);

-- Published snapshot (immutable copy of flow definition at publish time)
-- Live flows execute from this table; editing the draft never changes what users see
CREATE TABLE flow_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id         UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    version_number  INTEGER NOT NULL,
    definition      JSONB NOT NULL,   -- full denormalized snapshot of steps + triggers + settings
    change_summary  TEXT,
    published_by    UUID REFERENCES members(id),
    published_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_live         BOOLEAN NOT NULL DEFAULT false,  -- only one live per flow
    UNIQUE (flow_id, version_number)
);
CREATE UNIQUE INDEX idx_flow_versions_live ON flow_versions(flow_id) WHERE is_live = true;
CREATE INDEX idx_flow_versions_tenant ON flow_versions(tenant_id);

-- Tags for organizing flows
CREATE TABLE tags (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    name        TEXT NOT NULL,
    color       TEXT,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE flow_tags (
    flow_id  UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tag_id   UUID NOT NULL REFERENCES tags(id) ON DELETE CASCADE,
    PRIMARY KEY (flow_id, tag_id)
);
```

---

## Hierarchical Step Content

```sql
-- Step groups (e.g., pages in a multi-page tour, sections in a checklist)
CREATE TABLE step_groups (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id     UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    name        TEXT,
    position    SMALLINT NOT NULL DEFAULT 0,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_step_groups_flow ON step_groups(flow_id, position);

-- Individual steps (tooltips, modals, hotspots, checklist items)
CREATE TABLE steps (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_group_id   UUID NOT NULL REFERENCES step_groups(id) ON DELETE CASCADE,
    flow_id         UUID NOT NULL REFERENCES flows(id),       -- denormalized for fast lookups
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    parent_step_id  UUID REFERENCES steps(id),               -- for branching sub-steps (adjacency list)
    name            TEXT NOT NULL,
    step_type       TEXT NOT NULL,  -- tooltip | modal | hotspot | checklist_item | banner | survey_question
    position        SMALLINT NOT NULL DEFAULT 0,
    -- Element targeting
    selector_primary   TEXT,        -- CSS selector (preferred)
    selector_fallback  TEXT,        -- secondary CSS selector
    selector_xpath     TEXT,        -- XPath for resilience
    selector_data_attr TEXT,        -- data-testid / data-appcues-id value
    selector_text      TEXT,        -- visible text match (last resort)
    selector_strategy  TEXT NOT NULL DEFAULT 'css',  -- css | xpath | text | data_attr | compound
    -- Content
    content_html    TEXT,
    content_blocks  JSONB,          -- rich block-based content (title, body, media, buttons)
    -- Accessibility
    aria_role       TEXT,
    aria_label      TEXT,
    -- Positioning
    placement       TEXT,           -- top | bottom | left | right | center | auto
    offset_x        INTEGER DEFAULT 0,
    offset_y        INTEGER DEFAULT 0,
    -- Progression
    skippable       BOOLEAN NOT NULL DEFAULT true,
    required_action TEXT,           -- 'click_element' | 'fill_form' | null (free progress)
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_steps_flow ON steps(flow_id, position);
CREATE INDEX idx_steps_group ON steps(step_group_id, position);
-- Recursive CTE for branching: WITH RECURSIVE subtree AS (SELECT * FROM steps WHERE id = $1 UNION ALL SELECT s.* FROM steps s JOIN subtree t ON s.parent_step_id = t.id) SELECT * FROM subtree;

-- Actions triggered from a step (button clicks, skip, complete, navigate)
CREATE TABLE step_actions (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    step_id      UUID NOT NULL REFERENCES steps(id) ON DELETE CASCADE,
    tenant_id    UUID NOT NULL REFERENCES tenants(id),
    trigger_type TEXT NOT NULL,  -- 'button_click' | 'element_click' | 'timer' | 'scroll' | 'skip'
    action_type  TEXT NOT NULL,  -- 'next_step' | 'prev_step' | 'go_to_step' | 'complete_flow' | 'dismiss' | 'open_url' | 'fire_event' | 'branch'
    action_data  JSONB,          -- { "step_id": "...", "url": "...", "event_name": "..." }
    label        TEXT,           -- button label
    position     SMALLINT DEFAULT 0,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## Targeting & Triggering

```sql
-- Trigger rules: when a flow should appear
CREATE TABLE flow_triggers (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id      UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id    UUID NOT NULL REFERENCES tenants(id),
    trigger_type TEXT NOT NULL,  -- 'page_visit' | 'event' | 'element_click' | 'api_call' | 'schedule' | 'manual'
    config       JSONB NOT NULL, -- { "url_pattern": "/dashboard*", "event_name": "feature_clicked", "delay_ms": 1000 }
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Segments (named audience definitions)
CREATE TABLE segments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    project_id  UUID NOT NULL REFERENCES projects(id),
    name        TEXT NOT NULL,
    description TEXT,
    created_by  UUID REFERENCES members(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Condition tree nodes (recursive; root node is top-level AND/OR)
-- Each row is one node in the condition tree
CREATE TABLE segment_conditions (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    segment_id   UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    parent_id    UUID REFERENCES segment_conditions(id),  -- null = root node
    tenant_id    UUID NOT NULL REFERENCES tenants(id),
    node_type    TEXT NOT NULL,  -- 'group' (AND/OR container) | 'condition' (leaf)
    bool_op      TEXT,           -- 'AND' | 'OR' | 'NOT' (for group nodes)
    -- For leaf conditions:
    property_ref TEXT,           -- 'user.plan' | 'user.company.size' | 'event.count' | 'account.mrr'
    operator     TEXT,           -- 'eq' | 'neq' | 'lt' | 'lte' | 'gt' | 'gte' | 'contains' | 'not_contains' | 'is_set' | 'is_not_set' | 'in' | 'not_in' | 'matches_regex'
    value        JSONB,          -- scalar, array, or null depending on operator
    position     SMALLINT DEFAULT 0,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_segment_conditions_segment ON segment_conditions(segment_id);
CREATE INDEX idx_segment_conditions_parent ON segment_conditions(parent_id);

-- Example condition tree for (user.plan = 'pro' OR user.plan = 'enterprise') AND event.count >= 3:
-- Root: {node_type:'group', bool_op:'AND', parent_id: null}
--   Child 1: {node_type:'group', bool_op:'OR', parent_id: root}
--     Leaf: {node_type:'condition', property_ref:'user.plan', operator:'eq', value:'"pro"'}
--     Leaf: {node_type:'condition', property_ref:'user.plan', operator:'eq', value:'"enterprise"'}
--   Child 2: {node_type:'condition', property_ref:'event.count', operator:'gte', value:'3'}

-- Link flows to target segments
CREATE TABLE flow_segments (
    flow_id     UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    segment_id  UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    PRIMARY KEY (flow_id, segment_id)
);
```

---

## Experiments (A/B Testing)

```sql
-- Experiment definition
CREATE TABLE experiments (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id         UUID NOT NULL REFERENCES tenants(id),
    project_id        UUID NOT NULL REFERENCES projects(id),
    name              TEXT NOT NULL,
    hypothesis        TEXT,
    status            TEXT NOT NULL DEFAULT 'draft',  -- draft | running | paused | concluded
    stat_method       TEXT NOT NULL DEFAULT 'frequentist',  -- frequentist | bayesian
    confidence_level  NUMERIC(4,3) NOT NULL DEFAULT 0.95,  -- 0.95 = 95%
    min_sample_size   INTEGER,                  -- per variant; null = auto-calculate
    primary_metric    TEXT NOT NULL,             -- 'flow_completion_rate' | 'step_N_completion' | 'custom_event'
    primary_metric_config JSONB,                -- { "event_name": "upgraded", "window_days": 14 }
    started_at        TIMESTAMPTZ,
    concluded_at      TIMESTAMPTZ,
    winner_variant_id UUID,                      -- set when concluded
    created_by        UUID REFERENCES members(id),
    created_at        TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at        TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_experiments_tenant ON experiments(tenant_id, status);

-- Variants (control + treatments)
CREATE TABLE experiment_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    flow_id         UUID NOT NULL REFERENCES flows(id),      -- which flow version this variant runs
    name            TEXT NOT NULL,                           -- 'control' | 'variant_a' | 'variant_b'
    is_control      BOOLEAN NOT NULL DEFAULT false,
    traffic_weight  NUMERIC(5,4) NOT NULL DEFAULT 0.5,       -- 0.0–1.0; weights across variants must sum to 1.0
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_variants_experiment ON experiment_variants(experiment_id);

-- Per-user variant assignment (sticky; deterministic after first assignment)
CREATE TABLE experiment_assignments (
    id             UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id  UUID NOT NULL REFERENCES experiments(id),
    variant_id     UUID NOT NULL REFERENCES experiment_variants(id),
    tenant_id      UUID NOT NULL REFERENCES tenants(id),
    end_user_id    TEXT NOT NULL,    -- opaque identifier from host product
    assigned_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, end_user_id)
);
CREATE INDEX idx_assignments_experiment ON experiment_assignments(experiment_id, variant_id);

-- Aggregate results snapshot (recomputed daily by analytics job)
CREATE TABLE experiment_results (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id),
    variant_id      UUID NOT NULL REFERENCES experiment_variants(id),
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    participants    INTEGER NOT NULL DEFAULT 0,
    conversions     INTEGER NOT NULL DEFAULT 0,
    conversion_rate NUMERIC(8,6),
    p_value         NUMERIC(8,6),                      -- frequentist
    probability_best NUMERIC(8,6),                     -- bayesian
    credible_interval_low  NUMERIC(8,6),               -- bayesian
    credible_interval_high NUMERIC(8,6),               -- bayesian
    computed_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (experiment_id, variant_id, computed_at)
);
```

---

## User Journey Event Stream

```sql
-- End-users observed by the platform (from the host product's user base)
CREATE TABLE end_users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL REFERENCES tenants(id),
    project_id   UUID NOT NULL REFERENCES projects(id),
    external_id  TEXT NOT NULL,             -- user ID from host product
    traits       JSONB NOT NULL DEFAULT '{}', -- { "plan": "pro", "company_size": 150, "role": "admin" }
    group_id     TEXT,                       -- company/account identifier
    group_traits JSONB NOT NULL DEFAULT '{}',
    first_seen   TIMESTAMPTZ,
    last_seen    TIMESTAMPTZ,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, project_id, external_id)
);
CREATE INDEX idx_end_users_tenant_project ON end_users(tenant_id, project_id);

-- Immutable event log for all user journey events
CREATE TABLE user_journey_events (
    id             BIGSERIAL PRIMARY KEY,     -- sequential for ordering; no UUID (high-write table)
    event_id       UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,  -- stable external reference
    tenant_id      UUID NOT NULL REFERENCES tenants(id),
    project_id     UUID NOT NULL REFERENCES projects(id),
    environment_id UUID NOT NULL REFERENCES environments(id),
    end_user_id    UUID NOT NULL REFERENCES end_users(id),
    flow_id        UUID REFERENCES flows(id),
    flow_version_id UUID REFERENCES flow_versions(id),
    step_id        UUID REFERENCES steps(id),
    experiment_id  UUID REFERENCES experiments(id),
    variant_id     UUID REFERENCES experiment_variants(id),
    event_type     TEXT NOT NULL,  -- 'flow_started' | 'step_viewed' | 'step_completed' | 'step_skipped' | 'flow_completed' | 'flow_dismissed' | 'checklist_item_completed' | 'experiment_assigned'
    event_payload  JSONB NOT NULL DEFAULT '{}',
    url            TEXT,
    otel_trace_id  TEXT,           -- OpenTelemetry trace linkage
    consent_flags  JSONB,          -- GDPR/CCPA consent state at time of event
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now()
) PARTITION BY RANGE (occurred_at);  -- partition by month for query performance

-- Monthly partitions (created by maintenance job)
CREATE TABLE user_journey_events_2026_05 PARTITION OF user_journey_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

CREATE INDEX idx_journey_events_tenant_user ON user_journey_events(tenant_id, end_user_id, occurred_at DESC);
CREATE INDEX idx_journey_events_flow ON user_journey_events(flow_id, event_type, occurred_at DESC);

-- Materialized completion state (derived from event stream; refreshed asynchronously)
CREATE TABLE user_flow_states (
    id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id            UUID NOT NULL REFERENCES tenants(id),
    end_user_id          UUID NOT NULL REFERENCES end_users(id),
    flow_id              UUID NOT NULL REFERENCES flows(id),
    flow_version_id      UUID REFERENCES flow_versions(id),
    state                TEXT NOT NULL DEFAULT 'not_started',  -- not_started | in_progress | completed | dismissed | skipped
    steps_completed      INTEGER NOT NULL DEFAULT 0,
    steps_total          INTEGER NOT NULL DEFAULT 0,
    started_at           TIMESTAMPTZ,
    completed_at         TIMESTAMPTZ,
    dismissed_at         TIMESTAMPTZ,
    last_seen_step_id    UUID REFERENCES steps(id),
    last_event_at        TIMESTAMPTZ,
    UNIQUE (tenant_id, end_user_id, flow_id)
);
CREATE INDEX idx_flow_states_tenant_flow ON user_flow_states(tenant_id, flow_id, state);
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenant foundation | 6 | tenants, projects, environments, api_keys, members, RLS policies |
| Flow authoring | 4 | flows, flow_versions, tags, flow_tags |
| Hierarchical content | 3 | step_groups, steps, step_actions |
| Targeting & triggering | 4 | flow_triggers, segments, segment_conditions, flow_segments |
| Experiments | 4 | experiments, experiment_variants, experiment_assignments, experiment_results |
| User journey | 3 | end_users, user_journey_events (partitioned), user_flow_states |
| **Total** | **24** | |

---

## Key Design Decisions

1. **Snapshot-on-publish versioning**: `flow_versions.definition` stores a full denormalized JSONB snapshot of all steps, groups, and actions at publish time. Live execution reads from `flow_versions` (immutable), not from `steps`/`step_groups` (mutable drafts). Only one `flow_version` per flow has `is_live = true`.

2. **Recursive condition tree table**: `segment_conditions` uses an adjacency list (`parent_id` self-reference) to model arbitrary-depth AND/OR trees. Leaf nodes carry `property_ref`, `operator`, `value`; group nodes carry `bool_op`. Evaluated recursively in application code or via recursive CTE — avoids JSONB opacity while keeping the structure queryable.

3. **Compound selector storage**: `steps` carries five selector fields (`selector_primary`, `selector_fallback`, `selector_xpath`, `selector_data_attr`, `selector_text`) plus a `selector_strategy` enum. The SDK tries strategies in priority order; `data-testid`/data attribute selectors are most resilient to DOM refactors.

4. **Partitioned event log**: `user_journey_events` is range-partitioned by month. Events are append-only (no UPDATE/DELETE); completion state is materialized into `user_flow_states` by an async background worker after each event batch.

5. **Sticky experiment assignment**: `experiment_assignments` has a UNIQUE constraint on `(experiment_id, end_user_id)`. Traffic weight is enforced in application code (hash-based deterministic bucketing) before inserting; users never switch variants mid-experiment.

6. **RLS via `current_setting`**: Every tenant-scoped table has `ENABLE ROW LEVEL SECURITY` with a policy using `current_setting('app.current_tenant_id')::uuid`. The app layer sets `SET LOCAL app.current_tenant_id = '...'` per request inside a transaction; the setting expires with the transaction to prevent cross-tenant leakage.

7. **`members` vs `end_users`**: `members` are product team members who use the builder UI. `end_users` are the end-users of the instrumented product who experience flows. These two user populations have entirely separate tables and identity systems.
