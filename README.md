# User Onboarding Flow Builder

> Part of the [worlds-biggest-software-project](https://github.com/worlds-biggest-software-project) initiative.
>
> An AI-native, open-source platform for building in-app tours, checklists, tooltips, and activation analytics without paying enterprise SaaS prices or being locked into proprietary stacks.

User Onboarding Flow Builder is a no-code, AI-augmented onboarding platform for SaaS product, growth, and customer success teams who need to drive activation without engineering dependency. It combines a visual flow builder with behavioural triggering, segmentation, and analytics — and uses AI to generate, personalise, and optimise flows from real user behaviour.

---

## Why User Onboarding Flow Builder?

- **Incumbent pricing locks out smaller teams.** Appcues starts at ~$299/month, UserPilot at $249/month, Chameleon at ~$279/month, and Pendo ranges from $15K to $140K+/year — pushing many teams to deploy 3–4 partial tools instead of a single platform.
- **Open-source alternatives are developer-only.** Intro.js is MIT-licensed and free, but provides no analytics, no segmentation, no visual builder, and no targeting.
- **Existing AI offerings are shallow.** Userflow's FlowAI is the leading-edge differentiator in 2026, yet most platforms still rely on manual rule configuration, static flows, and human-driven A/B test analysis.
- **Tooling is fragmented.** Pendo is the only platform that cohesively combines analytics, guidance, and feedback; everywhere else, teams stitch together multiple vendors.
- **Implementation is heavy.** Pendo deployments commonly run 6–12 months; Chameleon has a steep learning curve. Teams want faster time-to-first-flow without sacrificing depth.

---

## Key Features

### Flow Authoring

- Visual no-code builder for tours, checklists, tooltips, hotspots, banners, and surveys
- Live preview and progressive disclosure: start simple, add complexity as needed
- Multi-experience types in a single coordinated flow
- Resource center for in-product help documentation

### Targeting & Personalisation

- Behavioural triggering on user actions, page visits, and custom events
- Advanced segmentation by user properties, plan type, and imported company data
- Personalisation by user variables and segments
- Mobile and web support from a single platform

### Analytics & Experimentation

- Engagement, completion-rate, and performance dashboards
- A/B testing with statistical significance scoring
- Funnel and path analysis for understanding user workflows
- Session replay for qualitative context to quantitative analytics

### Integration & Extensibility

- REST API for flow management, analytics querying, and data sync
- JavaScript SDK for event tracking and programmatic flow control
- Webhooks for form submissions and user data sync
- Native integrations with analytics, CRM, and messaging platforms

### Compliance & Accessibility

- WCAG 2.1 / 2.2 keyboard navigation and screen-reader compatibility
- GDPR / CCPA-aware behavioural and session data handling
- Content Security Policy (CSP) compatible overlay injection

---

## AI-Native Advantage

AI is positioned to do what current platforms cannot: analyse real session recordings and click patterns to auto-generate the most effective onboarding path per user segment, branch flows dynamically based on inferred user role or company size, and convert plain-English product-manager descriptions into working tour steps, trigger conditions, and copy. Predictive drop-off detection surfaces the right checklist step or chat prompt to users about to churn before activation, and continuous A/B testing automatically selects winners and rolls them out — without a data analyst in the loop.

---

## Tech Stack & Deployment

The platform is designed around a JavaScript SDK plus REST API model, the same integration surface used by every commercial incumbent. Behavioural events can be consumed via OpenTelemetry, and overlay injection respects host-application CSP headers — a frequent integration hurdle on enterprise deployments. Deployment targets cover both web and mobile via native SDKs.

---

## Market Context

The broader product adoption / digital adoption platform (DAP) market is growing rapidly alongside Product-Led Growth adoption: Pendo alone is valued at over $2.6B, and SAP acquired WalkMe for $1.5B in 2023. Pricing tiers run roughly $69–$249/month for SMB tools, $249–$500/month for mid-market, and $15K–$140K+/year for enterprise platforms — typically billed on monthly active users. Primary buyers are SaaS product managers, customer success teams, and growth teams running activation experiments.

---

## Project Status

> This project is in the **research and specification phase**.  
> Contributions, feedback, and domain expertise are welcome.

---

## Contributing

We welcome contributions from developers, domain experts, and potential users.
See [CONTRIBUTING.md](CONTRIBUTING.md) for guidelines.

**Important:** All contributions must be your own original work or clearly attributed
open-source material with a compatible licence. Copyright infringement and licence
violations will not be tolerated and will result in immediate removal of the offending
contribution. If you are unsure whether a piece of code, text, or other material is
safe to contribute, open an issue and ask before submitting.

---

## Licence

Licence to be determined. See [discussion](#) for context.
