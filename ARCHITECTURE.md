# IjimPay — Architecture & Technology Blueprint

**A payment aggregator for Cameroon (MTN Mobile Money + Orange Money), API-first, serving websites, e-commerce, POS, ERP integrations, subscription billing, payment links, and payroll/disbursements.**

---

## 1. Product Scope

| Capability | Description |
|---|---|
| **Collections API** | Accept MoMo/OM payments via REST API (charge, status, refund) |
| **Payment Links** | No-code hosted checkout pages shareable via WhatsApp/SMS/social |
| **Hosted Checkout** | Embeddable widget + redirect checkout for e-commerce |
| **Subscriptions** | Recurring billing engine (MoMo has no native card-on-file, so this is scheduled debit requests + retry logic) |
| **Disbursements / Payouts** | Bulk payouts: salaries, supplier payments, withdrawals |
| **POS / ERP connectors** | SDKs + plugins (WooCommerce, Shopify app, Odoo, custom ERP via API) |
| **Merchant Dashboard** | Transactions, settlement, reconciliation, team management, API keys |
| **Webhooks** | Reliable event delivery to merchant systems |

## 2. Core Design Principles

1. **API-first**: every feature exists as a documented REST API before any UI. Dashboard and checkout are just API clients.
2. **Ledger-first**: an immutable double-entry ledger is the source of truth for all money movement. Never store balances as mutable columns.
3. **Idempotency everywhere**: every money-moving endpoint requires an `Idempotency-Key`. Telecom APIs time out constantly; retries must be safe.
4. **Asynchronous by default**: MTN MoMo and Orange Money APIs are slow (user must approve on phone, 30–120s). All charges are async: `PENDING → SUCCESSFUL/FAILED` with webhooks + polling fallback.
5. **Provider abstraction**: a single internal `PaymentProvider` interface; MTN, Orange (and later banks, cards, Wave, etc.) are adapters. Adding a provider must not touch core logic.
6. **Reconciliation is a first-class subsystem**, not an afterthought. Telco reports and your ledger WILL disagree; automate detection.

## 3. Recommended Stack

### Language & Framework — **TypeScript on Node.js with NestJS** (primary recommendation)

Why:
- Payment aggregation is I/O-bound (HTTP calls to telcos, webhooks, DB) — Node's async model fits perfectly; you don't need CPU-heavy compute.
- TypeScript gives type safety across API contracts, shared with frontend and SDKs (one language for backend, dashboard, checkout widget, and the merchant JS SDK).
- NestJS provides structure (modules, DI, guards, interceptors), first-class OpenAPI generation, queues (BullMQ), and scheduling out of the box.
- Largest hiring pool in the region and globally; fast iteration.

Credible alternatives:
- **Go (chi/echo + sqlc)** — better raw performance and deployment simplicity (single binary). Choose if the team is Go-strong. Slightly slower iteration for product features.
- **Elixir/Phoenix** — excellent for this domain (fault tolerance, queues), but small hiring pool in Cameroon.
- Avoid PHP/Laravel-only or Django-monolith for the core money engine unless team constraints demand it; both are workable but weaker on typed contracts and long-running async workers.

### Data & Infrastructure

| Concern | Choice | Notes |
|---|---|---|
| Primary DB | **PostgreSQL 16** | ACID, row locking for ledger, JSONB for provider payloads. One DB until real scale. |
| Ledger | Double-entry tables in Postgres (`accounts`, `journal_entries`, `postings`) | Consider [TigerBeetle] later; Postgres is fine to millions of tx/month. |
| Queue / async jobs | **Redis + BullMQ** | Charge polling, webhook delivery w/ exponential backoff, payout batches. Upgrade path: NATS/Kafka. |
| Cache / rate limiting | Redis | API key rate limits, session cache. |
| Object storage | S3-compatible | Settlement reports, KYC documents. |
| Search/analytics (later) | ClickHouse or Postgres read replica | Merchant analytics dashboards. |
| Secrets | Vault or cloud KMS | Telco API credentials, webhook signing keys. |

### Deployment

- **Start**: Docker Compose → a single cloud VM or managed containers (Fly.io, Render, AWS ECS/Lightsail). Don't start with Kubernetes.
- **Scale**: move to Kubernetes (EKS/GKE) or ECS when you have a team to operate it.
- **Region**: eu-west (Paris) or af-south (Cape Town) for latency to Cameroon; check CEMAC/data-residency requirements (COBAC, BEAC regulations for PSPs).
- CI/CD: GitHub Actions; IaC: Terraform.
- Observability: OpenTelemetry + Grafana/Loki/Tempo or a hosted APM. **Alert on provider error rates and pending-transaction age** — that's how you detect telco outages before merchants call you.

### Frontend

- **Dashboard**: Next.js (React) + TypeScript, talking only to the public/partner API.
- **Hosted checkout & payment links**: separate lightweight Next.js app — must load fast on 3G Android phones. Aggressive perf budget, French + English i18n.
- **Checkout widget**: framework-free embeddable JS (<30KB) merchants drop into any site.

## 4. System Architecture

**Recommendation: a modular monolith first, not microservices.** One NestJS codebase with strictly separated modules and one Postgres DB. Split services only when scale or team size forces it. Fintech startups die from reconciliation bugs and slow iteration, not from monoliths.

```
                        ┌────────────────────────────────────────┐
  Merchant site/ERP ───▶│  API Gateway (auth, rate limit, HMAC)  │
  Payment link user ───▶│                                        │
  Dashboard         ───▶└───────────────┬────────────────────────┘
                                        │
        ┌───────────────────────────────┴───────────────────────────┐
        │                    Core Application (NestJS)              │
        │  ┌──────────┐ ┌──────────┐ ┌───────────┐ ┌─────────────┐  │
        │  │ Payments │ │ Payouts  │ │Subscription│ │Payment Links│  │
        │  └────┬─────┘ └────┬─────┘ └─────┬─────┘ └──────┬──────┘  │
        │  ┌────┴────────────┴─────────────┴──────────────┴──────┐  │
        │  │            Ledger (double-entry, append-only)       │  │
        │  └────┬────────────────────────────────────────────────┘  │
        │  ┌────┴─────┐ ┌───────────┐ ┌──────────┐ ┌─────────────┐  │
        │  │ Provider │ │ Webhooks  │ │ Merchants│ │ Risk / KYC  │  │
        │  │ Adapters │ │ (outbound)│ │ & API keys│ │& Compliance │  │
        │  └────┬─────┘ └───────────┘ └──────────┘ └─────────────┘  │
        └───────┼───────────────────────────────────────────────────┘
                │                          Workers (BullMQ):
        ┌───────┴────────┐                 - charge status pollers
        │ MTN MoMo API   │                 - webhook delivery + retries
        │ Orange Money   │                 - subscription scheduler
        │ (future: banks,│                 - payout batch processor
        │  cards, Wave)  │                 - reconciliation jobs
        └────────────────┘
```

### Provider adapter interface (the key abstraction)

```ts
interface PaymentProvider {
  requestToPay(req: ChargeRequest): Promise<ProviderRef>;      // async init
  getTransactionStatus(ref: ProviderRef): Promise<TxStatus>;   // poll
  transfer(req: PayoutRequest): Promise<ProviderRef>;          // disbursement
  getBalance(): Promise<Money>;
  verifyCallback(payload: unknown, sig: string): CallbackEvent; // if provider pushes
}
```

Provider notes:
- **MTN MoMo**: official Open API (sandbox at momodeveloper.mtn.com) — Collections, Disbursements, Remittances products. OAuth per product, `X-Reference-Id` (UUID) is your idempotency handle. Callbacks are unreliable → always run a status poller.
- **Orange Money**: Web Payment API (checkout redirect) and, for aggregators, the Orange Money API via Orange Developer / local Orange Cameroon partnership. Expect a commercial contract and IP allowlisting.
- Both require becoming a licensed partner/aggregator locally — start commercial discussions with MTN Cameroon and Orange Cameroon early; sandbox behavior differs materially from production.

### Transaction state machine

`CREATED → PENDING_PROVIDER → (SUCCESSFUL | FAILED | EXPIRED)` with a timeout job that expires stuck transactions and a poller that reconciles PENDING ones every 15–30s with backoff. Ledger postings happen only on terminal states, atomically with the state change.

### Money handling rules

- Store amounts as **integer minor units** (XAF has no decimals — store whole francs as `BIGINT`), never floats.
- Currency always explicit (`XAF`).
- Fees computed and posted as separate ledger lines (merchant payable = gross − fees).
- Settlement: merchant balance accrues in ledger; payouts to merchant MoMo/bank accounts on schedule (T+1) or on demand.

## 5. API Design

- REST + JSON, versioned path (`/v1/...`), OpenAPI spec as the contract; generate SDKs (JS/TS, PHP, Python) from it.
- Auth: `Bearer sk_live_...` / `sk_test_...` secret keys; publishable keys for the widget. Full test mode with a fake-provider adapter so merchants integrate without real money.
- Webhooks: HMAC-SHA256 signed (`X-Ijimpay-Signature` with timestamp to stop replays), retried with exponential backoff for 72h, event log visible in dashboard.
- Errors: RFC 7807-style problem JSON with stable error codes.
- Rate limiting per API key; strict input validation (zod/class-validator).

Example flow (charge):

```
POST /v1/charges  {amount: 5000, currency: "XAF", channel: "mtn_momo", phone: "2376XXXXXXX"}
→ 202 {id: "ch_...", status: "pending"}
... customer approves on phone ...
→ webhook: charge.succeeded  (+ GET /v1/charges/ch_... for polling)
```

## 6. Security & Compliance (non-negotiable)

- **Regulatory**: PSP/aggregator activity in Cameroon falls under BEAC/COBAC regulation (Règlement CEMAC on payment services). Options: obtain a Payment Service Provider license, or operate under a partner bank/telco umbrella initially. Engage a local fintech lawyer before launch.
- KYC/KYB for merchants (registration docs, ID, bank/MoMo account ownership); tiered limits.
- AML: velocity rules, sanctions screening, suspicious-activity flags; transaction monitoring from day one.
- Secrets in KMS/Vault, TLS everywhere, encrypted PII at rest, full audit log (append-only) of admin actions.
- 2FA on dashboard; role-based access for merchant teams.
- No card data initially → no PCI-DSS scope; if cards come later, use a tokenizing partner.

## 7. Build Order (pragmatic roadmap)

1. **Phase 1 – Core rails (8–12 wks)**: ledger, merchants/API keys, MTN + Orange collection adapters, charges API, webhooks, status pollers, test mode, minimal dashboard.
2. **Phase 2 – Distribution**: payment links + hosted checkout, WooCommerce plugin, JS widget, settlement payouts to merchants.
3. **Phase 3 – Money out**: disbursements API, bulk payroll (CSV + API), approval workflows.
4. **Phase 4 – Recurring & ecosystem**: subscription engine (scheduled debits + smart retries + SMS/WhatsApp dunning), Odoo/ERP connectors, POS SDK (Android), reconciliation automation, analytics.

## 8. Summary of Recommendations

| Decision | Choice |
|---|---|
| Architecture | Modular monolith + workers, ledger-first, provider adapters |
| Backend | **TypeScript / Node.js / NestJS** (alt: Go) |
| Database | PostgreSQL (+ Redis, BullMQ) |
| Frontend | Next.js dashboard + ultra-light hosted checkout |
| API style | REST, OpenAPI-driven, idempotent, webhook-based async |
| Deployment | Docker on managed containers → K8s later; Paris/Cape Town region |
| Compliance | Double-entry ledger, KYC/AML, CEMAC PSP licensing path |
