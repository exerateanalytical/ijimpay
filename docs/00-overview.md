# IjimPay — Product Overview

## Vision

IjimPay is an API-first payment aggregator for Cameroon that lets any business accept MTN Mobile Money and Orange Money — on their website, in their ERP or POS, through payment links, in a mobile app — and move money out (payroll, supplier payments, withdrawals) through one integration.

**One API. One dashboard. One mobile app. Every mobile money rail in Cameroon.**

## Problems We Solve

1. **Fragmented integrations** — accepting both MTN MoMo and Orange Money today means two contracts, two APIs, two reconciliation processes.
2. **No developer experience** — telco APIs are poorly documented, unreliable, and lack sandbox/test tooling.
3. **No tooling for non-developers** — small merchants can't integrate APIs; they need payment links, QR codes, and a mobile app.
4. **Payouts are manual** — businesses pay salaries and suppliers by hand, one MoMo transfer at a time.
5. **Reconciliation pain** — matching telco statements to sales is manual and error-prone.

## Target Customers (Personas)

| Persona | Needs | Primary surface |
|---|---|---|
| **Developer / SaaS** | Clean REST API, webhooks, test mode, SDKs | API + docs |
| **E-commerce shop** | Checkout plugin (WooCommerce, Shopify), hosted checkout | Plugins + dashboard |
| **Small merchant / informal seller** | Payment links, QR at counter, phone-based POS | **Mobile app** |
| **Enterprise / ERP user** | ERP connector (Odoo, SAP-lite, custom), bulk payouts, approvals | API + dashboard |
| **Subscription platform** | Recurring billing, dunning, retries | API + dashboard |
| **Employer** | Bulk salary disbursement with approval workflow | Dashboard + mobile app (approvals) |

## Product Surfaces

1. **Public REST API** (`api.ijimpay.com/v1`) — the foundation; everything else is a client of it.
2. **Merchant Dashboard** (web) — transactions, settlements, payment links, payouts, team, developer tools.
3. **Merchant Mobile App** (Android-first, iOS second) — see `04-mobile-app.md`. Acts as mini-POS, payment-link creator, notification center, and payout approver.
4. **Hosted Checkout + Payment Links** — customer-facing payment pages, optimized for low-end Android on 3G.
5. **Plugins & SDKs** — WooCommerce, Shopify, Odoo; JS/TS, PHP, Python SDKs; Android POS SDK.

## Revenue Model

- Per-transaction fee on collections (e.g. 1.5–2.5% capped, negotiable at volume).
- Flat fee per disbursement.
- Optional monthly plans for premium features (subscription engine, ERP connectors, dedicated support).
- Float income where regulation permits.

## Success Metrics (first 12 months)

- Time-to-first-successful-charge for a new developer: **< 30 minutes** (sandbox).
- Payment success rate ≥ telco baseline; pending-transaction resolution < 3 minutes p95.
- 100+ active merchants; ≥ 60% weekly-active via mobile app or dashboard.
- Zero ledger-vs-telco unexplained discrepancies at month-end.

## Documentation Map

| Doc | Contents |
|---|---|
| `ARCHITECTURE.md` (repo root) | System architecture, stack choices, deployment |
| `01-product-requirements.md` | Detailed functional requirements per module |
| `02-api-spec.md` | API design: resources, endpoints, auth, webhooks, errors |
| `03-data-model.md` | Database schema and ledger design |
| `04-mobile-app.md` | Mobile app product spec + technical plan |
| `05-security-compliance.md` | Security controls, KYC/AML, CEMAC regulatory path |
| `06-roadmap.md` | Phased delivery plan with milestones |
