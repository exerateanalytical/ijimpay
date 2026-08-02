# IjimPay — Delivery Roadmap

Team assumption: 2–3 backend, 1–2 frontend, 2 Flutter (from Phase 2), 1 designer, 1 product/ops. Adjust durations proportionally.

---

## Phase 0 — Foundations & Paperwork (weeks 0–4, overlaps Phase 1)

**Engineering**
- Monorepo setup (pnpm workspaces): `apps/api` (NestJS), `apps/dashboard` (Next.js), `apps/checkout` (Next.js), `packages/sdk-node`, `openapi/`.
- CI/CD (GitHub Actions), environments (dev/staging/prod), Terraform, observability baseline.
- MTN MoMo sandbox + Orange sandbox credentials; spike each API end-to-end; document real behaviors (latency, callback reliability) in `docs/providers/`.

**Business (critical path — starts day 1)**
- Legal counsel engaged; partner-license strategy chosen.
- MTN & Orange aggregator commercial discussions opened.
- Company KYC pack prepared for telco contracts.

**Exit criteria**: one successful sandbox charge + payout on both providers from our code; deploy pipeline green.

## Phase 1 — Core Rails (weeks 3–12) → *Private sandbox launch*

- Merchants, users, roles, API keys (test mode only at first).
- **Ledger** (accounts, journal entries, postings, invariants) — built before any live money.
- Charges API with MTN + Orange adapters, simulated provider, status pollers, expiry jobs.
- Events + outbox + webhook delivery with retries; `/balance`, `/balance_transactions`.
- Minimal dashboard: login, transactions, API keys, webhook logs.
- OpenAPI published; docs site quickstart; JS/TS SDK generated.

**Exit criteria**: an external pilot developer integrates and charges (sandbox) in < 30 min unaided. Internal load + chaos tests pass.

## Phase 2 — Get Paid Everywhere (weeks 10–20) → *Live pilot with 5–10 merchants*

- Live mode behind KYB (Tier 1 flow, ops review queue).
- Payment links + hosted checkout + QR; checkout widget; WooCommerce plugin + PHP SDK.
- Settlements (T+1) to merchant accounts; statements; reconciliation importer v1.
- **Mobile app v1.0** (charge at counter, links, transactions, push) — Flutter work starts week 10 against sandbox; Play Store closed track for pilots.
- Ops console v1 (KYB queue, transaction search, discrepancy queue).

**Exit criteria**: pilot merchants collecting real money daily; month-end reconciliation clean; app crash-free rate > 99.5%.

## Phase 3 — Money Out (weeks 18–28) → *Public launch*

- Payouts: single, batches, CSV upload, maker–checker, payout wallet + top-ups.
- Payroll: saved beneficiaries, scheduled runs, reports.
- Mobile app v1.1 (payout approvals, team, settlements).
- Refunds; instant-settlement option; rate cards per merchant.
- Marketing site, self-serve signup open, support processes (WhatsApp + email), status page.

**Exit criteria**: a real payroll run for an external company; public signup live.

## Phase 4 — Recurring & Ecosystem (weeks 26–40)

- Subscription engine: plans, invoices, retry ladder, dunning via SMS/WhatsApp links.
- Shopify app, Odoo connector, Python SDK.
- Mobile app v1.2 (catalog, analytics, thermal printer); iOS release.
- Analytics dashboards; ClickHouse if needed.
- Own PSP license process progressing; add next rails as partnerships allow (bank transfer, cards via partner, possibly Wave/regional expansion scoping — Chad, Gabon, CAR share the CEMAC framework).

## Cross-Cutting Workstreams (continuous)

| Workstream | Cadence |
|---|---|
| Reconciliation accuracy | Every phase adds automation; monthly clean close is a release gate |
| Security | Pen test before Phase 3 public launch; quarterly dependency/infra review |
| Regulatory | Monthly legal checkpoint; license application milestones tracked like features |
| Docs & DX | Docs updated in the same PR as any API change — enforced in review |
| Pilot feedback | Weekly merchant interviews during Phases 2–3 |

## Top Risks & Mitigations

| Risk | Impact | Mitigation |
|---|---|---|
| Telco production access delayed (contracts) | Blocks live launch | Start negotiations week 0; partner-PSP umbrella as plan B; build against sandbox + simulator meanwhile |
| Callback/API unreliability from providers | Stuck transactions, merchant distrust | Poller-first design; synthetic monitoring; aggressive timeouts + clear merchant messaging |
| Regulatory surprise | Rework or halt | Counsel engaged pre-code; conservative fund-flow design (segregated accounts) |
| Reconciliation drift | Financial loss, audit failure | Ledger-first, nightly invariants, discrepancy queue from Phase 2 |
| Low-end device performance (app/checkout) | Lost conversions | Perf budgets in CI; test on real 2 GB-RAM devices from day 1 |
| Fraud (fake merchants, payout abuse) | Losses, telco penalties | KYB tiers, velocity rules, maker–checker, payout wallet pre-funding |

## Immediate Next Steps (this month)

1. Approve these documents (this PR) → they become the spec of record.
2. Kick off telco + legal tracks (Phase 0 business items).
3. Scaffold the monorepo (`apps/api`, `apps/dashboard`, `apps/checkout`, `openapi/`) and the NestJS skeleton with the ledger module first.
4. Register sandbox accounts: momodeveloper.mtn.com and Orange Developer.
5. Design kickoff: dashboard + mobile app design system (shared tokens), FR-first copywriting.
