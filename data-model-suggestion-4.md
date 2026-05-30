# Data Model Suggestion 4: Graph-Relational (Flow as DAG)

> Project: User Onboarding Flow Builder · Created: 2026-05-22

## Philosophy

Onboarding flows with conditional branching, looping back to earlier steps, and dynamic paths based on user actions are fundamentally directed graphs — not linear sequences. This model makes that structure explicit: steps are `graph_nodes` and transitions between them are `graph_edges` carrying condition expressions. The adjacency-list graph lives inside PostgreSQL using two tables (`flow_nodes`, `flow_edges`) and recursive CTEs for traversal — no graph database required.

A separate relational spine handles multi-tenancy, publishing, and event tracking. The graph layer handles what other models struggle with: "if the user clicks CTA, go to step 5; if they dismiss, show a recovery modal; if they've already seen feature X, skip the tour". This is the architectural pattern used by Temporal (workflow graphs), Apache Airflow (DAG task dependencies), and advanced experiment platforms (multi-armed bandits with path-dependent treatment).

**Best for:** Products with complex conditional onboarding logic (role-based paths, feature-gated steps, recovery flows for at-risk users), AI-generated branching flows, and teams who want the flow structure to be first-class queryable data rather than embedded JSONB.

**Trade-offs:**
- More complex than linear-sequence models for simple use cases
- Graph traversal queries (recursive CTEs) have higher cognitive overhead
- Edge condition evaluation at runtime requires a rule evaluator per edge
- Excellent fit for AI path optimisation: the graph structure is directly traversable by ML algorithms
- Supports loops (re-engagement flows) — DAGs don't permit cycles, but with a `max_visits` guard on edges, safe re-entry is possible

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| Directed Acyclic Graph (DAG) | `flow_edges` with `source_node_id` → `target_node_id`; a `CHECK` constraint prevents self-loops; cycle detection via recursive CTE at write time |
| JsonLogic | Edge transition conditions: `{"and":[{"==": [{"var":"action"},"cta_click"]},{"!": {"var":"user.has_seen_feature_x"}}]}` |
| OpenTelemetry | Node visits recorded as OTel spans; `trace_id` links SDK execution trace to journey events |
| WCAG 2.1 | `flow_nodes.content->>'aria_role'` and `aria_label` fields in node content JSONB |
| PostgreSQL ltree | Alternative to adjacency list for path recording: `flow_nodes.path ltree` stores materialised path from root node |
| GDPR / CCPA | `end_users.pii_hash`; event payloads use opaque IDs; erasure via anonymisation |
| PostgreSQL RLS | `tenant_id` on all tables with `current_setting` policy |

---

## Multi-Tenant Foundation

```sql
-- Same as Suggestion 1 (tenants, projects, environments, api_keys, members)
-- Abbreviated here; identical schema
CREATE TABLE tenants (
    id    UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    slug  TEXT NOT NULL UNIQUE,
    name  TEXT NOT NULL,
    plan  TEXT NOT NULL DEFAULT 'starter',
    data_residency_region TEXT NOT NULL DEFAULT 'us-east-1',
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE projects (
    id         UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id  UUID NOT NULL REFERENCES tenants(id) ON DELETE CASCADE,
    name       TEXT NOT NULL,
    slug       TEXT NOT NULL,
    sdk_key    TEXT NOT NULL UNIQUE DEFAULT encode(gen_random_bytes(24), 'base64'),
    created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, slug)
);
```

---

## Flow as DAG

```sql
-- Flow container (metadata only; structure is in nodes/edges)
CREATE TABLE flows (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL REFERENCES tenants(id),
    project_id   UUID NOT NULL REFERENCES projects(id),
    name         TEXT NOT NULL,
    flow_type    TEXT NOT NULL DEFAULT 'tour',  -- tour | checklist | tooltip | survey | banner
    status       TEXT NOT NULL DEFAULT 'draft', -- draft | published | archived
    frequency    TEXT NOT NULL DEFAULT 'once',  -- once | every_session | every_time
    entry_node_id UUID,                         -- FK set after node creation (deferred)
    published_at TIMESTAMPTZ,
    published_by UUID,
    schema_version INTEGER NOT NULL DEFAULT 1,
    created_by   UUID,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_flows_tenant ON flows(tenant_id, project_id, status);

-- DAG nodes: each step is a node in the graph
CREATE TABLE flow_nodes (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id     UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    node_type   TEXT NOT NULL,  -- 'step' | 'decision' | 'action' | 'terminal'
    -- 'step'     = displays content to user (tooltip, modal, hotspot, etc.)
    -- 'decision' = invisible branching node (evaluates conditions, routes to child)
    -- 'action'   = fires a side effect (webhook, event, redirect) without display
    -- 'terminal' = end of path (flow_completed or flow_dismissed)
    name        TEXT NOT NULL,
    -- Element targeting (for 'step' nodes)
    selector    JSONB,
    -- selector shape: {
    --   "primary": "[data-flow=nav-menu]",
    --   "fallback": ".main-nav",
    --   "xpath": "//nav[@id='main']",
    --   "data_attr": "flow=nav-menu",
    --   "text_match": "Get started",
    --   "strategy": "data_attr"
    -- }
    -- Content (for 'step' nodes)
    content     JSONB,
    -- content shape: {
    --   "step_type": "tooltip",
    --   "placement": "bottom",
    --   "title": "Your dashboard",
    --   "body": "This is where you'll spend most of your time.",
    --   "media": {"type": "video", "url": "https://loom.com/..."},
    --   "buttons": [{"label": "Next", "fires": "cta_click"}, {"label": "Skip", "fires": "dismiss"}],
    --   "aria": {"role": "tooltip", "label": "Dashboard tour step", "live": "polite"},
    --   "skippable": true
    -- }
    -- Action config (for 'action' nodes)
    action_config JSONB,
    -- action_config: {"type":"fire_event","event_name":"onboarding_started"} |
    --                {"type":"webhook","url":"https://hooks.example.com/..."} |
    --                {"type":"redirect","url":"/billing"}
    -- Materialised path for fast ancestor queries (ltree extension)
    path        TEXT,   -- e.g. 'root.node-a.node-b.this-node' (updated on edge changes)
    position_x  INTEGER DEFAULT 0,   -- visual canvas X (for builder UI)
    position_y  INTEGER DEFAULT 0,   -- visual canvas Y
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_flow_nodes_flow ON flow_nodes(flow_id);
CREATE INDEX idx_flow_nodes_tenant ON flow_nodes(tenant_id);
CREATE INDEX idx_flow_nodes_content ON flow_nodes USING GIN (content jsonb_path_ops);

-- DAG edges: directed transitions between nodes with optional conditions
CREATE TABLE flow_edges (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id         UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    source_node_id  UUID NOT NULL REFERENCES flow_nodes(id) ON DELETE CASCADE,
    target_node_id  UUID NOT NULL REFERENCES flow_nodes(id) ON DELETE CASCADE,
    -- Transition trigger (what user action fires this edge)
    trigger_action  TEXT NOT NULL,  -- 'cta_click' | 'dismiss' | 'skip' | 'auto' | 'element_click' | 'timer' | 'any'
    -- Optional condition (JsonLogic): if set, edge only followed when rule evaluates true
    condition       JSONB,
    -- condition examples:
    -- Always follow: null
    -- Only if plan is pro: {"==": [{"var":"user.plan"},"pro"]}
    -- Only if user hasn't seen feature: {"!": {"var":"user.has_seen_feature_x"}}
    -- Only on 3rd visit: {">=": [{"var":"user.visit_count"},3]}
    priority        SMALLINT NOT NULL DEFAULT 0,  -- lower = evaluated first for same trigger_action
    label           TEXT,                          -- optional edge label for builder UI
    max_visits      SMALLINT DEFAULT 1,            -- >1 allows re-entry (safe re-engagement loops)
    -- Prevent self-loops at DB level
    CONSTRAINT no_self_loop CHECK (source_node_id <> target_node_id),
    UNIQUE (source_node_id, target_node_id, trigger_action)
);
CREATE INDEX idx_flow_edges_source ON flow_edges(source_node_id);
CREATE INDEX idx_flow_edges_flow   ON flow_edges(flow_id);

-- Graph traversal: find all reachable nodes from entry point
-- WITH RECURSIVE reachable AS (
--   SELECT n.* FROM flow_nodes n WHERE n.id = $entry_node_id
--   UNION ALL
--   SELECT n.* FROM flow_nodes n
--   JOIN flow_edges e ON e.target_node_id = n.id
--   JOIN reachable r ON r.id = e.source_node_id
-- ) SELECT * FROM reachable;

-- Cycle detection (run at edge insertion time in application layer):
-- WITH RECURSIVE path_check AS (
--   SELECT target_node_id, ARRAY[source_node_id] AS visited
--   FROM flow_edges WHERE source_node_id = $new_source
--   UNION ALL
--   SELECT e.target_node_id, pc.visited || e.source_node_id
--   FROM flow_edges e JOIN path_check pc ON e.source_node_id = pc.target_node_id
--   WHERE NOT e.source_node_id = ANY(pc.visited)
-- ) SELECT 1 FROM path_check WHERE target_node_id = $new_source LIMIT 1;
-- If this returns a row, the new edge would create a cycle — reject it.
```

---

## Published Flow Snapshot

```sql
-- Published snapshot: serialise the entire graph (nodes + edges) as JSONB at publish time
CREATE TABLE flow_versions (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id         UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    version_number  INTEGER NOT NULL,
    -- Full graph snapshot: {nodes: [...], edges: [...], entry_node_id: "..."}
    graph_snapshot  JSONB NOT NULL,
    change_summary  TEXT,
    published_by    UUID,
    published_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    is_live         BOOLEAN NOT NULL DEFAULT false,
    UNIQUE (flow_id, version_number)
);
CREATE UNIQUE INDEX idx_flow_versions_live ON flow_versions(flow_id) WHERE is_live = true;
```

---

## Targeting: Triggers and Segments

```sql
-- Flow-level triggers (when to attempt to show the flow)
CREATE TABLE flow_triggers (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    flow_id      UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    tenant_id    UUID NOT NULL REFERENCES tenants(id),
    trigger_type TEXT NOT NULL,  -- 'page_visit' | 'event' | 'element_click' | 'api_call' | 'manual'
    config       JSONB NOT NULL,
    -- config examples:
    -- page_visit: {"url_pattern": "/dashboard*", "delay_ms": 500}
    -- event:      {"event_name": "feature_x_clicked", "min_count": 1}
    -- api_call:   {}   (triggered programmatically by host product)
    priority     SMALLINT NOT NULL DEFAULT 0,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- Segments stored as JsonLogic (same pattern as Suggestion 3)
CREATE TABLE segments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    project_id  UUID NOT NULL REFERENCES projects(id),
    name        TEXT NOT NULL,
    rule        JSONB NOT NULL,   -- JsonLogic rule tree
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, name)
);

CREATE TABLE flow_segments (
    flow_id    UUID NOT NULL REFERENCES flows(id) ON DELETE CASCADE,
    segment_id UUID NOT NULL REFERENCES segments(id) ON DELETE CASCADE,
    PRIMARY KEY (flow_id, segment_id)
);
```

---

## Experiments

```sql
-- Experiments: variant maps to an entry_node_id within the same flow (graph branch)
-- OR to a completely different flow version
CREATE TABLE experiments (
    id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id   UUID NOT NULL REFERENCES tenants(id),
    project_id  UUID NOT NULL REFERENCES projects(id),
    name        TEXT NOT NULL,
    status      TEXT NOT NULL DEFAULT 'draft',
    stat_method TEXT NOT NULL DEFAULT 'bayesian',  -- bayesian | frequentist
    primary_metric TEXT NOT NULL,
    started_at  TIMESTAMPTZ,
    concluded_at TIMESTAMPTZ,
    created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_variants (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    experiment_id   UUID NOT NULL REFERENCES experiments(id) ON DELETE CASCADE,
    tenant_id       UUID NOT NULL REFERENCES tenants(id),
    name            TEXT NOT NULL,
    is_control      BOOLEAN NOT NULL DEFAULT false,
    traffic_weight  NUMERIC(5,4) NOT NULL DEFAULT 0.5,
    -- Variant can point to a different flow OR a different graph entry node
    flow_id         UUID REFERENCES flows(id),
    entry_node_id   UUID REFERENCES flow_nodes(id),   -- null = use flow's default entry
    created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE experiment_assignments (
    tenant_id      UUID NOT NULL,
    end_user_id    TEXT NOT NULL,
    experiment_id  UUID NOT NULL REFERENCES experiments(id),
    variant_id     UUID NOT NULL REFERENCES experiment_variants(id),
    assigned_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, experiment_id, end_user_id)
);
```

---

## User Journey: Node-Level Execution State

```sql
CREATE TABLE end_users (
    id           UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id    UUID NOT NULL,
    project_id   UUID NOT NULL REFERENCES projects(id),
    external_id  TEXT NOT NULL,
    traits       JSONB NOT NULL DEFAULT '{}',
    group_id     TEXT,
    group_traits JSONB NOT NULL DEFAULT '{}',
    first_seen   TIMESTAMPTZ,
    last_seen    TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (tenant_id, project_id, external_id)
);
CREATE INDEX idx_end_users_tenant  ON end_users(tenant_id, project_id);
CREATE INDEX idx_end_users_traits  ON end_users USING GIN (traits jsonb_path_ops);

-- Node-level visit tracking (fine-grained; supports per-node analytics)
CREATE TABLE user_node_visits (
    id              BIGSERIAL PRIMARY KEY,
    tenant_id       UUID NOT NULL,
    end_user_id     UUID NOT NULL REFERENCES end_users(id),
    flow_id         UUID NOT NULL REFERENCES flows(id),
    flow_version_id UUID REFERENCES flow_versions(id),
    node_id         UUID NOT NULL REFERENCES flow_nodes(id),
    visit_number    SMALLINT NOT NULL DEFAULT 1,  -- which traversal of this node
    entered_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    exited_at       TIMESTAMPTZ,
    exit_action     TEXT,         -- which action the user took ('cta_click','dismiss',etc.)
    exit_edge_id    UUID REFERENCES flow_edges(id),
    dwell_ms        INTEGER,
    otel_trace_id   TEXT
) PARTITION BY RANGE (entered_at);
CREATE INDEX idx_node_visits_user_flow ON user_node_visits(tenant_id, end_user_id, flow_id, entered_at DESC);
CREATE INDEX idx_node_visits_node      ON user_node_visits(node_id, entered_at DESC);

-- Flow-level execution state (derived from node visits)
CREATE TABLE user_flow_executions (
    id                UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id         UUID NOT NULL,
    end_user_id       UUID NOT NULL REFERENCES end_users(id),
    flow_id           UUID NOT NULL REFERENCES flows(id),
    flow_version_id   UUID REFERENCES flow_versions(id),
    state             TEXT NOT NULL DEFAULT 'not_started',  -- not_started | in_progress | completed | dismissed | skipped
    current_node_id   UUID REFERENCES flow_nodes(id),
    nodes_visited     INTEGER NOT NULL DEFAULT 0,
    terminal_reason   TEXT,   -- 'completed' | 'dismissed' | 'expired'
    started_at        TIMESTAMPTZ,
    completed_at      TIMESTAMPTZ,
    last_activity_at  TIMESTAMPTZ,
    path_taken        UUID[],  -- array of node IDs in traversal order (for AI path analysis)
    UNIQUE (tenant_id, end_user_id, flow_id)
);
CREATE INDEX idx_flow_executions_flow ON user_flow_executions(tenant_id, flow_id, state);

-- Path analysis query: "What paths do users take through this flow?"
-- SELECT path_taken, COUNT(*), AVG(nodes_visited) FROM user_flow_executions
-- WHERE flow_id = $fid AND state = 'completed'
-- GROUP BY path_taken ORDER BY count DESC;
```

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Multi-tenant | 4 | tenants, projects, api_keys, members |
| Flow graph | 3 | flows, flow_nodes, flow_edges |
| Versioning | 1 | flow_versions (graph snapshot JSONB) |
| Targeting | 3 | flow_triggers, segments, flow_segments |
| Experiments | 3 | experiments, experiment_variants, experiment_assignments |
| User journey | 3 | end_users, user_node_visits (partitioned), user_flow_executions |
| **Total** | **17** | |

---

## Key Design Decisions

1. **Step types as node types**: The `flow_nodes.node_type` enum distinguishes `step` (renders UI), `decision` (invisible condition branching), `action` (fires webhook/event), and `terminal` (ends the flow). Decision nodes enable "if user.plan = enterprise, skip to step 5" without embedding that logic inside step content.

2. **Edge conditions as JsonLogic**: `flow_edges.condition` stores a JsonLogic expression evaluated at traversal time against `{user: traits, event: last_event, action: fired_action}`. A null condition means "always follow this edge". Multiple edges from the same node with the same `trigger_action` are evaluated in `priority` order; the first matching edge wins.

3. **Cycle detection at write time**: The application layer runs a recursive CTE cycle-detection query before inserting any edge. If the new edge would create a cycle (source reachable from target), the insert is rejected. `max_visits > 1` on re-engagement edges is the only sanctioned form of looping.

4. **`path_taken` array for AI training**: `user_flow_executions.path_taken` stores the ordered array of `flow_node.id` values traversed. This makes "which path leads to highest completion?" directly queryable with array operators, and is immediately usable as training data for an ML model optimising flow paths.

5. **Node-level `user_node_visits`**: Unlike other models that track only step-level or flow-level events, this model records every node visit including `dwell_ms`, `exit_action`, and `exit_edge_id`. This enables per-node drop-off analysis: "62% of users who reach decision node D1 take the 'dismiss' edge rather than 'cta_click'".

6. **Graph snapshot at publish**: `flow_versions.graph_snapshot` serialises the entire graph as `{nodes: [...], edges: [...], entry_node_id: "..."}` at publish time. The SDK receives this snapshot and runs the graph traversal client-side. Live execution never touches `flow_nodes`/`flow_edges` (the mutable draft); it always traverses the published snapshot.

7. **Experiment variants as graph entry points**: An `experiment_variant` can specify a different `entry_node_id` within the same flow — routing variant users into a different starting node of the same graph. This enables A/B testing different onboarding paths within a single flow definition rather than managing two completely separate flows.
