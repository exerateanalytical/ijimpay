# IjimPay

API-first payment aggregator for Cameroon — accept MTN Mobile Money and Orange Money on websites, e-commerce shops, POS, ERPs and subscription platforms; send money out for payroll and supplier payments. Payment links, hosted checkout, merchant dashboard, and a merchant mobile app.

**Status: documentation & planning phase.** No application code yet — the documents below are the spec of record.

## Documents

| Doc | Contents |
|---|---|
| [ARCHITECTURE.md](ARCHITECTURE.md) | System architecture, stack choices, provider adapters, deployment |
| [docs/00-overview.md](docs/00-overview.md) | Vision, personas, product surfaces, revenue model |
| [docs/01-product-requirements.md](docs/01-product-requirements.md) | Functional requirements (MUST/SHOULD/MAY) per module |
| [docs/02-api-spec.md](docs/02-api-spec.md) | API contract: charges, payouts, links, subscriptions, webhooks |
| [docs/03-data-model.md](docs/03-data-model.md) | Database schema and double-entry ledger design |
| [docs/04-mobile-app.md](docs/04-mobile-app.md) | Merchant mobile app spec (Flutter, Android-first) |
| [docs/05-security-compliance.md](docs/05-security-compliance.md) | Security controls, KYC/AML, CEMAC regulatory path |
| [docs/06-roadmap.md](docs/06-roadmap.md) | Phased delivery plan, risks, immediate next steps |
| [docs/07-brand-design-system.md](docs/07-brand-design-system.md) | Brand, colors, typography, Lucide icon map, components, layout, motion |
| [docs/08-web-screens.md](docs/08-web-screens.md) | Dashboard + ops console: every page, action, state; notifications/email matrix |
| [docs/09-checkout-screens.md](docs/09-checkout-screens.md) | Hosted checkout & payment link pages: every screen, state, edge case |
| [docs/10-mobile-screens.md](docs/10-mobile-screens.md) | Mobile app: full screen inventory, flows, offline/error behaviors |
| [docs/11-platform-inventory.md](docs/11-platform-inventory.md) | **Master inventory & index**: exact counts, full sitemap, traceability |
| [docs/12-dashboard-pages-spec.md](docs/12-dashboard-pages-spec.md) | Dashboard: 49 pages, 35 modals — every field, action, state |
| [docs/13-ops-console-spec.md](docs/13-ops-console-spec.md) | Ops console: 24 pages, 14 modals — dual-control, recon, risk |
| [docs/14-checkout-pages-spec.md](docs/14-checkout-pages-spec.md) | Checkout: 22 screens with final FR/EN copy for every state |
| [docs/15-mobile-app-screens-spec.md](docs/15-mobile-app-screens-spec.md) | Mobile app: 50 screens, 19 sheets — gestures, offline, push routing |
| [docs/16-flows-catalog.md](docs/16-flows-catalog.md) | 74 end-to-end flows + complete message catalog (35 notifications) |
| [docs/17-internal-api.md](docs/17-internal-api.md) | Internal ops API: 77 endpoints, dual-control lifecycle, WebAuthn |
| [docs/review/](docs/review/) | Adversarial completeness reviews (gap reports) |

## Stack (decided)

Backend: TypeScript / NestJS · DB: PostgreSQL + Redis/BullMQ · Web: Next.js · Mobile: Flutter (Android-first) · API: REST, OpenAPI 3.1, webhooks.
