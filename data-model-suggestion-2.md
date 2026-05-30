# Data Model Suggestion 2: Event-Sourced / Journey-First

> Project: User Onboarding Flow Builder · Created: 2026-05-22

## Philosophy

The immutable event log is the single source of truth for everything observable about a user's onboarding journey. Flow definitions are stored relationally (they are authored data, not events), but all user-facing state — completion, progress, experiment assignments, checklist status — is derived by replaying a stream of typed events. Periodic snapshots prevent full replay from the beginning of time; a CQRS read model materialises the current state for fast API responses.

This mirrors how Temporal and Apache Airflow model workflow execution: each step transition appends a record rather than updating a status column. It makes the complete "what happened and when" history permanently queryable, supports temporal queries ("what was the state on 15 March?"), and feeds AI/ML pipelines directly from the raw event stream.

**Best for:** Teams prioritising full audit history, AI-powered journey analysis, temporal debugging ("why did this user not complete step 3?"), and compliance environments requiring immutable behavioural records.

**Trade-offs:**
- Higher write volume; mitigate with Kafka/SQS fan-out for high-MAU tenants
- Current state requires materialisation; adds async projection infrastructure
- Schema evolution of event types needs careful versioning (upcasting)
- Simpler read queries once read model is built; complex to set up initially
- Excellent fit for OpenTelemetry event ingestion (OTel → event store)

---

## Standards Alignment

| Standard | How It's Used |
|----------|---------------|
| OpenTelemetry | Inbound OTel spans map directly to `journey_events.event_type`; `trace_id`/`span_id` preserved |
| CloudEvents 1.0 | `journey_events` schema aligns with CloudEvents spec: `specversion`, `source`, `type`, `subject`, `time`, `data` |
| GDPR / CCPA | Events are immutable; erasure handled via `gdpr_erasure_log` + tombstone events; event payloads exclude PII in favour of opaque `end_user_id` |
| JSON Schema 2020-12 | Each `event_type` has a registered JSON Schema; `event_schemas` table validates payloads on insert |
| OpenAPI 3.1 | Read model projection endpoints are OpenAPI-documented; event ingestion endpoint uses CloudEvents content-type |
| PostgreSQL RLS | `journey_events.tenant_id` carries RLS policy; snapshot tables likewise |

---

## Multi-Tenant Foundation

```sql
-- Same tenant/project/environment/members model as suggestion 1
-- (omitted for brevity — identical schema)
-- Key addition: tenant-level event retention policy
ALTER TABLE tenants ADD COLUMN event_retention_days INTEGER NOT NULL DEFAULT 730;
ALTER TABLE tenants ADD COLUMN snapshot_interval    INTEGER NOT NULL DEFAULT 50; -- snapshot every N events
```

---

## Flow Definition (Authored Data — Relational)

Flow definitions are authored relationally (same as Suggestion 1's flows, step_groups, steps tables). They are NOT event-sourced — they are the "recipe", not the "execution". The key difference is that execution state lives entirely in the event store.

```sql
-- Flows, step_groups, steps, step_actions: same as Suggestion 1
-- flow_versions: same snapshot-on-publish pattern

-- Event type registry: canonical list of known event types with their JSON Schema
CREATE TABLE event_schemas (
    event_type     TEXT PRIMARY KEY,  -- 'flow_started' | 'step_viewed' | 'step_completed' | etc.
    json_schema    JSONB NOT NULL,    -- JSON Schema 2020-12 validating the event payload
    description    TEXT,
    introduced_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
    deprecated_at  TIMESTAMPTZ        -- null = current
);

-- Seed the canonical event types
INSERT INTO event_schemas (event_type, description, json_schema) VALUES
  ('flow_started',           'User began a flow',                           '{"type":"object","required":["flow_id","flow_version_id"]}'),
  ('step_viewed',            'User saw a step',                             '{"type":"object","required":["step_id","dwell_ms"]}'),
  ('step_completed',         'User completed required step action',         '{"type":"object","required":["step_id"]}'),
  ('step_skipped',           'User skipped an optional step',               '{"type":"object","required":["step_id"]}'),
  ('flow_completed',         'User reached the end of the flow',            '{"type":"object","required":["flow_id"]}'),
  ('flow_dismissed',         'User explicitly closed/dismissed the flow',   '{"type":"object","required":["flow_id","dismissed_at_step_id"]}'),
  ('checklist_item_started', 'User began a checklist item',                 '{"type":"object","required":["item_id"]}'),
  ('checklist_item_completed','User completed a checklist item',            '{"type":"object","required":["item_id"]}'),
  ('experiment_assigned',    'User bucketed into an experiment variant',    '{"type":"object","required":["experiment_id","variant_id"]}'),
  ('nps_submitted',          'User submitted NPS score',                    '{"type":"object","required":["score"]}'),
  ('custom_event',           'Host product fired a named custom event',     '{"type":"object","required":["event_name"]}');
```

---

## The Event Store

```sql
-- The core append-only event store (partitioned by month)
-- CloudEvents-aligned schema
CREATE TABLE journey_events (
    -- CloudEvents envelope fields
    id            UUID NOT NULL DEFAULT gen_random_uuid(),  -- CloudEvents `id`
    spec_version  TEXT NOT NULL DEFAULT '1.0',              -- CloudEvents `specversion`
    source        TEXT NOT NULL,                            -- CloudEvents `source` (SDK endpoint)
    event_type    TEXT NOT NULL,                            -- CloudEvents `type` (namespaced, e.g. 'io.flowbuilder.flow_started')
    subject       TEXT NOT NULL,                            -- CloudEvents `subject` = end_user_id
    time          TIMESTAMPTZ NOT NULL DEFAULT now(),       -- CloudEvents `time`
    -- Routing fields (denormalized for partitioned query performance)
    tenant_id     UUID NOT NULL,
    project_id    UUID NOT NULL,
    environment_id UUID NOT NULL,
    end_user_id   UUID NOT NULL,
    -- Domain fields
    stream_id     UUID NOT NULL,     -- aggregate stream: (tenant_id, end_user_id, flow_id) hash → UUID
    stream_seq    BIGINT NOT NULL,   -- monotonically increasing per stream; for ordering & concurrency
    -- Payload
    data          JSONB NOT NULL DEFAULT '{}',
    -- Observability
    otel_trace_id TEXT,
    otel_span_id  TEXT,
    -- Immutability guard: no UPDATE/DELETE permissions granted to app role
    PRIMARY KEY (id, time)          -- compound PK required for Postgres partitioning
) PARTITION BY RANGE (time);

-- Enforce ordering within each stream (prevents duplicate/out-of-order events)
CREATE UNIQUE INDEX idx_journey_events_stream_seq
    ON journey_events(stream_id, stream_seq);

CREATE INDEX idx_journey_events_tenant_user
    ON journey_events(tenant_id, end_user_id, time DESC);

CREATE INDEX idx_journey_events_type
    ON journey_events(tenant_id, event_type, time DESC);

-- Revoke mutation rights from application role
REVOKE UPDATE, DELETE ON journey_events FROM app_role;

-- Monthly partitions
CREATE TABLE journey_events_2026_05 PARTITION OF journey_events
    FOR VALUES FROM ('2026-05-01') TO ('2026-06-01');

-- GDPR erasure: tombstone events anonymise PII without deleting rows
-- Erasure is handled by updating end_user_id → anonymised UUID and logging
CREATE TABLE gdpr_erasure_log (
    id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id       UUID NOT NULL,
    original_end_user_id  UUID NOT NULL,
    anonymised_end_user_id UUID NOT NULL,
    erased_at       TIMESTAMPTZ NOT NULL DEFAULT now(),
    erased_by       UUID,              -- member who triggered erasure
    event_count     INTEGER            -- how many events were anonymised
);
```

---

## Aggregate Snapshots

```sql
-- Snapshot table: periodic materialised state to avoid full event replay
-- Taken every N events (configurable per tenant)
CREATE TABLE journey_snapshots (
    id               UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    tenant_id        UUID NOT NULL,
    stream_id        UUID NOT NULL,     -- same stream_id as journey_events
    end_user_id      UUID NOT NULL,
    flow_id          UUID NOT NULL,
    at_seq           BIGINT NOT NULL,   -- stream_seq at which snapshot was taken
    state            JSONB NOT NULL,    -- full aggregate state at this point
    -- state example: {
    --   "flow_state": "in_progress",
    --   "steps_seen": ["step-uuid-1", "step-uuid-2"],
    --   "steps_completed": ["step-uuid-1"],
    --   "current_step_id": "step-uuid-2",
    --   "started_at": "2026-05-01T10:00:00Z",
    --   "checklist_items": { "item-uuid-1": "completed", "item-uuid-2": "not_started" }
    -- }
    taken_at         TIMESTAMPTZ NOT NULL DEFAULT now(),
    UNIQUE (stream_id, at_seq)
);
CREATE INDEX idx_snapshots_stream ON journey_snapshots(stream_id, at_seq DESC);
```

---

## CQRS Read Models (Projections)

```sql
-- Projection: current flow state per user (rebuilt from events + latest snapshot)
-- Updated asynchronously after each event batch; NOT the source of truth
CREATE TABLE proj_user_flow_states (
    tenant_id        UUID NOT NULL,
    end_user_id      UUID NOT NULL,
    flow_id          UUID NOT NULL,
    state            TEXT NOT NULL,   -- not_started | in_progress | completed | dismissed | skipped
    progress_pct     NUMERIC(5,2),   -- 0.00–100.00
    steps_completed  INTEGER NOT NULL DEFAULT 0,
    steps_total      INTEGER NOT NULL DEFAULT 0,
    started_at       TIMESTAMPTZ,
    completed_at     TIMESTAMPTZ,
    last_event_seq   BIGINT NOT NULL,  -- stream_seq of last event projected
    projected_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, end_user_id, flow_id)
);

-- Projection: experiment variant assignments (read-side cache)
CREATE TABLE proj_experiment_assignments (
    tenant_id      UUID NOT NULL,
    end_user_id    UUID NOT NULL,
    experiment_id  UUID NOT NULL,
    variant_id     UUID NOT NULL,
    assigned_at    TIMESTAMPTZ NOT NULL,
    last_event_seq BIGINT NOT NULL,
    PRIMARY KEY (tenant_id, end_user_id, experiment_id)
);

-- Projection: aggregate experiment metrics (recomputed by analytics worker)
CREATE TABLE proj_experiment_metrics (
    tenant_id        UUID NOT NULL,
    experiment_id    UUID NOT NULL,
    variant_id       UUID NOT NULL,
    metric_date      DATE NOT NULL,
    participants     INTEGER NOT NULL DEFAULT 0,
    conversions      INTEGER NOT NULL DEFAULT 0,
    conversion_rate  NUMERIC(8,6),
    p_value          NUMERIC(8,6),
    probability_best NUMERIC(8,6),
    computed_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, experiment_id, variant_id, metric_date)
);

-- Projection: tenant-level activation funnel (daily rollup)
CREATE TABLE proj_activation_funnel (
    tenant_id      UUID NOT NULL,
    project_id     UUID NOT NULL,
    flow_id        UUID NOT NULL,
    metric_date    DATE NOT NULL,
    started        INTEGER NOT NULL DEFAULT 0,
    step_1_seen    INTEGER NOT NULL DEFAULT 0,
    step_2_seen    INTEGER NOT NULL DEFAULT 0,
    completed      INTEGER NOT NULL DEFAULT 0,
    dismissed      INTEGER NOT NULL DEFAULT 0,
    median_time_ms BIGINT,
    computed_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
    PRIMARY KEY (tenant_id, flow_id, metric_date)
);
```

---

## Temporal Query Pattern

```sql
-- "What was user X's state on flow Y as of 2026-03-15?"
-- 1. Find latest snapshot before the target date:
SELECT state, at_seq FROM journey_snapshots
WHERE stream_id = $stream_id AND taken_at <= '2026-03-15'
ORDER BY at_seq DESC LIMIT 1;

-- 2. Replay events from that snapshot's at_seq up to the target date:
SELECT * FROM journey_events
WHERE stream_id = $stream_id
  AND stream_seq > $snapshot_seq
  AND time <= '2026-03-15'
ORDER BY stream_seq ASC;

-- 3. Apply events to snapshot state in application code → point-in-time state
```

---

## Condition & Trigger Storage

Segmentation rules and triggers are stored identically to Suggestion 1 (relational condition tree + `flow_triggers` table). The difference is that rule evaluation at runtime queries the `proj_user_flow_states` and `end_users.traits` JSONB rather than raw event replays.

---

## Table Count Summary

| Category | Tables | Notes |
|----------|--------|-------|
| Event store | 2 | journey_events (partitioned), event_schemas |
| Snapshots | 1 | journey_snapshots |
| CQRS projections | 4 | proj_user_flow_states, proj_experiment_assignments, proj_experiment_metrics, proj_activation_funnel |
| Flow authoring | 4 | flows, flow_versions, step_groups, steps (same as suggestion 1) |
| Targeting | 3 | flow_triggers, segments, segment_conditions |
| Experiments | 2 | experiments, experiment_variants (assignments via events) |
| Compliance | 1 | gdpr_erasure_log |
| Multi-tenant | 5 | tenants, projects, environments, api_keys, members |
| **Total** | **22** | |

---

## Key Design Decisions

1. **CloudEvents envelope**: `journey_events` fields (`spec_version`, `source`, `event_type`, `subject`, `time`, `data`) align with the CloudEvents 1.0 spec. This lets OpenTelemetry-instrumented host products push spans directly into the event store with minimal transformation.

2. **Stream identity**: `stream_id` is a deterministic UUID derived from `(tenant_id, end_user_id, flow_id)`. All events for one user's journey through one flow share a stream. The `stream_seq` monotonically increases within a stream; a `UNIQUE (stream_id, stream_seq)` constraint prevents out-of-order writes and acts as an optimistic concurrency guard.

3. **Snapshot cadence**: Every `tenant.snapshot_interval` events (default 50), the projection worker writes a `journey_snapshots` row. Cold reads replay only from the latest snapshot forward — worst case 50 events, not thousands.

4. **GDPR erasure without deletion**: Event rows are never deleted (immutability). Erasure anonymises: `end_user_id` in all events for that user is replaced with a new random UUID; the mapping is recorded in `gdpr_erasure_log`. The event history is preserved for aggregate analytics; the individual is unidentifiable.

5. **Projection lag tolerance**: Read models (`proj_*` tables) are asynchronous. The API returns projection state plus a `projected_at` timestamp. For real-time SDK decisions (e.g., "should I show this flow now?"), the SDK queries `proj_user_flow_states` and accepts up to 5-second lag; for analytics, lag is tolerated up to the batch cadence.

6. **Event schema registry**: `event_schemas` stores the JSON Schema for each event type. The ingestion endpoint validates `data` against the registered schema on write, preventing malformed events from polluting the store.

7. **Experiments via events**: Variant assignment is recorded as an `experiment_assigned` event (not a mutable table row). The `proj_experiment_assignments` projection caches the current assignment for fast SDK lookups. This means experiment assignment history is fully auditable and replayable.
