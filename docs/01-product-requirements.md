# IjimPay — Product Requirements (PRD)

Status: Draft v1 · Owner: Product · Last updated: 2026-08-02

Requirement levels: **MUST** (MVP), **SHOULD** (fast-follow), **MAY** (later).

---

## 1. Merchant Onboarding & Accounts

- MUST: self-serve signup (email or phone + OTP), business profile, KYB document upload (RCCM/registration, ID of owner, proof of MoMo/OM/bank account ownership).
- MUST: tiered verification — Tier 0 (sandbox only), Tier 1 (limited live volume), Tier 2 (full limits) — with clear status in dashboard/app.
- MUST: team members with roles: Owner, Admin, Developer, Finance, Viewer.
- MUST: API keys per environment (test/live), secret + publishable, rotation, last-used tracking.
- SHOULD: sub-accounts / multiple business profiles under one login.
- MAY: marketplace/platform accounts (split payments to sub-merchants).

## 2. Collections (Charges)

- MUST: create charge via API with channel `mtn_momo` or `orange_money`, phone number, amount (XAF integer), reference, metadata.
- MUST: async lifecycle `pending → succeeded | failed | expired`; USSD/approval push to the payer's phone; configurable expiry (default 5 min).
- MUST: status polling endpoint + signed webhooks for terminal states.
- MUST: idempotency keys on all create operations.
- MUST: full test mode with simulated provider (magic phone numbers to force success/failure/timeout).
- SHOULD: automatic channel detection from phone prefix (65x/66x/67x/68x/69x mapping), with override.
- SHOULD: refunds (full/partial) where provider supports; otherwise tracked manual refund flow.
- MAY: card and bank rails via future partners.

## 3. Payment Links & Hosted Checkout

- MUST: create a link in < 30 seconds (dashboard or mobile app): amount fixed or open, title, description, expiry, single-use or reusable.
- MUST: hosted payment page — French/English, < 3s load on 3G Android, phone-number entry, live status (approve-on-phone screen), receipt.
- MUST: QR code for every link (print at counter, show on phone screen).
- MUST: share to WhatsApp/SMS with prefilled message.
- SHOULD: product catalog links (name, image, price), quantity selection.
- SHOULD: embeddable checkout widget (JS, < 30 KB) and redirect checkout for e-commerce.
- MAY: invoice links (itemized, due date, partial payment).

## 4. Disbursements (Payouts)

- MUST: single payout API (to MoMo/OM number) and bulk payouts (API + CSV upload in dashboard).
- MUST: maker–checker approval workflow (Finance creates, Admin/Owner approves) with configurable thresholds.
- MUST: payout batches with per-item status, retry of failed items, downloadable report.
- MUST: balance checks and pre-funding model (merchant funds a payout wallet from collections balance or external deposit).
- SHOULD: scheduled/recurring payroll (monthly run, saved beneficiary list with named employees).
- SHOULD: beneficiary directory with verified names (name-check via provider where available).
- MAY: payouts to bank accounts via partner bank.

## 5. Subscriptions & Recurring Billing

Mobile money has no card-on-file; recurring = scheduled charge requests the customer approves each cycle, so dunning is core, not an edge case.

- MUST: plans (amount, interval, currency), subscriptions (customer phone + plan), lifecycle `active → past_due → canceled`.
- MUST: scheduler that fires charge attempts on the billing date; smart retry ladder (e.g. T+0, +6h, +24h, +72h) at times of day with historically high approval rates.
- MUST: dunning notifications to the customer (SMS/WhatsApp deep link to a payment link as fallback).
- SHOULD: proration-free simple model first; pause/resume; coupon codes.
- MAY: usage-based billing.

## 6. Ledger, Balances & Settlement

- MUST: double-entry ledger; every fee, charge, payout, reversal is a balanced journal entry (see `03-data-model.md`).
- MUST: merchant available vs pending balance; settlement schedule T+1 (configurable) to merchant's MoMo/OM/bank.
- MUST: exportable statements (CSV/PDF) per period, per rail.
- MUST: automated reconciliation jobs against provider reports; discrepancy queue for ops.
- SHOULD: instant settlement (fee-based) option.

## 7. Webhooks & Developer Experience

- MUST: webhook endpoints per merchant (test/live), HMAC-SHA256 signatures with timestamp, retries with exponential backoff up to 72h, event log + manual redelivery in dashboard.
- MUST: OpenAPI 3.1 spec published; docs site with quickstarts (charge in 5 minutes), recipes per persona.
- MUST: SDKs: JS/TS and PHP at launch; Python next.
- SHOULD: WooCommerce plugin at launch; Shopify app and Odoo module fast-follow.
- MAY: Android POS SDK for third-party POS vendors.

## 8. Dashboard (Web)

- MUST: transactions list with filters/search/export; transaction detail with full timeline (provider refs, webhook attempts).
- MUST: home with today's volume, success rate, balance; payment links manager; payouts + approvals; team & roles; developer section (keys, webhooks, logs).
- SHOULD: analytics (volume by channel/day, success-rate trends, top customers).
- SHOULD: in-dashboard notifications + email digests.

## 9. Mobile App (summary — full spec in `04-mobile-app.md`)

- MUST: accept a payment at the counter (enter amount → customer pays by QR/link/push), instant sound + push notification on success, today's totals, payment link creation/sharing, payout approvals, balance view.
- Android-first; French/English; usable offline-tolerant (queues and retries on flaky data).

## 10. Notifications

- MUST: push (mobile app), email; SMS for critical merchant events (large payout approved, settlement sent).
- SHOULD: WhatsApp Business API for receipts and dunning.

## 11. Admin (internal ops console)

- MUST: merchant review/KYB approval queue, transaction search across merchants, discrepancy/reconciliation queue, provider health dashboard, manual ledger adjustment with dual control + audit trail.
- MUST: risk flags (velocity, unusual patterns) surfaced for review.

## 12. Non-Functional Requirements

| Area | Requirement |
|---|---|
| Availability | 99.9% API uptime target; graceful degradation per provider (MTN down ≠ Orange down) |
| Latency | API p95 < 300 ms (excluding provider wait, which is async) |
| Durability | No money event ever lost: outbox pattern, at-least-once webhook delivery, append-only ledger |
| Scale target Y1 | 500k transactions/month headroom on single Postgres |
| Localization | FR + EN everywhere customer- or merchant-facing |
| Accessibility | Hosted checkout usable on Android 8+/2G-3G, low-bandwidth mode |
| Auditability | Every admin and money action in append-only audit log |
