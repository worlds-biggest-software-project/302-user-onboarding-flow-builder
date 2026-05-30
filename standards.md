# Standards & API Reference

> Project: User Onboarding Flow Builder · Generated: 2026-05-22

---

## Industry Standards & Specifications

### Accessibility Standards

#### WCAG 2.1 / 2.2 — Web Content Accessibility Guidelines
- **Maintaining body:** W3C Web Accessibility Initiative (WAI)
- **URL:** https://www.w3.org/WAI/WCAG22/
- **Key criteria for overlays and tours:**
  - **2.1.1 Keyboard** — all tour steps, tooltips, and checklist items must be operable without a mouse
  - **2.1.2 No Keyboard Trap** — focus must not become trapped in a flow step; users must be able to dismiss and exit
  - **2.4.3 Focus Order** — when a tour step opens, focus must move into it immediately and return to the trigger element on close
  - **2.4.7 Focus Visible** — the focused element inside any overlay must always have a visible focus indicator
  - **2.4.11 / 2.4.12 Focus Not Obscured (WCAG 2.2)** — sticky banners and tooltips must not cover the focused element
  - **1.4.3 Contrast** — overlay text must meet 4.5:1 contrast ratio against background
  - **1.4.11 Non-text Contrast** — close buttons and UI components in overlays must meet 3:1
  - **1.4.13 Content on Hover or Focus** — tooltip content appearing on hover/focus must be dismissable (Escape key), hoverable, and persistent
- **Data model relevance:** Every flow `Step` entity must carry an `a11yAuditResult` field recording WCAG criterion pass/fail status; the builder UI must surface inline warnings.

#### WAI-ARIA 1.3 — Accessible Rich Internet Applications
- **Maintaining body:** W3C
- **URL:** https://w3c.github.io/aria/ (spec), https://www.w3.org/WAI/ARIA/apg/ (patterns)
- **Roles and required attributes for onboarding elements:**

| Element type | ARIA role | Required attributes |
|---|---|---|
| Tour step modal | `dialog` | `aria-modal="true"`, `aria-labelledby` → step title ID, `aria-describedby` → step body ID |
| Blocking alert step | `alertdialog` | `aria-modal="true"`, `aria-labelledby`, `aria-describedby` (pointing to the alert message) |
| Tooltip / hotspot | `tooltip` | `id` on tooltip; trigger element carries `aria-describedby` pointing to that ID |
| Progress notification | `status` | Implicit live region; wraps completion-rate text so screen readers announce updates |
| Checklist container | `list` / `listitem` | Each item carries `aria-checked` (for checkbox items) or `aria-disabled` |
| Dismissal button | `button` | `aria-label="Close"` when no visible text; must be keyboard-reachable |

- **Focus management pattern:** On tour step open → move focus to first interactive element. On close → return focus to element that triggered the step. The entire background page must receive `aria-hidden="true"` or `inert` while a modal dialog is open.
- **Data model relevance:** The `StepConfig` schema must store `ariaRole`, `ariaLabelledBy`, `ariaDescribedBy`, and `focusTarget` fields. The renderer must apply these at paint time.

#### ATAG 2.0 — Authoring Tool Accessibility Guidelines
- **Maintaining body:** W3C WAI
- **URL:** https://www.w3.org/TR/ATAG20/
- **Two-part structure:**
  - **Part A — Accessible authoring UI:** The visual flow builder itself must be keyboard-navigable, screen-reader-compatible, and meet WCAG 2.1 AA. Drag-and-drop canvas operations must have keyboard alternatives.
  - **Part B — Support accessible content production:** The builder must guide authors toward accessible output — e.g., flag missing `alt` text on images, warn when colour contrast fails, prompt for ARIA label input.
- **Conformance levels:** A, AA, AAA (same scale as WCAG).
- **Data model relevance:** A `ContentAudit` record per flow should store ATAG Part B check results, surfaced to the author in the builder before publish.

#### ACT Rules Format 1.1 — Accessibility Conformance Testing
- **Maintaining body:** W3C Accessibility Guidelines Working Group
- **URL:** https://www.w3.org/TR/act-rules-format/ (became W3C Recommendation 2026)
- **Purpose:** Defines a machine-readable format for writing automated accessibility test rules, enabling consistent automated validation across tools. 47 published rules, each reviewed by three independent parties and implemented in at least two tools.
- **Data model relevance:** The platform's a11y validation engine should evaluate generated flows against ACT rule implementations (e.g., Deque axe-core, IBM Equal Access). Store `ActRuleResult { ruleId, outcome: "passed"|"failed"|"inapplicable", impact, element }` per step.

---

### Event & Telemetry Standards

#### OpenTelemetry Semantic Conventions — Events
- **Maintaining body:** Cloud Native Computing Foundation (CNCF), OpenTelemetry project
- **URL:** https://opentelemetry.io/docs/specs/semconv/general/events/ (v1.41.0, August 2025)
- **Core model:** An OTel Event is a specialised `LogRecord` with:
  - `event.name` — unique identifier for the event structure (e.g., `onboarding.step.completed`)
  - `severity_number` — standardised severity
  - Standard resource attributes: `service.name`, `service.version`, `deployment.environment`
  - Custom attributes on the log body representing the event payload
- **Relevant conventions for UI interaction:** `user.id`, `session.id`, browser attributes (`browser.platform`, `browser.user_agent`)
- **Data model relevance:** All product events ingested to trigger flows should be modelled as OTel-compatible log records. The `ProductEvent` entity carries `eventName`, `resourceAttributes` (JSON), `body` (JSON), `timestampUnixNano`.

#### CloudEvents v1.0 — Event Schema Specification
- **Maintaining body:** CNCF CloudEvents Working Group
- **URL:** https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md
- **Required fields:**

| Field | Type | Description |
|---|---|---|
| `id` | String | Unique event ID; combined with `source` must be unique |
| `source` | URI-reference | Context in which the event occurred (e.g., `/accounts/acme/app`) |
| `specversion` | String | Always `"1.0"` |
| `type` | String | Reverse-DNS event type (e.g., `com.yourapp.user.activated`) |

- **Optional fields:** `subject` (sub-resource within source), `time` (RFC 3339 timestamp), `datacontenttype` (`application/json`), `dataschema` (URI to JSON Schema), `data` (event payload)
- **Data model relevance:** Inbound webhook events from customer systems should be validated against the CloudEvents schema. The `InboundEvent` entity maps directly to these fields; `type` and `source` drive flow triggering rules.

---

### Privacy & Consent Standards

#### GDPR Article 6 — Lawful Bases for Processing
- **Maintaining body:** European Parliament / European Data Protection Board (EDPB)
- **URL:** https://gdpr-info.eu/art-6-gdpr/
- **Six lawful bases:** consent, contract, legal obligation, vital interests, public task, legitimate interests (Art. 6(1)(f))
- **Onboarding platform implications:**
  - **Behavioural tracking for analytics** — requires consent (Art. 6(1)(a)) or legitimate interest (Art. 6(1)(f)) with a three-part test: purpose, necessity, balancing
  - **Session recording / replay** — almost certainly requires explicit consent; cannot be justified as legitimate interest alone
  - **Switching bases is prohibited** — if a user withdraws consent, the platform cannot retroactively claim legitimate interest
  - Enforcement precedent: Meta fined €390M for misapplying lawful basis on behavioural advertising
- **Data fields required in the consent management model:** `userId`, `lawfulBasis`, `consentTimestamp`, `consentVersion`, `dataCategories[]`, `withdrawnAt`, `retentionPeriodDays`

#### ePrivacy Directive 2002/58/EC (Cookie Law)
- **Maintaining body:** European Commission (national implementations by member states)
- **URL:** https://gdpr.eu/cookies/
- **Core requirement:** Prior, informed, freely-given consent before setting any non-essential cookie or accessing device storage — including pixels, localStorage, sessionStorage, fingerprinting, and URL-parameter tracking. Strictly necessary cookies (session management) are exempt.
- **2025–2026 enforcement posture:** Intensified enforcement of "prior consent"; rejecting cookies must be as easy as accepting (CNIL fines exceed €100M). Expanded scope covers all tracking technologies, not just HTTP cookies.
- **Data model relevance:** The platform's SDK must not write any tracking storage until the host page's consent signal is received. `ConsentSignal { granted: boolean, categories: string[], timestamp, source: "user"|"gpc" }` must be evaluated before any `identify()` or `track()` call executes.

#### CCPA / CPRA — California Consumer Privacy Act
- **Maintaining body:** California Privacy Protection Agency (CPPA)
- **URL:** https://oag.ca.gov/privacy/ccpa
- **Key requirements for onboarding platforms:**
  - Must honour **Global Privacy Control (GPC)** signals as a valid opt-out
  - Must surface "Do Not Sell or Share My Personal Information" link in product UI or documentation
  - "Sharing" specifically covers cross-context behavioural advertising; opt-out must cease data flows immediately
  - Opt-out decisions must be retained for at least 12 months before re-prompting
  - Must propagate opt-out downstream to all third parties who received the data
- **Data model relevance:** `UserPrivacyPreference { userId, doNotSell: boolean, doNotShare: boolean, gpcSignal: boolean, optOutTimestamp, propagatedToVendors: boolean }`

#### ISO/IEC 27701:2025 — Privacy Information Management System (PIMS)
- **Maintaining body:** ISO / IEC
- **URL:** https://www.iso.org/standard/27701
- **Status:** Revised 2025 as a standalone certifiable standard (no longer requires ISO 27001 as a prerequisite)
- **Structure relevant to a SaaS platform:**
  - **Clause 7 — PII Controller controls:** consent management, purpose limitation, data subject rights (access, erasure, portability)
  - **Clause 8 — PII Processor controls:** sub-processor agreements, data breach notification, processing records
  - **Annex D:** Maps to GDPR requirements — useful for compliance gap analysis
- **Data model relevance:** Requires maintaining a `ProcessingActivityRecord` (Art. 30 GDPR equivalent): `{ activity, purpose, lawfulBasis, dataCategories[], dataSubjectCategories[], retentionPeriod, recipients[], transferMechanisms[] }`.

---

### Content Security Policy

#### W3C CSP Level 3
- **Maintaining body:** W3C WebAppSec Working Group
- **URL:** https://www.w3.org/TR/CSP3/
- **Problem for overlay injection:** An onboarding SDK injected via a `<script>` tag is blocked by a host application's restrictive CSP. The following directives are directly relevant:

| Directive | Purpose | SDK requirement |
|---|---|---|
| `script-src` | Controls which scripts may execute | SDK domain must be whitelisted, or SDK must use nonce (`'nonce-<base64>'`) or hash (`'sha256-<base64>'`) |
| `style-src` | Controls which stylesheets may load | Inline styles injected by the overlay renderer require `'unsafe-inline'` or a nonce/hash |
| `frame-src` | Controls which origins may be framed | If the SDK uses an iframe sandbox, the SDK origin must be listed |
| `frame-ancestors` | Prevents clickjacking on the host | Relevant if the builder's preview pane embeds the host app in an iframe |
| `connect-src` | Controls fetch/XHR destinations | The SDK's API endpoint domain must be listed |

- **Nonce-based compliance pattern:** The SDK should support a `nonce` configuration option; the host application passes its per-request CSP nonce to the SDK initialisation call so the SDK's injected `<script>` and `<style>` elements carry matching nonce attributes.
- **`strict-dynamic`:** When used, trust propagates from a nonced root script to all scripts it dynamically loads — enables the SDK loader pattern without whitelisting every sub-resource.
- **Data model relevance:** `ProjectSettings.cspNonce` (set per-page by the host app), `ProjectSettings.allowedOrigins[]`. Documentation must enumerate all origins the SDK contacts.

---

### API Standards

#### OpenAPI 3.1 / 3.2
- **Maintaining body:** OpenAPI Initiative (Linux Foundation)
- **URL:** https://spec.openapis.org/oas/v3.1.0.html; 3.2.0 released September 2025
- **Key improvement over 3.0:** Full alignment with JSON Schema 2020-12. `type` may now be an array (`["string", "null"]` replaces the `nullable` extension). `$ref` may coexist with sibling keywords. `if/then/else` supported for conditional schemas.
- **3.2 additions:** Structured tag navigation, streaming-friendly media types, fresh OAuth flows.
- **Data model relevance:** The platform REST API must ship an OpenAPI 3.1 spec. All flow definition schemas reuse the same JSON Schema 2020-12 definitions, enabling client SDK generation and contract testing.

#### JSON Schema 2020-12
- **Maintaining body:** JSON Schema Organization
- **URL:** https://json-schema.org/draft/2020-12/json-schema-validation
- **Use for flow definition validation:** The `FlowDefinition` JSON document (steps, triggers, segmentation rules) should be validated against a published JSON Schema. Key vocabulary features: `$defs` for reusable sub-schemas, `if/then/else` for conditional step structures, `unevaluatedProperties: false` for strict object validation, `$dynamicRef` for recursive schemas (e.g., nested conditional branches).

#### OAuth 2.0 + OIDC
- **Maintaining body:** IETF (RFC 6749, RFC 7636 PKCE, RFC 8414 discovery); OpenID Foundation (OIDC Core 1.0)
- **URLs:** https://datatracker.ietf.org/doc/rfc6749/ · https://openid.net/specs/openid-connect-core-1_0.html
- **Recommended flows:**
  - **Authorization Code + PKCE (RFC 7636):** Required for all browser-based and mobile SDK clients. `code_verifier` (random 43–128 chars) → SHA-256 → base64url → `code_challenge`. Replaces Implicit flow entirely.
  - **Client Credentials (RFC 6749 §4.4):** For server-to-server API access (customer backends calling the platform API).
- **OIDC claims relevant to segmentation:** `sub` (user ID), `email`, `name`, `org_id` (custom claim), `plan` (custom claim). The platform should accept OIDC ID tokens from customer identity providers to bootstrap user identity without a separate `identify()` call.
- **Data model relevance:** `OAuthClient { clientId, clientSecret (hashed), grantTypes[], redirectUris[], scopes[], pkceRequired: boolean }`

#### Webhook Signature Standard — HMAC-SHA256
- **De facto standard:** Adopted by Stripe, GitHub, Shopify, Slack, Twilio
- **Reference:** https://webhooks.fyi/security/hmac
- **Mechanism:**
  1. Generate a random per-endpoint secret (min 32 bytes, stored as env var)
  2. On delivery: compute `HMAC-SHA256(secret, rawRequestBodyBytes)`; base64-encode; send as `X-Webhook-Signature: sha256=<value>`
  3. Include `X-Webhook-Timestamp` (Unix epoch seconds); reject if older than 300 seconds (replay protection)
  4. On receipt: re-compute HMAC; compare using constant-time comparison (`crypto.timingSafeEqual`, `hmac.compare_digest`)
- **Critical:** Sign the raw byte body, not the parsed JSON representation.
- **Data model relevance:** `WebhookEndpoint { url, secret (encrypted at rest), signingAlgorithm: "hmac-sha256", timeoutMs, retryPolicy }`. Delivery log stores `{ deliveredAt, statusCode, responseMs, signatureValid }`.

---

### Accessibility Testing

#### ACT Rules Format 1.1 (detailed above under Accessibility Standards)

Key ACT rules relevant to onboarding overlays:

| ACT Rule ID | Description |
|---|---|
| `5f99a7` | `aria-label` on elements with `dialog` role |
| `b5c3f8` | Elements with `dialog` role have an accessible name |
| `3e12e1` | `aria-hidden` on elements not excluding focusable descendants |
| `6cfa84` | `aria-required` fields |
| `afw4f7` | Focusable elements have accessible names |

- The platform's automated publish-gate should run ACT rule checks via axe-core or IBM Equal Access before a flow goes live, storing per-step results.

---

## Similar Products — Developer Documentation & APIs

### Appcues

- **Description:** No-code in-app experience builder for tours, checklists, banners, and surveys. Fast time-to-publish with multi-channel orchestration and NPS.
- **API Documentation:** https://docs.appcues.com/dev-api-data
- **JavaScript API:** https://docs.appcues.com/dev-api-data/javascript-api-developer
- **HTTP REST API:** https://docs.appcues.com/dev-api-data/http-api-developer
- **Standards:** REST / JSON; API Key + client secret auth; JWT-signed webhooks
- **Authentication:** Client ID + secret for server-side API; JavaScript SDK token (write key) for client-side

### Pendo

- **Description:** Integrated product analytics, in-app guidance, and NPS feedback. Autocaptures interactions without manual event tagging; includes session replay.
- **API Documentation:** https://engageapi.pendo.io/
- **Developer Hub:** https://support.pendo.io/hc/en-us/articles/38099922926875-Pendo-developer-documentation
- **Standards:** REST / JSON; custom Track Event format (not CloudEvents); `x-pendo-integration-key` header auth
- **Authentication:** Integration Key (header-based) for REST API; JS snippet for client-side

### Chameleon

- **Description:** Highly customisable in-app experience platform with advanced A/B testing, personalisation, and Salesforce-native segmentation.
- **API Documentation:** https://www.trychameleon.com/developers
- **Standards:** REST / JSON; API Key auth; HMAC webhook signatures
- **Authentication:** API Key (server-side); JavaScript SDK token (client-side)

### Userflow

- **Description:** AI-assisted flow builder (FlowAI) with lightweight SDK, resource centre, and NPS. Generates flows from product interactions.
- **API Documentation:** https://userflow.com/docs/api
- **Developer Docs:** https://userflow.com/docs
- **Standards:** REST / JSON; API Key auth
- **Authentication:** API Key (server-side); `userflow.js` identify token (client-side)

### Intercom Product Tours

- **Description:** Tour feature within the Intercom customer communication suite. Tightly integrated with Intercom's messenger, tickets, and contact CRM.
- **API Documentation:** https://developers.intercom.com/docs/rest-apis
- **Developer Hub:** https://developers.intercom.com/
- **SDKs:** JavaScript, Python, Ruby, Go, Node.js — https://github.com/intercom
- **Standards:** REST / JSON; follows OpenAPI; OAuth 2.0 for app integrations
- **Authentication:** OAuth 2.0 (for marketplace apps); Bearer token (for direct API access)

### WalkMe (SAP)

- **Description:** Enterprise digital adoption platform acquired by SAP (2023, $1.5B). Covers desktop applications, web, and mobile. Governance and analytics-heavy.
- **API Documentation:** https://developer.walkme.com/
- **Standards:** REST / JSON; API Key + OAuth 2.0
- **Authentication:** OAuth 2.0 client credentials for server-side integrations

### UserPilot

- **Description:** No-code product experience platform with onboarding flows, feature adoption analytics, and in-app surveys. Targets PLG SaaS teams.
- **API Documentation:** https://docs.userpilot.com/api
- **Standards:** REST / JSON; API Key auth; webhook support
- **Authentication:** API Key (server-side); `userpilot.js` identify (client-side)

### Intro.js (Open Source)

- **Description:** MIT-licensed JavaScript library for step-by-step product tours and tooltips. No analytics, no segmentation, no visual builder — developer-only.
- **Repository:** https://github.com/usablica/intro.js
- **Documentation:** https://introjs.com/docs
- **Standards:** Pure JS, no API, no auth. Integrates with any framework.
- **Authentication:** None (client library only)

### Shepherd.js (Open Source)

- **Description:** Open-source JavaScript tour library. Actively maintained; supports accessibility hooks and custom positioning.
- **Repository:** https://github.com/shepherd-guide/shepherd
- **Documentation:** https://docs.shepherdjs.dev/
- **Standards:** Pure JS library; supports ARIA attributes in step configuration
- **Authentication:** None (client library only)

---

## Notes

### Standards Gaps and Evolving Areas

1. **No formal standard for flow definition schemas.** There is no W3C or IETF specification for the JSON structure of an onboarding flow. OpenAPI 3.1 + JSON Schema 2020-12 should be used to define and publish the flow definition schema, and it should be versioned and machine-readable for tool interoperability.

2. **CloudEvents adoption is still early for product analytics.** Most incumbents use proprietary event formats. An opportunity exists to be the first platform to offer CloudEvents-compliant event ingestion out of the box, enabling any OTel-instrumented customer app to trigger flows without a custom SDK.

3. **ePrivacy Regulation withdrawal (February 2025).** The proposed ePrivacy Regulation, which would have replaced the 2002 Directive, was withdrawn. The 2002 Directive remains in force, implemented inconsistently across member states. Monitor EDPB guidance for harmonisation.

4. **ISO/IEC 27701:2025 is now standalone.** The 2025 revision allows certification without ISO 27001, lowering the barrier for SMB SaaS vendors. This is worth pursuing as a trust signal for enterprise buyers.

5. **ACT Rules Format 1.1 became a W3C Recommendation in 2026.** This is now the baseline for automated accessibility testing pipelines. Implement axe-core (which implements ACT rules) in the platform's CI/CD publish gate.

6. **WCAG 2.2 Focus Not Obscured (2.4.11 / 2.4.12).** These criteria, added in WCAG 2.2, are specifically relevant to sticky banners and overlays — a core onboarding UI pattern. Many existing tools fail these criteria. Compliance here is a genuine differentiator.

### Recommended Alignment Priorities

| Priority | Standard | Action |
|---|---|---|
| P0 | WAI-ARIA 1.3 | Implement `dialog`, `alertdialog`, `tooltip`, `status` roles on all overlay types |
| P0 | WCAG 2.2 AA | Focus management, keyboard nav, contrast, and 1.4.13 hover/focus for all steps |
| P0 | W3C CSP Level 3 | Support `nonce` config option; document all `connect-src` / `script-src` requirements |
| P0 | GDPR Art. 6 + ePrivacy | Block SDK tracking calls until consent signal received |
| P1 | CCPA / CPRA | Honour GPC signal; propagate opt-out downstream |
| P1 | OpenAPI 3.1 | Publish machine-readable API spec; generate client SDKs from it |
| P1 | HMAC-SHA256 webhooks | Implement per-endpoint secrets + timestamp replay protection |
| P1 | OAuth 2.0 + PKCE | PKCE for browser SDK; client credentials for server-to-server |
| P2 | CloudEvents v1.0 | Accept CloudEvents-formatted inbound events for flow triggering |
| P2 | OpenTelemetry events | Model all internal product events as OTel LogRecord events |
| P2 | JSON Schema 2020-12 | Publish versioned flow definition schema |
| P2 | ATAG 2.0 | Make builder UI keyboard-navigable and guide authors toward accessible output |
| P3 | ACT Rules 1.1 | Integrate axe-core ACT rule validation into publish gate |
| P3 | ISO/IEC 27701:2025 | Adopt as compliance framework; target certification as enterprise trust signal |

---

## Sources

- [WAI-ARIA Dialog Pattern — W3C APG](https://www.w3.org/WAI/ARIA/apg/patterns/dialog-modal/)
- [WAI-ARIA Alertdialog Pattern — W3C APG](https://www.w3.org/WAI/ARIA/apg/patterns/alertdialog/)
- [WAI-ARIA 1.3 Specification (draft)](https://w3c.github.io/aria/)
- [WCAG 2.2 — W3C Recommendation](https://www.w3.org/WAI/WCAG22/)
- [ATAG 2.0 — W3C Recommendation](https://www.w3.org/TR/ATAG20/)
- [ACT Rules Format 1.1 — W3C Recommendation](https://www.w3.org/TR/act-rules-format/)
- [OpenTelemetry Semantic Conventions — Events](https://opentelemetry.io/docs/specs/semconv/general/events/)
- [CloudEvents Specification v1.0](https://github.com/cloudevents/spec/blob/main/cloudevents/spec.md)
- [GDPR Article 6 — lawful bases](https://gdpr-info.eu/art-6-gdpr/)
- [GDPR / ePrivacy — cookies overview](https://gdpr.eu/cookies/)
- [CCPA — California AG](https://oag.ca.gov/privacy/ccpa)
- [ISO/IEC 27701:2025](https://www.iso.org/standard/27701)
- [W3C CSP Level 3](https://www.w3.org/TR/CSP3/)
- [OpenAPI Specification 3.1.0](https://spec.openapis.org/oas/v3.1.0.html)
- [JSON Schema 2020-12 Validation](https://json-schema.org/draft/2020-12/json-schema-validation)
- [OAuth 2.0 — RFC 6749](https://datatracker.ietf.org/doc/rfc6749/)
- [PKCE — RFC 7636](https://datatracker.ietf.org/doc/html/rfc7636)
- [OpenID Connect Core 1.0](https://openid.net/specs/openid-connect-core-1_0.html)
- [HMAC webhook signatures — webhooks.fyi](https://webhooks.fyi/security/hmac)
- [Appcues Developer Docs](https://docs.appcues.com/dev-api-data)
- [Pendo Engage API](https://engageapi.pendo.io/)
- [Intercom Developer Hub](https://developers.intercom.com/)
- [WalkMe Developer Portal](https://developer.walkme.com/)
- [Intro.js GitHub](https://github.com/usablica/intro.js)
- [Shepherd.js Documentation](https://docs.shepherdjs.dev/)
