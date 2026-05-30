# User Onboarding Flow Builder — Phased Development Plan

> Project: 302-user-onboarding-flow-builder · Created: 2026-05-30
> Purpose: Provide sufficient detail for Claude Code (Opus) to implement each phase end-to-end.

This plan synthesizes `research.md`, `features.md`, `standards.md`, `README.md`, and the four `data-model-suggestion-*.md` files. The primary data model is **Suggestion 3 (Hybrid Relational + JSONB)** for MVP velocity, evolved to incorporate **Suggestion 4 (Graph/DAG)** for conditional branching in Phase 9, and **Suggestion 2 (event-store discipline)** for the append-only journey event log throughout.

---

## Core Requirements (Synthesis)

**What it does:** An AI-native, open-source platform for SaaS product/growth/CS teams to build in-app onboarding experiences (tours, checklists, tooltips, hotspots, banners, surveys) without engineering effort, target them behaviourally and by segment, measure activation, and run A/B experiments — at a fraction of incumbent (Appcues/Pendo/Chameleon) pricing.

**Who uses it:** (1) *Members* — product managers, growth, and CS team members who author flows in the builder UI; (2) *End-users* — users of the customer's instrumented product who experience the flows via the JS SDK.

**Key differentiators:** Open-source + self-hostable; CloudEvents-native event ingestion (no incumbent offers this); WCAG 2.2 / WAI-ARIA 1.3 compliant overlays with an automated publish-gate a11y check (a genuine gap in the market); CSP-nonce-aware SDK injection; AI-native flow authoring (natural language → flow), AI a11y/copy assistance, and predictive drop-off detection.

**Deployment model:** Hybrid. SaaS control plane (multi-tenant Postgres + REST API + builder web UI) plus a client-side JavaScript SDK injected into the customer's product. Self-hostable via Docker Compose. Mobile via a thin React Native SDK (post-MVP).

**Integration surface:** REST API (OpenAPI 3.1), JS SDK, CloudEvents inbound webhook ingestion, HMAC-signed outbound webhooks, OAuth 2.0 + PKCE / Client Credentials, OIDC ID-token bootstrap, LLM provider (for AI features).

**Standards the build must implement:** WCAG 2.2 AA + WAI-ARIA 1.3 (P0), CSP Level 3 nonce (P0), GDPR Art. 6 + ePrivacy consent gating (P0), CCPA/GPC (P1), OpenAPI 3.1 + JSON Schema 2020-12 (P1), HMAC-SHA256 webhooks (P1), OAuth 2.0 + PKCE (P1), CloudEvents v1.0 (P2), OpenTelemetry events (P2), ACT Rules 1.1 via axe-core (P3).

---

## Technology Decisions

| Concern | Choice | Rationale |
|---------|--------|-----------|
| Language (server + SDK) | TypeScript (Node.js 22 LTS) | Same language for control-plane API, builder UI, and the browser SDK. Segmentation rules use JsonLogic, which must evaluate identically in-browser and server-side (Suggestion 3 design decision) — a single TS implementation removes reimplementation risk. |
| Monorepo tooling | pnpm workspaces + Turborepo | Shared types (flow definition schema, JsonLogic, event types) consumed by `api`, `web`, and `sdk` packages. Turbo caches builds and tests across packages. |
| API framework | Fastify 5 + `@fastify/swagger` | High-throughput (the event ingestion endpoint is hot), first-class JSON Schema validation that doubles as the OpenAPI 3.1 source of truth, plugin model for auth/RLS hooks. |
| Validation / schema | Zod + `zod-to-json-schema` | Single Zod schema per resource → runtime validation + JSON Schema 2020-12 export for the published flow-definition schema and the OpenAPI doc. Satisfies the standards requirement that flow definitions be machine-readable and versioned. |
| Database | PostgreSQL 16 | Suggestion 3 (Hybrid Relational+JSONB) needs JSONB + GIN indexes; partitioned append-only event log; Row-Level Security for tenant isolation; recursive CTEs for the Phase 9 DAG. All native to Postgres — no second datastore for MVP. |
| Migrations / query | Drizzle ORM + drizzle-kit | Type-safe schema in TS (shared with Zod types), explicit SQL-first migrations, supports RLS `SET LOCAL` and partition DDL that heavier ORMs hide. |
| Cache / queue | Redis 7 + BullMQ | SDK config delivery is read-heavy → cache published flow bundles. BullMQ runs async workers: event projection (`user_flow_states`), webhook delivery with retry, experiment metric recomputation, a11y publish-gate. |
| Rule engine | `json-logic-js` (browser + server) | Portable segmentation/edge-condition format per Suggestions 3 & 4; one evaluator everywhere. |
| Overlay rendering (SDK) | Vanilla TS + Floating UI (`@floating-ui/dom`) | Zero host-framework dependency; Floating UI handles tooltip/popover placement and the WCAG 2.2 "Focus Not Obscured" (2.4.11/2.4.12) collision avoidance. Shadow DOM isolates styles from the host page. |
| A11y testing | axe-core (implements ACT Rules 1.1) | Runs in the SDK preview and in the server-side publish-gate worker against a headless render. Stores per-step `ActRuleResult`. |
| Builder web UI | Next.js 15 (App Router) + React 19 + Tailwind + shadcn/ui | Server components for the dashboard/analytics; the drag-and-drop flow canvas is a client island. ATAG 2.0 Part A requires the builder itself be keyboard-navigable — shadcn/Radix primitives are accessible by default. |
| Flow canvas | React Flow (`@xyflow/react`) | Phase 9 graph editor; node/edge model maps directly to Suggestion 4's `flow_nodes`/`flow_edges`. Used in linear mode for MVP, branching mode later. |
| Auth (members) | Lucia-style sessions + OAuth 2.0/OIDC server (`@fastify/oauth2` patterns) | Members log in via email/password or SSO. The API issues OAuth tokens (Client Credentials for server-to-server, Auth Code + PKCE for SDK) per standards.md P1. |
| LLM provider | Vercel AI SDK + provider-agnostic gateway | AI features (NL→flow, copy assist, a11y fixes, drop-off scoring) need model routing/failover and structured (tool-call) output. Provider-agnostic keeps the open-source project from hard-coding one vendor. |
| Charts | Recharts | Analytics dashboards (funnels, completion rates, experiment results). |
| Testing | Vitest (unit/integration), Playwright (E2E + SDK browser + a11y) | Vitest for fast TS unit/integration with Testcontainers-backed Postgres; Playwright drives the real SDK in a real browser and runs axe-core scans. |
| Test infra | Testcontainers (Postgres + Redis) | Integration tests run against real Postgres (RLS, partitions, GIN behave differently from mocks) spun up per suite. |
| Lint / format / types | ESLint (flat config) + Prettier + `tsc --noEmit` | Standard TS quality gate; enforced in CI and per-phase Definition of Done. |
| Containerisation | Docker + docker-compose | Self-host story (README). Compose brings up api, web, worker, Postgres, Redis. SDK ships to a CDN/npm. |
| CI | GitHub Actions | Lint → typecheck → unit → integration (Testcontainers) → build → Playwright E2E + axe-core gate. |

---

### Project Structure

```
user-onboarding-flow-builder/
├── package.json                  # pnpm workspace root
├── pnpm-workspace.yaml
├── turbo.json
├── docker-compose.yml            # api + web + worker + postgres + redis
├── tsconfig.base.json
├── .github/workflows/ci.yml
├── packages/
│   ├── shared/                   # cross-cutting types & logic (no runtime deps on api/web)
│   │   ├── src/
│   │   │   ├── flow-schema.ts     # Zod FlowDefinition (steps, triggers, segment, a11y)
│   │   │   ├── flow-schema.json   # generated JSON Schema 2020-12 (versioned)
│   │   │   ├── events.ts          # journey event types + CloudEvents envelope
│   │   │   ├── jsonlogic.ts       # typed wrapper over json-logic-js
│   │   │   ├── segmentation.ts    # evaluateSegment(rule, ctx)
│   │   │   └── selectors.ts       # selector strategy resolution (css/xpath/data/text)
│   │   └── tests/
│   ├── db/                        # Drizzle schema, migrations, RLS helpers
│   │   ├── src/
│   │   │   ├── schema/            # tenants, projects, flows, segments, experiments, events…
│   │   │   ├── rls.ts             # withTenant(tx, tenantId) → SET LOCAL
│   │   │   ├── partitions.ts      # monthly partition creation for journey_events
│   │   │   └── client.ts
│   │   ├── migrations/
│   │   └── tests/
│   ├── api/                       # Fastify REST API + OAuth + webhook ingestion
│   │   ├── src/
│   │   │   ├── server.ts
│   │   │   ├── plugins/           # auth, rls-context, rate-limit, swagger
│   │   │   ├── routes/            # flows, segments, experiments, sdk-config, events, webhooks, ai
│   │   │   ├── services/          # flow publishing, segment eval, experiment bucketing
│   │   │   └── openapi.ts         # assembles OpenAPI 3.1 from route Zod schemas
│   │   └── tests/
│   ├── worker/                    # BullMQ background jobs
│   │   ├── src/jobs/              # projection, webhook-delivery, experiment-metrics, a11y-gate, dropoff-scoring
│   │   └── tests/
│   ├── sdk/                       # browser JavaScript SDK (npm + CDN bundle)
│   │   ├── src/
│   │   │   ├── index.ts           # Flowbuilder.init/identify/track/page
│   │   │   ├── transport.ts       # batched event POST, config fetch
│   │   │   ├── consent.ts         # GDPR/ePrivacy/GPC gating
│   │   │   ├── matcher.ts         # trigger + segment evaluation client-side
│   │   │   ├── renderer/          # shadow-DOM overlays: tooltip, modal, hotspot, banner, checklist
│   │   │   ├── a11y/              # focus mgmt, ARIA application, axe preview
│   │   │   └── csp.ts             # nonce application
│   │   └── tests/                 # Vitest + Playwright
│   ├── web/                       # Next.js builder + dashboard
│   │   ├── app/
│   │   │   ├── (auth)/
│   │   │   ├── (app)/flows/       # list, builder canvas, preview
│   │   │   ├── (app)/segments/
│   │   │   ├── (app)/experiments/
│   │   │   └── (app)/analytics/
│   │   └── components/
│   └── sdk-react-native/          # Phase 11 (post-MVP)
└── fixtures/                      # shared test fixtures (sample flows, events, host pages)
```

---

## Phase 1: Foundation — Monorepo, Database Core, Tenant Isolation

### Purpose
Establish the monorepo, the Postgres schema for multi-tenancy, and the RLS-based tenant isolation that every subsequent phase depends on. After this phase, the project builds, lints, type-checks, runs migrations against a real Postgres, and can create tenants/projects/members with guaranteed cross-tenant isolation.

### Tasks

#### 1.1 — Monorepo & toolchain bootstrap
**What:** Initialise the pnpm/Turbo workspace with all packages, shared tsconfig, ESLint/Prettier, Vitest, and CI.

**Design:**
- `pnpm-workspace.yaml` lists `packages/*`. Root scripts: `build`, `lint`, `typecheck`, `test`, `test:integration`, `test:e2e` delegated through `turbo`.
- `tsconfig.base.json`: `"strict": true`, `"target": "ES2022"`, `"moduleResolution": "bundler"`, path aliases `@flowbuilder/shared`, `@flowbuilder/db`.
- ESLint flat config with `@typescript-eslint`, `eslint-plugin-import`; Prettier.
- `turbo.json` pipeline: `lint`, `typecheck`, `test` depend on `^build`.
- CI workflow stages: install → lint → typecheck → test → test:integration.

**Testing:**
- `Unit: turbo run typecheck → exit 0` on an empty-but-valid workspace.
- `CI smoke: a trivial Vitest test in packages/shared passes in CI.`

#### 1.2 — Database schema: multi-tenant foundation
**What:** Drizzle schema + migration for `tenants`, `projects`, `members`, `api_keys`, with RLS enabled (Suggestion 3 spine, `environments` folded into `projects` for MVP).

**Design (Drizzle, mirrors Suggestion 3 SQL):**
```ts
export const tenants = pgTable('tenants', {
  id: uuid('id').primaryKey().defaultRandom(),
  slug: text('slug').notNull().unique(),
  name: text('name').notNull(),
  plan: text('plan').notNull().default('starter'),       // starter|growth|enterprise
  dataResidencyRegion: text('data_residency_region').notNull().default('us-east-1'),
  settings: jsonb('settings').$type<TenantSettings>().notNull().default({}),
  createdAt: timestamptz('created_at').notNull().defaultNow(),
  updatedAt: timestamptz('updated_at').notNull().defaultNow(),
});
// TenantSettings = { mauLimit:number; eventRetentionDays:number; gdprEnabled:boolean; snapshotInterval:number; allowedDomains:string[] }
```
- `projects` carries `sdk_key TEXT UNIQUE DEFAULT encode(gen_random_bytes(24),'base64')`, `UNIQUE(tenant_id, slug)`.
- `members`: `role` enum `owner|admin|editor|viewer`, `UNIQUE(tenant_id, email)`, password hash via argon2.
- `api_keys`: `key_hash` (argon2), `scopes text[]`, `expires_at`, `last_used_at`.
- RLS: every tenant-scoped table `ENABLE ROW LEVEL SECURITY` with policy `USING (tenant_id = current_setting('app.current_tenant_id')::uuid)`.
- `withTenant(tx, tenantId, fn)` helper: opens a transaction, runs `SET LOCAL app.current_tenant_id = $1`, executes `fn`. Setting expires with the transaction.

**Testing:**
- `Integration (Testcontainers PG): migrate up → all tables + policies exist (query pg_policies).`
- `Integration: insert tenant A + project; insert tenant B + project; within withTenant(A), SELECT projects returns only A's rows.`
- `Integration: attempt to SELECT without app.current_tenant_id set → 0 rows (RLS denies).`
- `Unit: api_key creation returns raw key once; stored value is an argon2 hash that verifies.`

#### 1.3 — Monthly partition helper for the event log (forward-declared)
**What:** A `partitions.ts` utility + migration that creates `journey_events` as a `PARTITION BY RANGE (occurred_at)` parent and a job to roll monthly partitions.

**Design:**
- `journey_events` parent table per Suggestion 3 (`id BIGSERIAL`, `event_id UUID UNIQUE`, composite PK `(id, occurred_at)`).
- `ensureMonthlyPartition(date)`: `CREATE TABLE IF NOT EXISTS journey_events_YYYY_MM PARTITION OF journey_events FOR VALUES FROM (..) TO (..)`.
- Seed current + next month at migration time.

**Testing:**
- `Integration: ensureMonthlyPartition for 2026-06 → partition exists; insert event with occurred_at in June lands in it (check tableoid).`
- `Integration: insert into a month with no partition → error surfaced clearly (guards against silent loss).`

---

## Phase 2: Flow Definition Schema & Flow CRUD

### Purpose
Define the canonical, versioned flow-definition document (the heart of the product) as a Zod schema that exports JSON Schema 2020-12, and build the CRUD + publish lifecycle around it. After this phase, a flow can be created, edited as a draft, validated, and published to an immutable snapshot — with nothing yet rendering it.

### Tasks

#### 2.1 — FlowDefinition schema (`packages/shared`)
**What:** The Zod schema for a complete flow document, matching Suggestion 3's `definition` shape, exported to versioned JSON Schema.

**Design:**
```ts
export const StepType = z.enum(['tooltip','modal','hotspot','banner','checklist_item','survey_question']);
export const Selector = z.object({
  primary: z.string().optional(), fallback: z.string().optional(),
  xpath: z.string().optional(), dataAttr: z.string().optional(), textMatch: z.string().optional(),
  strategy: z.enum(['css','xpath','text','data_attr','compound']).default('css'),
});
export const Aria = z.object({                       // standards.md WAI-ARIA mapping
  role: z.enum(['dialog','alertdialog','tooltip','status','button']).optional(),
  label: z.string().optional(), labelledBy: z.string().optional(),
  describedBy: z.string().optional(), live: z.enum(['polite','assertive','off']).optional(),
  focusTarget: z.string().optional(),
});
export const StepButton = z.object({ label:z.string(), action:z.enum(['next_step','prev_step','go_to_step','complete_flow','dismiss','open_url','fire_event']), data:z.record(z.any()).optional() });
export const Step = z.object({
  id: z.string().uuid(), type: StepType, name: z.string(), position: z.number().int(),
  selector: Selector.optional(), placement: z.enum(['top','bottom','left','right','center','auto']).default('auto'),
  content: z.object({ title:z.string().optional(), body:z.string().optional(),
    media:z.object({type:z.enum(['image','video']), url:z.string().url()}).optional(),
    buttons:z.array(StepButton).default([]) }),
  aria: Aria.optional(), skippable: z.boolean().default(true),
  requiredAction: z.enum(['click_element','fill_form']).nullable().default(null),
});
export const Trigger = z.object({
  type: z.enum(['page_visit','event','element_click','api_call','manual']),
  urlPattern: z.string().optional(), eventName: z.string().optional(),
  delayMs: z.number().int().default(0), conditions: JsonLogicRule.optional(),
});
export const FlowDefinition = z.object({
  schemaVersion: z.literal(1),
  frequency: z.enum(['once','every_session','every_time']).default('once'),
  trigger: Trigger, segment: JsonLogicRule.optional(),
  steps: z.array(Step).min(1),
}).strict();
```
- Build step: `zod-to-json-schema` emits `flow-schema.json` (JSON Schema 2020-12) into `packages/shared`, versioned by `schemaVersion`. This is the published, machine-readable schema required by standards.md.
- `validateFlowDefinition(doc)`: returns typed result or structured errors (path + message).

**Testing:**
- `Unit: minimal valid tour (1 step) → parses.`
- `Unit: step with unknown field → .strict() rejects, error path includes field.`
- `Unit: empty steps array → "min 1" error.`
- `Unit: generated flow-schema.json validates the same fixtures via ajv (2020-12) — Zod and JSON Schema agree.`

#### 2.2 — Flow + flow_history tables and CRUD service
**What:** `flows` (with `definition` + `published_definition` JSONB) and append-only `flow_history` per Suggestion 3; service layer for create/read/update/list.

**Design:**
- `flows` columns: `name`, `flow_type`, `status (draft|published|archived)`, `schema_version`, `definition jsonb`, `published_definition jsonb`, `published_at`, `published_by`, audit cols. GIN index `USING GIN (definition jsonb_path_ops)`.
- `FlowService.update(tenantId, flowId, definition)`: validates with `validateFlowDefinition`, stores draft, bumps `updated_at`.
- All methods run inside `withTenant`.

**Testing:**
- `Integration: create draft → GET returns it; status=draft.`
- `Integration: update with invalid definition → 422, draft unchanged.`
- `Integration: tenant B cannot read tenant A's flow (RLS).`

#### 2.3 — Publish lifecycle
**What:** Publish endpoint that snapshots the draft into `published_definition` + writes a `flow_history` row.

**Design:**
- State machine: `draft → published → (archived)`; re-publishing a published flow creates a new `flow_history` version (`version_number` = max+1) and overwrites `published_definition`.
- `publish(tenantId, flowId, memberId, changeSummary)`:
  1. Validate current `definition`.
  2. (Phase 8 hook) enqueue a11y publish-gate; for now publish synchronously.
  3. `INSERT flow_history (version_number, definition_snap, change_summary, published_by)`.
  4. `UPDATE flows SET status='published', published_definition=definition, published_at=now()`.
- Editing a published flow's draft never alters `published_definition` (live execution reads only the published snapshot).

**Testing:**
- `Integration: publish draft → published_definition set, history v1 created.`
- `Integration: edit definition after publish → published_definition unchanged; GET ?view=live returns the snapshot.`
- `Integration: publish twice → history has v1 and v2; version_number strictly increases.`
- `Unit: publish a flow with zero steps → rejected before snapshot.`

---

## Phase 3: REST API, Auth, and OpenAPI

### Purpose
Stand up the Fastify API with OAuth 2.0 auth, RLS request context, and an auto-generated OpenAPI 3.1 document, exposing the Phase 2 flow operations. After this phase, an authenticated client can manage flows over HTTP and the API self-documents.

### Tasks

#### 3.1 — Fastify server, RLS context, error model
**What:** Bootstrap the server with a per-request tenant-context plugin and a uniform error envelope.

**Design:**
- `rls-context` plugin: resolves `tenantId` from the authenticated principal, opens a request-scoped transaction, runs `SET LOCAL app.current_tenant_id`, and attaches `req.tx`.
- Error envelope: `{ error: { code, message, details?, requestId } }`; map Zod errors → 422, auth → 401/403, not found → 404.
- Health: `GET /healthz` (liveness), `GET /readyz` (DB + Redis ping).

**Testing:**
- `Integration: request without auth → 401 envelope.`
- `Integration: DB down → /readyz 503; /healthz 200.`
- `Integration: handler throwing Zod error → 422 with field details.`

#### 3.2 — OAuth 2.0 + member sessions + API keys
**What:** Two auth paths: member sessions for the web UI; OAuth 2.0 (Client Credentials for server-to-server, Auth Code + PKCE for SDK/3rd-party) and API keys for the REST API. Aligns with standards.md P1.

**Design:**
- `OAuthClient { clientId, clientSecretHash, grantTypes[], redirectUris[], scopes[], pkceRequired }`.
- Token endpoint `POST /oauth/token`: `client_credentials` (server backends) and `authorization_code` + PKCE (RFC 7636: verifier 43–128 chars → SHA-256 → base64url challenge). Implicit flow not supported.
- Bearer JWT access tokens (short-lived) with `tenant_id`, `scopes` claims; verified by an `auth` plugin that sets the principal.
- API-key auth: `Authorization: Bearer fb_live_…`; looked up by argon2 hash, scopes enforced per route.
- OIDC bootstrap (forward hook for SDK): accept a customer IdP ID token to derive end-user identity (`sub`, `email`, `org_id`, `plan`).

**Testing:**
- `Integration: client_credentials with valid secret → access token with correct scopes.`
- `Integration: auth_code without PKCE when pkceRequired → 400 invalid_request.`
- `Integration: PKCE verifier mismatch → 400 invalid_grant.`
- `Integration: flows:read token hitting POST /flows → 403 insufficient_scope.`
- `Unit: expired API key → 401.`

#### 3.3 — Flow routes + OpenAPI 3.1 generation
**What:** REST routes for flows backed by Phase 2 services; assemble OpenAPI 3.1 from route Zod schemas.

**Design:**
- Routes: `GET/POST /v1/flows`, `GET/PATCH /v1/flows/:id`, `POST /v1/flows/:id/publish`, `POST /v1/flows/:id/archive`, `GET /v1/flows/:id/history`.
- Each route declares Zod request/response; `zod-to-json-schema` → component schemas reused from `flow-schema.json` (JSON Schema 2020-12). `@fastify/swagger` serves `/openapi.json` (3.1) and Swagger UI at `/docs`.
- Pagination: cursor-based (`?cursor=&limit=`), responses wrap `{ data, nextCursor }`.

**Testing:**
- `Integration: full CRUD+publish cycle over HTTP with a scoped token.`
- `Contract: /openapi.json validates against the OpenAPI 3.1 meta-schema.`
- `Contract: FlowDefinition component in OpenAPI === packages/shared/flow-schema.json.`
- `E2E: generate a TS client from /openapi.json and round-trip create→publish.`

---

## Phase 4: JavaScript SDK — Identity, Events, Consent

### Purpose
Build the browser SDK's non-visual core: initialization, identity, event tracking with batching, and consent gating. This is the data-collection backbone every flow, segment, and analytic depends on, and it must be privacy-correct from day one. After this phase, a host page can load the SDK, identify a user, and emit events that land in `journey_events`.

### Tasks

#### 4.1 — SDK init, transport, and CSP nonce
**What:** `Flowbuilder.init({ sdkKey, apiHost, nonce? })`, batched event transport, config fetch.

**Design:**
- `init` resolves the project by `sdkKey` (CDN config endpoint `GET /v1/sdk/:sdkKey/config`), stores it, sets up a flush timer.
- Transport: queue events, flush on 20 events or 5s or `visibilitychange:hidden` via `navigator.sendBeacon` fallback to `fetch(keepalive)`. Endpoint `POST /v1/events` (batch).
- CSP (standards.md P0): all injected `<script>`/`<style>` carry the host-supplied `nonce`; SDK documents required `connect-src`/`script-src`/`style-src` origins. Styles live in a Shadow DOM to avoid host CSS bleed.

**Testing:**
- `Unit: 20 queued events → one batched POST; payload shape correct.`
- `Unit (jsdom): visibilitychange hidden → sendBeacon called with the queue.`
- `Integration (Playwright, strict CSP host page): SDK loads, injects nonce'd style, no CSP violation in console.`

#### 4.2 — Consent gating (GDPR / ePrivacy / GPC)
**What:** No tracking storage or network call until a consent signal permits it (standards.md P0/P1).

**Design:**
- `ConsentSignal { granted:boolean; categories:string[]; timestamp; source:'user'|'gpc' }`.
- SDK reads `navigator.globalPrivacyControl`; if GPC true → behave as opt-out (no analytics tracking) until explicit grant.
- Before any `identify`/`track` writes localStorage or POSTs: evaluate consent. If not granted, buffer in memory only (no persistence) and drop on unload.
- Host calls `Flowbuilder.consent({ granted, categories })`. Default posture configurable per project (`gdprEnabled` → deny-by-default).

**Testing:**
- `Unit: gdprEnabled + no consent → track() writes nothing, no fetch, no localStorage.`
- `Unit: GPC=true → analytics category treated as opted-out even after generic consent.`
- `Unit: consent granted → buffered events flush; localStorage write occurs.`
- `Integration (Playwright): cookie/storage tab empty until consent granted.`

#### 4.3 — identify / track / page + event ingestion endpoint
**What:** Client methods plus the server `POST /v1/events` that validates and writes to `journey_events`/`end_users`.

**Design:**
- `identify(externalId, traits)`, `track(eventName, props)`, `page(name, props)`, `group(groupId, traits)`.
- Server upserts `end_users` (`UNIQUE(tenant_id, project_id, external_id)`), merges `traits` JSONB; appends `journey_events` rows (`event_type='custom_event'` etc., `data` JSONB) into the correct monthly partition.
- Event validation against the `events.ts` registry; reject unknown shapes (defensive, mirrors Suggestion 2's schema-registry discipline).
- All events carry `consent_flags` snapshot.

**Testing:**
- `Integration: batch of identify+track → end_user upserted, traits merged, events inserted with consent_flags.`
- `Integration: malformed event in batch → that event 422'd, valid ones still written (partial success report).`
- `Integration: events for tenant A never visible to tenant B (RLS).`
- `Load (smoke): 1k events/req batch completes < 200ms against Testcontainers PG.`

---

## Phase 5: SDK Renderer & Accessible Overlays

### Purpose
Render the actual onboarding experiences in the browser — tooltips, modals, hotspots, banners — with WCAG 2.2 / WAI-ARIA 1.3 compliance baked in. This is where the product becomes visible to end-users. After this phase, a published linear flow displays correctly and accessibly on a host page.

### Tasks

#### 5.1 — Selector resolution & element targeting
**What:** Resolve a step's target element using the multi-strategy selector (Suggestion 1/3 selector design).

**Design:**
- `resolveTarget(selector): Element|null` tries strategies in order: `data_attr` → `primary` (css) → `fallback` (css) → `xpath` → `textMatch`. Returns first hit.
- Handles late-rendering DOM via a `MutationObserver` with a timeout (default 5s) before declaring the target missing → emits `step_target_not_found` event.

**Testing:**
- `Unit (jsdom): data-attr present → resolves via data_attr even if css also matches.`
- `Unit: primary missing, fallback present → resolves via fallback.`
- `Unit: target appears after 1s → MutationObserver resolves it.`
- `Unit: never appears → null after timeout, event emitted.`

#### 5.2 — Accessible overlay components (Shadow DOM)
**What:** Renderers for `modal`(dialog), `alertdialog`, `tooltip`, `hotspot`, `banner`, `checklist`, each applying the ARIA roles/attributes from standards.md.

**Design (per standards.md WAI-ARIA table):**
- Modal/alertdialog: `role=dialog|alertdialog`, `aria-modal="true"`, `aria-labelledby`→title id, `aria-describedby`→body id. Background gets `inert`.
- Tooltip: `role=tooltip` with `id`; trigger gets `aria-describedby`. Content dismissable via Escape, hoverable, persistent (WCAG 1.4.13).
- Progress (`status` live region) for checklist completion announcements.
- Floating UI positions overlays and applies WCAG 2.2 2.4.11/2.4.12 (Focus Not Obscured) collision avoidance so sticky banners/tooltips never cover the focused element.
- Each renderer reads `step.aria`, `step.placement`, `step.content`. All styling inside Shadow DOM; close button `aria-label="Close"`.

**Testing:**
- `Unit: modal renders with aria-modal, labelledby/describedby pointing to real ids.`
- `Playwright + axe-core: each overlay type → zero serious/critical ACT violations.`
- `Playwright: open tooltip on focus, press Escape → dismisses; hover keeps it open.`
- `Playwright: banner near focused input → Floating UI repositions so input remains visible.`

#### 5.3 — Focus management & keyboard navigation
**What:** WCAG 2.1.1/2.1.2/2.4.3/2.4.7 focus handling across a flow.

**Design:**
- On step open: move focus to `aria.focusTarget` or first interactive element; trap focus within the overlay only for `dialog`/`alertdialog` (Escape always exits — no keyboard trap, 2.1.2).
- On close/advance: return focus to the triggering element (2.4.3).
- Visible focus ring enforced via Shadow DOM CSS (2.4.7).
- Tour navigation: Tab/Shift-Tab within overlay; Enter activates primary button; Escape dismisses (records `flow_dismissed`).

**Testing:**
- `Playwright: open modal → focus inside; Tab cycles within; Shift-Tab wraps; Escape closes and focus returns to trigger.`
- `Playwright: full 3-step tour driven entirely by keyboard → completes, emits flow_completed.`
- `axe-core: no focus-trap or focus-order violations across the tour.`

#### 5.4 — Flow runtime (linear) & state events
**What:** Drive a published linear flow start→finish, emitting journey events at each transition.

**Design:**
- `FlowRunner` loads `published_definition`, iterates `steps` by `position`, renders each, listens for button actions (`next_step`/`prev_step`/`dismiss`/`complete_flow`/`open_url`/`fire_event`).
- Emits: `flow_started`, `step_viewed`(with `dwell_ms`), `step_completed`, `step_skipped`, `flow_completed`, `flow_dismissed` — the canonical event types from Suggestion 2's registry.
- Respects `frequency` (once/every_session/every_time) using `user_flow_states` (Phase 6) + sessionStorage.

**Testing:**
- `Playwright: 3-step tour, click Next×2 + Done → 1 flow_started, 3 step_viewed, 1 flow_completed.`
- `Playwright: dismiss at step 2 → flow_dismissed with at_step_id.`
- `Unit: frequency=once + already completed → runner does not start.`

---

## Phase 6: Targeting Engine — Segments, Triggers, SDK Config Delivery

### Purpose
Decide *who* sees *which* flow *when*. Implement JsonLogic segmentation (shared evaluator), trigger matching, and the SDK config bundle that ships published flows + their targeting to the browser. After this phase, flows appear automatically for the right users on the right pages.

### Tasks

#### 6.1 — Segments (JsonLogic) CRUD + shared evaluator
**What:** `segments` table (Suggestion 3: `rule jsonb` = JsonLogic) + `evaluateSegment(rule, context)` in `packages/shared`.

**Design:**
- `segments`: `name`, `rule jsonb`, GIN index, `UNIQUE(tenant_id, name)`.
- `evaluateSegment(rule, { user: traits, account: groupTraits, event })` wraps `json-logic-js`; identical in SDK and server.
- Routes: `GET/POST/PATCH/DELETE /v1/segments`. Rule validated for known operators on save.

**Testing:**
- `Unit: {"==":[{"var":"user.plan"},"pro"]} vs {plan:"pro"} → true; vs {plan:"free"} → false.`
- `Unit: compound and/or/in/! rule evaluates correctly across fixtures.`
- `Unit: identical result in SDK build and server build for the same rule+context (cross-runtime fixture).`
- `Integration: invalid operator on save → 422.`

#### 6.2 — Trigger matching & frequency
**What:** Server + SDK logic that decides whether a flow's `trigger` fires for the current page/event, honoring `frequency`.

**Design:**
- Trigger types: `page_visit` (URL glob match), `event` (event name + optional count from `journey_events`), `element_click`, `api_call` (host calls `Flowbuilder.start(flowId)`), `manual`.
- `trigger.conditions` (JsonLogic) gate firing in addition to segment membership.
- Frequency enforced against `user_flow_states` + sessionStorage so a `once` flow never re-shows.

**Testing:**
- `Unit: url_pattern "/dashboard*" matches "/dashboard/x", not "/settings".`
- `Unit: event trigger min_count=3 fires only on the 3rd matching event.`
- `Unit: frequency=every_session → shows once per session, again next session.`

#### 6.3 — SDK config bundle endpoint
**What:** `GET /v1/sdk/:sdkKey/config` returns published flows + segments + triggers for the project, cached in Redis.

**Design:**
- Response: `{ flows:[{ id, frequency, trigger, segment, definition: published_definition }], updatedAt }`. Only `status=published` flows.
- Redis cache keyed by `sdkKey`, invalidated on publish/archive. ETag support; SDK sends `If-None-Match`.
- SDK evaluates segment+trigger client-side using `evaluateSegment` against the identified user's traits → picks the flow to run.

**Testing:**
- `Integration: publish flow → config bundle includes it; archive → excluded.`
- `Integration: second request with matching ETag → 304.`
- `Integration: cache invalidated on publish (stale flow not served).`
- `E2E (Playwright): host page with pro user on /dashboard → targeted tour auto-starts; free user → no tour.`

---

## Phase 7: Builder Web UI & Analytics Dashboard

### Purpose
Give non-technical members the no-code authoring experience — the table-stakes feature of every competitor — plus the activation analytics that make onboarding measurable. After this phase, a PM can build, preview, target, publish a flow, and read its completion metrics without touching code. ATAG 2.0 Part A (accessible authoring tool) applies.

### Tasks

#### 7.1 — Auth UI, project switcher, flow list
**What:** Next.js auth pages, tenant/project context, and a flow list with status/tags.

**Design:**
- App Router: `(auth)/login`, `(app)/flows`. Server components fetch via the API with the member session. shadcn/ui table; filters by status/type/tag.
- Accessible by keyboard (Radix primitives); honors prefers-reduced-motion.

**Testing:**
- `E2E: log in → land on flows list scoped to the member's tenant.`
- `axe-core: login + flow-list pages have no critical violations.`

#### 7.2 — Visual flow builder (linear) with live preview
**What:** Drag-and-drop step authoring with a side panel for content/selector/targeting and an embedded SDK preview.

**Design:**
- React Flow in linear mode: steps as ordered nodes; reorder updates `position`. Side panel edits `content`, `selector`, `placement`, `aria`, buttons → writes the `FlowDefinition` draft via `PATCH /flows/:id`.
- Element picker: "pick element" opens the preview in an iframe and captures the clicked element's resilient selector (prefers `data-*`/`data-testid`).
- Live preview runs the real SDK renderer against a sample/host URL in an iframe.
- ATAG Part B hooks (Phase 8): inline warnings for missing alt text / low contrast / missing ARIA label surface here.

**Testing:**
- `E2E: add 2 steps, set content+selector, save → draft definition matches expected JSON.`
- `E2E: reorder steps → positions persist.`
- `E2E: element picker click → selector populated with data-attr strategy.`
- `axe-core: builder canvas operable by keyboard (ATAG A) — drag has keyboard alternative.`

#### 7.3 — Segment & trigger builder UI
**What:** Visual builder for JsonLogic segments and a targeting panel on each flow.

**Design:**
- Condition-group UI (AND/OR groups, property/operator/value rows) ↔ JsonLogic rule. Live "matches N users" preview via a server count endpoint.
- Flow targeting panel: choose trigger type + config, attach segment, set frequency.

**Testing:**
- `E2E: build (plan in [pro,enterprise]) AND (signups>=10) → JsonLogic matches expected; preview count returned.`
- `Unit: UI rule ↔ JsonLogic round-trips without loss.`

#### 7.4 — Analytics dashboard (funnel, completion, drop-off)
**What:** Per-flow analytics from the projected state (Phase 6 events) — completion rate, step funnel, drop-off step.

**Design:**
- Worker projects `journey_events` → `user_flow_states` and a daily `proj_activation_funnel` rollup (Suggestion 2 projections).
- API `GET /v1/flows/:id/analytics?from&to` returns `{ started, completedRate, byStep:[{stepId, viewed, completed, droppedPct}], medianTimeMs }`.
- Recharts funnel + completion trend; highlights the biggest drop-off step.

**Testing:**
- `Integration: seed events for 100 users (60 complete, 40 drop at step 2) → analytics: completedRate 0.6, step2 droppedPct highest.`
- `E2E: dashboard renders funnel; drop-off step flagged.`
- `Integration: analytics respect from/to window.`

---

## Phase 8: Accessibility Publish-Gate & Outbound Webhooks

### Purpose
Operationalize the project's headline differentiator — automated accessibility validation before a flow goes live — and add outbound webhooks so customer systems can react to onboarding events. After this phase, inaccessible flows are caught at publish time and external systems receive signed event notifications.

### Tasks

#### 8.1 — axe-core publish-gate (ACT Rules 1.1)
**What:** A worker job that headlessly renders each step and runs axe-core, blocking or warning on publish per severity.

**Design:**
- On `publish`, enqueue `a11y-gate(flowId)`. Worker uses Playwright (headless) to render each step's overlay against a synthetic host page, runs axe-core, stores `ActRuleResult[]` per step: `{ ruleId, outcome:'passed'|'failed'|'inapplicable', impact, element }`.
- Policy: `critical`/`serious` failures block publish (configurable to warn) for the ACT rules in standards.md (`5f99a7`, `b5c3f8`, `3e12e1`, `afw4f7`, …). Results surfaced inline in the builder (ATAG Part B).
- `flows.last_a11y_audit jsonb` stores the latest result set.

**Testing:**
- `Integration: flow with a dialog missing accessible name → gate reports b5c3f8 fail, publish blocked.`
- `Integration: compliant flow → gate passes, publish proceeds.`
- `Integration: gate set to warn-mode → publish proceeds with stored warnings.`
- `E2E: builder shows the inline a11y warning produced by the gate.`

#### 8.2 — AI accessibility & copy assistance
**What:** AI helpers that propose ARIA labels / alt text and improved step copy, using the gate results as input.

**Design:**
- `POST /v1/ai/a11y-fix` (step + axe findings) → structured suggestions `{ field, suggestion }` via the AI SDK (tool-call/structured output). Author applies with one click.
- `POST /v1/ai/copy` (step content + goal) → improved title/body variants.
- Prompts templated; no auto-apply (human-in-the-loop). Provider-agnostic via the gateway.

**Testing:**
- `Integration (mocked LLM): a11y-fix returns aria.label for an unlabeled dialog; applying it makes the gate pass.`
- `Unit: prompt template includes the failing ACT ruleIds and the step content.`
- `Integration: provider error → graceful 503, builder still usable.`

#### 8.3 — Outbound webhooks (HMAC-SHA256)
**What:** Deliver onboarding events to customer endpoints with signed, retried, replay-protected requests (standards.md P1).

**Design:**
- `webhook_endpoints { url, secret (encrypted at rest), signingAlgorithm:'hmac-sha256', events[], timeoutMs, active }`.
- Delivery (BullMQ): sign the raw body `HMAC-SHA256(secret, rawBytes)` → `X-Webhook-Signature: sha256=<b64>`, add `X-Webhook-Timestamp`; retry with exponential backoff; log `{ deliveredAt, statusCode, responseMs, signatureValid }`.
- Events: `flow.completed`, `flow.dismissed`, `experiment.concluded`, etc. (CloudEvents-typed payload).

**Testing:**
- `Integration: completion event → endpoint receives request with valid HMAC over raw bytes + timestamp.`
- `Unit: signature verified with constant-time compare; tampered body fails.`
- `Integration: 500 from endpoint → retried per backoff; delivery log records attempts.`
- `Unit: timestamp older than 300s rejected by the verification helper.`

---

## Phase 9: Conditional Branching (Graph/DAG Flows)

### Purpose
Upgrade flows from linear sequences to directed graphs with conditional branching — the foundation for role-based paths, feature-gated steps, and recovery flows, and the structure AI path-optimization needs. This adopts Suggestion 4 (Graph-Relational) layered onto the existing flow model. After this phase, a flow can branch on user action or traits.

### Tasks

#### 9.1 — Graph schema (nodes/edges) & cycle guard
**What:** Add `flow_nodes` and `flow_edges` (Suggestion 4) as the branching representation; `entry_node_id` on flows.

**Design:**
- `flow_nodes { node_type: step|decision|action|terminal, selector jsonb, content jsonb, action_config jsonb, position_x/y }`.
- `flow_edges { source_node_id, target_node_id, trigger_action, condition jsonb (JsonLogic), priority, max_visits, CHECK(source<>target), UNIQUE(source,target,trigger_action) }`.
- Cycle detection via recursive CTE at edge-insert time (reject edges that make `source` reachable from `target`); `max_visits>1` is the only sanctioned re-entry.
- `FlowDefinition.schemaVersion: 2` adds an optional `graph: { entryNodeId, nodes, edges }`; linear `steps` remain valid (v1) for simple flows.

**Testing:**
- `Integration: insert edge creating a cycle → rejected by cycle-check.`
- `Integration: valid DAG of 4 nodes → reachable-set CTE returns all from entry.`
- `Unit: schemaVersion 1 (linear) and 2 (graph) both validate.`

#### 9.2 — Decision nodes & edge evaluation runtime
**What:** Extend the SDK `FlowRunner` to traverse the graph: evaluate edge conditions against `{user, event, action}` and route accordingly.

**Design:**
- At a node, on a fired `trigger_action`, collect outgoing edges with that action (or `any`), evaluate `condition` JsonLogic in `priority` order, follow the first match; `decision` nodes evaluate immediately with no UI; `action` nodes fire webhook/event then continue; `terminal` ends the flow.
- Published `graph_snapshot` (Suggestion 4 `flow_versions.graph_snapshot`) delivered in the SDK config bundle; traversal is client-side.

**Testing:**
- `Playwright: CTA click on step → goes to branch A; dismiss → recovery modal.`
- `Unit: two edges same action, priority 0 condition false / priority 1 true → follows priority-1 edge.`
- `Unit: max_visits guard prevents infinite re-entry.`
- `Playwright: decision node routes enterprise users past a step (skip).`

#### 9.3 — Graph builder UI (React Flow branching mode)
**What:** Visual branching editor: draw edges, set trigger/condition, switch a flow between linear and graph mode.

**Design:**
- React Flow canvas with typed node palette (step/decision/action/terminal); edge editor sets `trigger_action` + JsonLogic condition + priority.
- Client-side cycle check mirrors the server guard for instant feedback.

**Testing:**
- `E2E: build a 2-branch flow, publish → graph_snapshot persisted; SDK runs both branches.`
- `E2E: attempt to draw a cycle → UI blocks with message.`

---

## Phase 10: A/B Experiments

### Purpose
Add statistically rigorous A/B testing with automatic winner selection — a top differentiator (Chameleon/Appcues) and a clear AI-augmentation target. After this phase, members can run experiments across flow variants and trust the significance results.

### Tasks

#### 10.1 — Experiment schema, variants, sticky bucketing
**What:** `experiments`, `experiment_variants`, `experiment_assignments` (Suggestion 4 relational variants) with deterministic sticky assignment.

**Design:**
- `experiments { status:draft|running|paused|concluded, stat_method:bayesian|frequentist, confidence_level, primary_metric, primary_metric_config jsonb }`.
- `experiment_variants { flow_id, entry_node_id?, is_control, traffic_weight (sum=1.0) }`.
- Sticky bucketing: `variant = weightedBucket(hash(experimentId+endUserId))`; persisted in `experiment_assignments (UNIQUE experiment,end_user)`; recorded as an `experiment_assigned` journey event for auditability (Suggestion 2 decision).

**Testing:**
- `Unit: same user+experiment → same variant across calls (deterministic).`
- `Unit: 10k users with weights 0.5/0.5 → split within ±2%.`
- `Integration: assignment persisted once; second call returns the stored variant.`

#### 10.2 — Stats engine (frequentist + Bayesian) & auto-conclude
**What:** Daily metric recompute and significance test with automatic winner selection + roll-out.

**Design:**
- Worker computes per-variant `participants, conversions, conversion_rate`; frequentist two-proportion z-test → `p_value`; Bayesian Beta-Binomial → `probability_best` + credible intervals. Stored in `proj_experiment_metrics`.
- Auto-conclude when `min_sample_size` reached AND (p<1−confidence OR probability_best≥threshold); set `winner_variant_id`, optionally roll out winner as the live flow and fire `experiment.concluded` webhook.

**Testing:**
- `Unit: known 2×2 conversion table → z-test p-value matches reference within tolerance.`
- `Unit: Bayesian probability_best for clear winner ≈ 1.0.`
- `Integration: experiment hitting sample size + significance → auto-concludes, winner set, webhook fired.`
- `Integration: insufficient sample → stays running.`

#### 10.3 — Experiments UI + results
**What:** Create/monitor experiments and view significance results.

**Design:**
- Wizard: pick variants (flows or graph entry nodes), weights, primary metric, method, confidence. Results view: per-variant rates, p-value/probability-best, significance badge, recommended winner.

**Testing:**
- `E2E: create experiment, seed events, run metrics job → results view shows winner + significance.`
- `E2E: variant weights must sum to 1.0 — UI validation blocks otherwise.`

---

## Phase 11: AI-Native Features & Mobile SDK

### Purpose
Deliver the AI-native advantages that define the product's positioning — natural-language flow authoring and predictive drop-off detection — and extend reach to mobile. These are the differentiators no incumbent fully offers; they come last because they build on the flow schema, event stream, and analytics from all prior phases.

### Tasks

#### 11.1 — Natural-language flow authoring
**What:** `POST /v1/ai/generate-flow` turns a plain-English description into a valid `FlowDefinition` draft.

**Design:**
- Input: `{ prompt, projectContext: { sampleSelectors[], pageUrls[] } }`. AI SDK with structured output constrained to the `flow-schema.json` (JSON Schema 2020-12) → guaranteed-valid `FlowDefinition`; re-validate with Zod before persisting as a draft.
- System prompt enforces WCAG-correct ARIA defaults and resilient selectors. Output is a draft (human reviews/edits before publish).

**Testing:**
- `Integration (mocked LLM): "Walk new users through billing setup" → valid 3-step FlowDefinition that passes Zod + JSON Schema.`
- `Unit: generated output failing schema → repair loop or 422 (never persists invalid).`
- `Integration: generated steps include aria roles/labels (a11y-by-default).`

#### 11.2 — Predictive drop-off detection
**What:** Score in-progress users for activation-abandonment risk and surface a targeted next-step nudge.

**Design:**
- Worker computes per-user features from `journey_events`/`user_flow_states` (dwell, steps stalled, time since last event) → risk score (start with a calibrated heuristic/logistic model; pluggable). High-risk users get a recommended checklist step or help prompt exposed via the SDK config decisioning.
- `GET /v1/flows/:id/at-risk` lists at-risk users for the dashboard.

**Testing:**
- `Unit: user stalled on step 2 for 7 days → high risk; freshly active user → low.`
- `Integration: at-risk endpoint returns ranked users; SDK surfaces the recommended step for a high-risk user.`

#### 11.3 — React Native mobile SDK (parity core)
**What:** Mobile SDK with identity/events/consent + native modal/tooltip/banner renderers.

**Design:**
- `packages/sdk-react-native` shares `@flowbuilder/shared` (schema, JsonLogic, events). Reuses the config bundle + event endpoints. Native accessible components map ARIA semantics to RN accessibility props (`accessibilityRole`, `accessibilityLabel`).
- Selector model adapts to RN: target by `testID`/accessibility label instead of CSS.

**Testing:**
- `Unit: shared segmentation/JsonLogic evaluates identically in RN runtime.`
- `E2E (Detox/Maestro): identify + linear tour renders and emits flow_completed.`
- `Unit: consent gating blocks tracking until granted (parity with web).`

---

## Phase Summary & Dependencies

```
Phase 1: Foundation (monorepo, DB, RLS, partitions) ─── required by everything
    │
Phase 2: Flow Definition Schema & CRUD ─── requires 1
    │
Phase 3: REST API + Auth + OpenAPI ─── requires 2
    │
    ├── Phase 4: SDK core (identity/events/consent) ─── requires 3
    │       │
    │       └── Phase 5: SDK Renderer & Accessible Overlays ─── requires 4
    │               │
    │               └── Phase 6: Targeting (segments/triggers/config) ─── requires 5
    │
    └── Phase 7: Builder UI & Analytics ─── requires 3 (UI), 6 (targeting UI), Phase 4 events (analytics)
            │
Phase 8: A11y Publish-Gate + AI assist + Webhooks ─── requires 5 (render), 7 (builder surfacing)
    │
Phase 9: Conditional Branching (DAG) ─── requires 5 (runner), 6 (JsonLogic), 7 (builder)
    │
Phase 10: A/B Experiments ─── requires 6 (targeting), 9 (variants can be graph entry nodes), 7 (UI)
    │
Phase 11: AI-Native Features + Mobile ─── requires 2 (schema), 6 (events/analytics), 10 (experiment hooks)
```

**Parallelism opportunities:**
- After Phase 3: **Phase 4 (SDK)** and **Phase 7 (Builder UI)** can be developed concurrently by separate tracks (the UI mocks SDK preview until Phase 5 lands).
- Within Phase 8: webhooks (8.3) is independent of the a11y gate (8.1/8.2) and can be built in parallel.
- Phase 11's mobile SDK (11.3) is independent of the AI tasks (11.1/11.2) and can run in parallel once Phase 6 is done.

**Estimated scope: large** (11 phases, 33 tasks; full-stack platform with browser SDK, builder UI, async workers, AI, and mobile).

---

## Definition of Done (per phase)

A phase is complete only when all of the following hold:

1. All tasks in the phase are implemented.
2. All unit tests pass (`turbo run test`).
3. All integration tests pass against Testcontainers Postgres + Redis (`turbo run test:integration`).
4. ESLint and Prettier pass with no errors; `tsc --noEmit` passes across all packages.
5. `docker compose up` builds and starts api + web + worker + postgres + redis with no errors (from Phase 1 onward).
6. The phase's headline capability works end-to-end (demonstrated by the listed E2E/Playwright test).
7. Any new SDK or API config options are documented in the package README.
8. New/changed API endpoints appear correctly in the generated `/openapi.json` (OpenAPI 3.1) and the FlowDefinition schema stays in sync with `packages/shared/flow-schema.json`.
9. Database changes ship as Drizzle migrations; `drizzle-kit` reports no pending diff; new monthly partitions are covered by the partition job.
10. For any phase touching the SDK renderer or builder UI: axe-core reports zero serious/critical violations on the new surfaces (WCAG 2.2 AA / ACT Rules 1.1).
11. For any phase touching tracking: consent gating verified (no storage/network before consent) and tenant isolation verified (RLS test included).
