# Data Model Suggestion 3: Hybrid Relational + JSONB

> Project: User Onboarding Flow Builder · Created: 2026-05-22

## Philosophy

A lean relational spine handles identity, routing, and the most frequently queried fields. Everything else — step content, trigger conditions, segmentation rules, flow structure, experiment configuration — lives in validated JSONB columns. This is the architecture pattern used by Stripe (their API objects are partially JSONB-backed), Segment (track/identify calls stored as JSONB events), and most PLG-era SaaS tools that need to ship fast and iterate schema without migrations.

The flow definition itself is stored as a single JSONB document, making it easy to version (just snapshot the document), easy to edit (patch the JSONB), and easy to deliver to the SDK (serialize the column directly). Segmentation rules use JsonLogic's portable JSON format — the same rule object can be evaluated in the browser SDK, Node.js backend, and Python analytics pipeline without reimplementation.

**Best for:** MVP-velocity teams, multi-jurisdiction deployments where step content fields vary per locale, AI-generated flow structures that don't conform to a fixed schema, and teams that want one database (PostgreSQL) without heavy schema management.

**Trade-offs:**
- JSONB columns are opaque to the query planner beyond GIN indexes; no foreign-key constraints on nested fields
- JSON Schema validation must be enforced at application layer (or via PostgreSQL CHECK + `jsonb_schema_validate`)
- Easier to introduce inconsistent data shapes; discipline required
- Fewer JOINs; simpler queries for common cases
- GIN indexes on JSONB are fast for containment queries (`@>`) but not range queries on nested numbers

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| JsonLogic | Condition tree stored as JsonLogic expression: `{"and": [{"==": [{"var":"user.plan"},"pro"]},{">=": [{"var":"event.count"},3]}]}` |
| JSON Schema 2020-12 | `flows.definition` validated against a versioned schema; `flow_schema_version` field enables schema evolution |
| OpenTelemetry | Inbound OTel events mapped to `journey_events.event_type`; raw OTel payload stored as JSONB `data` |
| WCAG 2.1 / WAI-ARIA | `steps[].aria` JSONB sub-object: `{"role":"dialog","label":"Welcome tour","describedby":"step-1-body"}` |
| GDPR / CCPA | `end_users.pii_hash` replaces name/email in event payloads; `tenants.data_residency_region` for routing |
| PostgreSQL RLS | Same `current_setting('app.current_tenant_id')` pattern; fewer tables to policy |

---

## Multi-Tenant Foundation

```sql
-- Tenants and projects: minimal relational spine
CREATE TABLE tenants (
    id                    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug                  TEXT NOT NULL UNIQUE,
    name                  TEXT NOT NULL,
    plan                  TEXT NOT NULL DEFAULT 'starter',
    data_residency_region TEXT NOT NULL DEFAULT 'us-east-1',
    settings              JSONB NOT NULL DEFAULT '{}',
    -- settings example: {
    --   "mau_limit": 5000,
    --   "event_retention_days": 365,
    --   "allowed_domains": ["app.acme.com"],
    --   "gdpr_enabled": true,
    --   "snapshot_interval": 50
    -- }
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE projects (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name        TEXT NOT NULL,
    slug        TEXT NOT NULL,
    sdk_key     TEXT NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(24), 'base64'),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);
CREATE INDEX idx_projects_sdk_key ON projects(sdk_key);

CREATE TABLE members (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    email       TEXT NOT NULL,
    name        TEXT NOT NULL,
    role        TEXT NOT NULL DEFAULT 'editor',
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, email)
);

ALTER TABLE projects ENABLE ROW LEVEL SECURITY;
ALTER TABLE members  ENABLE ROW LEVEL SECURITY;
CREATE POLICY rls_projects ON projects USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
CREATE POLICY rls_members  ON members  USING (tenant_id = current_setting('app.current_tenant_id')::uuid);
```

---

## Flow Definitions (JSONB Document Store)

```sql
-- A single table holds all flows; definition is the full JSONB document
CREATE TABLE flows (
    id                  UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id           UUID NOT NULL REFERENCES tenants(id),
    project_id          UUID NOT NULL REFERENCES projects(id),
    name                TEXT NOT NULL,
    flow_type           TEXT NOT NULL,   -- tour | checklist | tooltip | banner | survey | announcement
    status              TEXT NOT NULL DEFAULT 'draft',  -- draft | published | archived
    schema_version      INTEGER NOT NULL DEFAULT 1,     -- for definition schema evolution
    -- The full flow definition: steps, triggers, settings, display rules
    definition          JSONB NOT NULL DEFAULT '{}',
    -- definition shape:
    -- {
    --   "frequency": "once",
    --   "trigger": {
    --     "type": "page_visit",
    --     "url_pattern": "/dashboard*",
    --     "delay_ms": 1000,
    --     "conditions": {"and": [{"==": [{"var":"user.plan"},"pro"]}]}
    --   },
    --   "segment": {"and": [{">=": [{"var":"account.size"},10]}]},   -- JsonLogic rule
    --   "steps": [
    --     {
    --       "id": "step-uuid-1",
    --       "type": "tooltip",
    --       "name": "Welcome tooltip",
    --       "position": 0,
    --       "selector": {
    --         "primary": "[data-flow=nav-menu]",
    --         "fallback": ".main-nav > ul",
    --         "xpath": "//nav[@id='main']",
    --         "data_attr": "flow=nav-menu",
    --         "strategy": "data_attr"
    --       },
    --       "placement": "bottom",
    --       "content": {
    --         "title": "Start here",
    --         "body": "This is your main navigation.",
    --         "media": {"type": "image", "url": "https://cdn.example.com/nav.png"},
    --         "buttons": [{"label": "Next", "action": "next_step"}, {"label": "Skip", "action": "dismiss"}]
    --       },
    --       "aria": {"role": "tooltip", "label": "Navigation guide", "live": "polite"},
    --       "branch_on_action": {
    --         "click_cta": {"go_to_step": "step-uuid-3"},
    --         "dismiss": {"complete_flow": true}
    --       }
    --     },
    --     { "id": "step-uuid-2", ... }
    --   ]
    -- }
    --
    -- Published snapshot (immutable copy set at publish time)
    published_definition JSONB,
    published_at         TIMESTAMPTZ,
    published_by         UUID REFERENCES members(id),
    created_by           UUID REFERENCES members(id),
    created_at           TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at           TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_flows_tenant_project  ON flows(tenant_id, project_id);
CREATE INDEX idx_flows_status          ON flows(tenant_id, status);
-- GIN index enables: WHERE definition @> '{"trigger":{"type":"page_visit"}}'
CREATE INDEX idx_flows_definition      ON flows USING GIN (definition jsonb_path_ops);

-- Version history (append-only; one row per publish)
CREATE TABLE flow_history (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id          UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id        UUID NOT NULL REFERENCES tenants(id),
    version_number   INTEGER NOT NULL,
    definition_snap  JSONB NOT NULL,    -- immutable snapshot of `definition` at publish time
    change_summary   TEXT,
    published_by     UUID REFERENCES members(id),
    published_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (flow_id, version_number)
);
CREATE INDEX idx_flow_history_tenant ON flow_history(tenant_id, flow_id);
```

---

## Segments as JsonLogic Rules

```sql
-- Named reusable audience segments stored as JsonLogic expressions
CREATE TABLE segments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    project_id  UUID NOT NULL REFERENCES projects(id),
    name        TEXT NOT NULL,
    description TEXT,
    -- JsonLogic rule tree: evaluatable in JS SDK, Node.js backend, Python analytics
    rule        JSONB NOT NULL,
    -- rule examples:
    -- Simple: {"==": [{"var": "user.plan"}, "pro"]}
    -- Compound: {"and": [
    --   {"in": [{"var":"user.plan"}, ["pro","enterprise"]]},
    --   {">=": [{"var":"account.size"}, 10]},
    --   {"!": {"var": "user.has_completed_onboarding"}}
    -- ]}
    -- Event-count: {">=": [{"var":"event.feature_x_clicked.count"}, 3]}
    created_by  UUID REFERENCES members(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);
CREATE INDEX idx_segments_tenant ON segments(tenant_id, project_id);
CREATE INDEX idx_segments_rule   ON segments USING GIN (rule jsonb_path_ops);
```

---

## Experiments

```sql
-- Experiment config stored as JSONB for flexibility (bayesian vs frequentist params differ)
CREATE TABLE experiments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    project_id  UUID NOT NULL REFERENCES projects(id),
    name        TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'draft',  -- draft | running | paused | concluded
    config      JSONB NOT NULL DEFAULT '{}',
    -- config shape:
    -- {
    --   "stat_method": "bayesian",
    --   "confidence_level": 0.95,
    --   "primary_metric": "flow_completion_rate",
    --   "variants": [
    --     {"id": "ctrl-uuid", "flow_id": "flow-uuid-1", "name": "control", "is_control": true, "weight": 0.5},
    --     {"id": "var-uuid",  "flow_id": "flow-uuid-2", "name": "variant_a", "is_control": false, "weight": 0.5}
    --   ],
    --   "segment_rule": {"==": [{"var":"user.plan"},"pro"]},
    --   "min_sample_size": 500,
    --   "max_duration_days": 30
    -- }
    results     JSONB,              -- populated by analytics worker; null while running
    -- results shape:
    -- {
    --   "computed_at": "2026-05-15T00:00:00Z",
    --   "winner_variant_id": "var-uuid",
    --   "variants": [
    --     {"id":"ctrl-uuid","participants":512,"conversions":102,"rate":0.199,"p_value":null,"prob_best":0.12},
    --     {"id":"var-uuid","participants":510,"conversions":138,"rate":0.271,"p_value":0.003,"prob_best":0.88}
    --   ]
    -- }
    started_at  TIMESTAMPTZ,
    concluded_at TIMESTAMPTZ,
    created_by  UUID REFERENCES members(id),
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_experiments_tenant ON experiments(tenant_id, status);

-- Sticky variant assignment (relational; needs UNIQUE constraint for correctness)
CREATE TABLE experiment_assignments (
    tenant_id      UUID NOT NULL,
    end_user_id    TEXT NOT NULL,      -- opaque host-product user ID
    experiment_id  UUID NOT NULL REFERENCES experiments(id),
    variant_id     UUID NOT NULL,       -- references config->variants[].id (JSON; not FK)
    assigned_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, experiment_id, end_user_id)
);
```

---

## User Journey Events

```sql
-- End-user profiles: just the routing fields; traits in JSONB
CREATE TABLE end_users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL,
    project_id   UUID NOT NULL REFERENCES projects(id),
    external_id  TEXT NOT NULL,
    traits       JSONB NOT NULL DEFAULT '{}',   -- {"plan":"pro","company_size":50,"role":"admin"}
    group_id     TEXT,
    group_traits JSONB NOT NULL DEFAULT '{}',
    pii_hash     TEXT,                           -- SHA-256 of email for deduplication without storing PII
    first_seen   TIMESTAMPTZ,
    last_seen    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, project_id, external_id)
);
CREATE INDEX idx_end_users_tenant  ON end_users(tenant_id, project_id);
-- GIN index for trait-based segment evaluation
CREATE INDEX idx_end_users_traits  ON end_users USING GIN (traits jsonb_path_ops);
CREATE INDEX idx_end_users_group   ON end_users(tenant_id, group_id);

-- Lean event log: fewer columns than event-sourced model; JSONB payload carries detail
CREATE TABLE journey_events (
    id             BIGSERIAL,                               -- sequential for ordering
    event_id       UUID NOT NULL DEFAULT gen_random_uuid() UNIQUE,
    tenant_id      UUID NOT NULL,
    project_id     UUID NOT NULL,
    end_user_id    UUID NOT NULL REFERENCES end_users(id),
    event_type     TEXT NOT NULL,                           -- 'flow_started' | 'step_viewed' | etc.
    flow_id        UUID REFERENCES flows(id),
    data           JSONB NOT NULL DEFAULT '{}',
    -- data shape varies by event_type:
    -- flow_started:    {"flow_version": 3, "trigger_type": "page_visit"}
    -- step_viewed:     {"step_id": "...", "step_type": "tooltip", "dwell_ms": 4200}
    -- step_completed:  {"step_id": "...", "action": "button_click"}
    -- flow_dismissed:  {"at_step_id": "...", "reason": "user_x_button"}
    -- experiment_assigned: {"experiment_id":"...","variant_id":"..."}
    occurred_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (id, occurred_at)
) PARTITION BY RANGE (occurred_at);
CREATE INDEX idx_journey_events_user ON journey_events(tenant_id, end_user_id, occurred_at DESC);
CREATE INDEX idx_journey_events_flow ON journey_events(flow_id, event_type, occurred_at DESC);
CREATE INDEX idx_journey_events_type ON journey_events(tenant_id, event_type, occurred_at DESC);

-- Materialized flow completion state (updated by trigger or async worker)
CREATE TABLE user_flow_states (
    tenant_id        UUID NOT NULL,
    end_user_id      UUID NOT NULL REFERENCES end_users(id),
    flow_id          UUID NOT NULL REFERENCES flows(id),
    state            TEXT NOT NULL DEFAULT 'not_started',
    progress         JSONB NOT NULL DEFAULT '{}',
    -- progress shape: {
    --   "steps_seen": ["step-uuid-1"],
    --   "steps_completed": ["step-uuid-1"],
    --   "current_step_id": "step-uuid-2",
    --   "pct": 50.0,
    --   "started_at": "2026-05-01T09:00:00Z"
    -- }
    started_at       TIMESTAMPTZ,
    completed_at     TIMESTAMPTZ,
    PRIMARY KEY (tenant_id, end_user_id, flow_id)
);
CREATE INDEX idx_flow_states_flow ON user_flow_states(tenant_id, flow_id, state);

-- JSONB containment query example:
-- "Find all users in 'in_progress' state who have completed step-uuid-1":
-- SELECT * FROM user_flow_states
-- WHERE tenant_id = $tid AND flow_id = $fid AND state = 'in_progress'
--   AND progress @> '{"steps_completed": ["step-uuid-1"]}';
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenant | 4 | tenants, projects, members, (environments merged into projects.settings) |
| Flow authoring | 2 | flows (with embedded steps), flow_history |
| Segments | 1 | segments (JsonLogic rule) |
| Experiments | 2 | experiments (config + results JSONB), experiment_assignments |
| User journey | 3 | end_users, journey_events (partitioned), user_flow_states |
| **Total** | **12** | Half the table count of Suggestion 1 |

---

## Key Design Decisions

1. **Flow as document**: The entire flow definition — steps, step groups, triggers, segment rule, frequency, and content — is stored as a single JSONB document in `flows.definition`. This eliminates five tables (step_groups, steps, step_actions, flow_triggers, flow_segments) at the cost of losing FK integrity on step IDs. The SDK receives the definition column directly; no multi-table JOIN required.

2. **JsonLogic for conditions**: Segmentation rules use [JsonLogic](https://jsonlogic.com/) — a compact, language-agnostic JSON rule format. The same rule object evaluates identically in the browser SDK (JavaScript), the trigger backend (Node.js), and the analytics pipeline (Python's `json-logic-py`). No custom recursive evaluator needed.

3. **`published_definition` column**: Rather than a separate `flow_versions` table, the published snapshot is stored in a second JSONB column on the `flows` row (`published_definition`). Full version history goes to `flow_history` (append-only). This reduces JOIN complexity for the SDK — it reads `published_definition` if `status = 'published'`.

4. **GIN indexes for trait queries**: `end_users.traits` has a GIN index, enabling fast containment queries like `WHERE traits @> '{"plan":"enterprise"}'` for segment evaluation. Combined with JsonLogic rule evaluation in-process, tenant-scoped segment matching runs without full-table scans.

5. **Experiment results as JSONB**: Both Bayesian (probability_best, credible intervals) and frequentist (p_value, significance) outputs fit in a single `experiments.results` JSONB column. The analytics worker writes the entire results object atomically, avoiding partially-updated multi-row result tables.

6. **Fewer tables = simpler migrations**: With 12 tables vs 24, schema migrations are half as complex. Adding a new field to step content means editing the JSONB shape (no migration) and updating the JSON Schema validator. Adding a new event type means inserting a row in the application's event schema registry (not a DB migration).

7. **Partitioned event log retained**: Despite the JSONB-heavy approach, `journey_events` remains a relational partitioned table (not JSONB documents) because event volume is high, ordering matters, and time-range queries must be fast. Only the `data` payload column is JSONB.
