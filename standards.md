# Standards & API Reference

> Project: User Onboarding Flow Builder · Generated: 2026-05-03

## Industry Standards & Specifications

### Accessibility Standards

| Standard | Authority | Relevance | URL |
|----------|-----------|-----------|-----|
| WCAG 2.1 / 2.2 | W3C Web Accessibility Initiative | Accessibility guidelines for digital content; onboarding tours and overlays must support keyboard navigation, focus management, and screen readers. Level AA conformance is standard for SaaS. | https://www.w3.org/WAI/WCAG21/Understanding/keyboard-accessible |
| WAI-ARIA | W3C | Accessible Rich Internet Applications specification; defines roles, states, and properties for dynamic UI components (modals, tooltips, dialogs). Essential for accessible overlay and modal implementation. | https://www.w3.org/WAI/ARIA/ |
| ATAG 2.0 | W3C | Authoring Tool Accessibility Guidelines; flow builders should be usable by authors with disabilities (e.g., keyboard-only operation, text alternatives). | https://www.w3.org/WAI/ATAG/overview/ |

### Privacy and Data Standards

| Standard | Authority | Relevance | URL |
|----------|-----------|-----------|-----|
| GDPR (General Data Protection Regulation) | EU | Onboarding platforms track user behavior and session data; requires explicit consent, data residency options, and right to erasure. | https://gdpr-info.eu/ |
| CCPA (California Consumer Privacy Act) | California | Privacy law requiring disclosure of data collection practices and user opt-out rights for behavioural tracking. | https://oag.ca.gov/privacy/ccpa |
| Content Security Policy (CSP) | W3C Specification | Security standard defining which scripts, styles, and resources can execute within a page. Onboarding overlays injected via script tags must comply with host site's CSP headers. | https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP |

### API and Integration Standards

| Standard | Description | Relevance | URL |
|----------|-------------|-----------|-----|
| REST API Design | HTTP/1.1 (RFC 7231) | Industry standard for onboarding platform APIs; all surveyed platforms expose REST APIs accepting JSON. | https://datatracker.ietf.org/doc/html/rfc7231 |
| OpenAPI 3.1 | Specification | Emerging standard for API documentation; enables automated client generation and integration testing. | https://spec.openapis.org/oas/v3.1.0.html |
| OAuth 2.0 | IETF RFC 6749 | Standard for authentication and authorization; enables third-party integrations and SSO. | https://datatracker.ietf.org/doc/rfc6749/ |
| OpenTelemetry | CNCF Project | Emerging standard for emitting product events that onboarding platforms consume to trigger contextual flows. Defines semantic conventions for events. | https://opentelemetry.io/docs/specs/semconv/general/events/ |
| JSON Schema 2020-12 | Meta-schema | Data validation for flow definitions, triggering rules, and segmentation logic. | https://json-schema.org/draft/2020-12/json-schema-validation |

### Product-Led Growth Framework

| Standard | Description | Relevance | URL |
|----------|-------------|-----------|-----|
| PLG Framework | Industry Methodology | Defines activation milestones, time-to-value metrics, and "aha moment" discovery that onboarding flows are designed to drive. Not a formal standard but widely adopted in SaaS. | https://www.productled.com/ |

---

## Similar Products — Developer Documentation & APIs

### Appcues

- **Description:** No-code flow builder for in-app experiences including tours, checklists, and surveys. Fast time-to-publish platform with multi-channel orchestration.
- **API Documentation:** https://docs.appcues.com/dev-api-data
- **Public API:** https://api.appcues.com/v2/docs
- **JavaScript SDK:** https://docs.appcues.com/dev-api-data/javascript-api-developer
- **HTTP API:** https://docs.appcues.com/dev-api-data/http-api-developer
- **SDKs/Libraries:** JavaScript SDK; REST APIs for Python, Go, Java via community libraries
- **Developer Guide:** https://docs.appcues.com/dev-api-data
- **Standards:** REST API with JSON payloads; OpenAPI documentation available.
- **Authentication:** API Key (client ID + secret) for server-side API calls; JWT for webhooks.

### Pendo

- **Description:** Integrated product analytics, in-app guidance, and feedback platform. Auto-captures user interactions without manual event tagging.
- **API Documentation:** https://engageapi.pendo.io/
- **REST API:** https://support.pendo.io/hc/en-us/articles/38099922926875-Pendo-developer-documentation
- **Event API:** https://support.pendo.io/hc/en-us/articles/23335876011803-Events-overview
- **SDKs/Libraries:** JavaScript SDK for web; native SDKs for iOS and Android
- **Developer Guide:** https://support.pendo.io/hc/en-us/articles/38099922926875-Pendo-developer-documentation
- **Standards:** REST API with JSON; custom Track Event format (not CloudEvents).
- **Authentication:** Integration Key (header-based `x-pendo-integration-key`) for REST API access.

### Chameleon

- **Description:** Highly customizable in-app experiences platform with advanced A/B testing and personalization.
- **API Documentation:** https://help.chameleon.io/ (developer docs)
- **REST API:** Available via developer portal
- **JavaScript SDK:** For event tracking and custom integrations
- **SDKs/Libraries:** JavaScript SDK; REST APIs for other languages
- **Developer Guide:** https://help.chameleon.io/
- **Standards:** REST API with JSON; custom event format.
- **Authentication:** API Key for server-side API calls; JWT for webhooks.

### Userflow

- **Description:** AI-powered flow builder with FlowAI auto-generation from product interactions. Lightweight no-code platform.
- **API Documentation:** https://www.userflow.com/api (developer documentation)
- **REST API:** For flow management and analytics querying
- **JavaScript SDK:** For event tracking and custom logic
- **SDKs/Libraries:** JavaScript SDK; REST APIs for other languages
- **Developer Guide:** https://www.userflow.com/docs
- **Standards:** REST API with JSON; custom event format.
- **Authentication:** API Key for server-side access; JavaScript SDK token for client-side.

### Intercom Product Tours

- **Description:** In-app tour feature bundled within Intercom's broader customer communication platform.
- **API Documentation:** https://developers.intercom.com/
- **REST API:** https://developers.intercom.com/building-apps/docs/rest-apis
- **JavaScript SDK:** For event tracking and tour triggering
- **SDKs/Libraries:** JavaScript SDK; REST APIs for Python, Ruby, Go, Node.js
- **Developer Guide:** https://developers.intercom.com/building-apps/docs
- **Standards:** REST API with JSON; Intercom custom event format.
- **Authentication:** OAuth 2.0 for server-to-server; API tokens for direct access.

---

## Notes

### Emerging Standards

1. **CloudEvents for Product Events**: OpenTelemetry adoption is enabling standardized event schemas; opportunity for onboarding platforms to consume CloudEvents-formatted product events for triggering flows.

2. **Accessibility-by-Design**: WCAG 2.1+ compliance is baseline; emerging opportunity for platforms to guide flow builders toward accessible design patterns and auto-validate for a11y.

3. **AI-Native Event Schemas**: As AI flow generation becomes more sophisticated, standardized schemas for recording user intent, product interactions, and optimal flow paths will emerge.

### Compliance and Regulatory Alignment

- **GDPR/CCPA**: Onboarding platforms must support data residency, consent management, and user data export. CCPA requires prominent opt-out for behavioural tracking.
- **CSP Compliance**: Overlay injection via script tags must respect host site's Content Security Policy headers; failure to comply causes script rejection.
- **WCAG 2.1 AA**: Onboarding tours must support keyboard-only navigation, focus management, and screen-reader compatibility. Inaccessible tours exclude users with disabilities and create legal liability.

### Integration Patterns

1. **Event-Driven Triggering**: Flows trigger on user events (page views, button clicks, form submissions) or custom product events (via JavaScript SDK or server-side API).
2. **Segmentation on User Properties**: Flows target based on user attributes (plan, company, sign-up date) or imported CRM data (imported via API or native integrations).
3. **Webhook Delivery for Actions**: Form submissions, survey responses, and engagement events delivered to customer systems via webhooks with HMAC signatures.
4. **Analytics Export**: Engagement, completion, and performance data exported via REST API for BI and analytics platforms.

### Security Best Practices

- **API Key Management**: Separate keys for different environments (dev/prod); rotate keys regularly.
- **OAuth 2.0 for Third-Party Access**: Prefer OAuth 2.0 over shared API keys for multi-tenant SaaS integrations.
- **CSP Compatibility**: Onboarding SDKs must be CSP-compliant; communicate required CSP directives clearly.
- **HTTPS Only**: All API communication must use HTTPS; no unencrypted data transmission.
- **Webhook Signature Verification**: All webhook deliveries include HMAC signatures; verify on receipt.

---

## Recommended Alignment with Standards

For project 302 (User Onboarding Flow Builder):

1. **Accessibility**: Ensure builder UI and generated flows comply with WCAG 2.1 Level AA. Guide builders toward accessible patterns via inline validation.
2. **REST API Design**: Expose OpenAPI 3.1-compliant REST API for flow management, analytics, and integrations.
3. **Authentication**: Implement OAuth 2.0 for third-party integrations; API Keys for direct server-to-server access.
4. **Event Standards**: Support OpenTelemetry-compatible event ingestion for triggering flows; adopt CloudEvents format if applicable.
5. **Privacy Compliance**: Support GDPR data residency, CCPA opt-out, and consent management; provide data export and deletion APIs.
6. **CSP Compliance**: Ensure JavaScript SDK respects host site's Content Security Policy headers; document required directives.
7. **Semantic Consistency**: Follow W3C ARIA roles and properties for accessible overlays and modals.

---

## Sources

- [WCAG 2.1 Web Accessibility Guidelines](https://www.w3.org/WAI/WCAG21/)
- [WAI-ARIA Specification](https://www.w3.org/WAI/ARIA/)
- [OpenTelemetry Specification](https://opentelemetry.io/)
- [Appcues API Documentation](https://docs.appcues.com/dev-api-data)
- [Pendo Developer Documentation](https://support.pendo.io/hc/en-us/articles/38099922926875-Pendo-developer-documentation)
- [Chameleon Help Center](https://help.chameleon.io/)
- [Userflow Developer Docs](https://www.userflow.com/docs)
- [Intercom Developer Hub](https://developers.intercom.com/)
- [OAuth 2.0 Authorization Framework](https://datatracker.ietf.org/doc/rfc6749/)
- [REST API Design Best Practices](https://datatracker.ietf.org/doc/html/rfc7231)
