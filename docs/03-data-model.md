# IjimPay — Data Model & Ledger Design

PostgreSQL 16. All money columns `BIGINT` minor units (XAF = whole francs). All tables have `id` (ULID), `created_at`, `updated_at`. Provider payloads stored raw in `JSONB` for audit/debug.

---

## 1. Entity Overview

```
merchants ─┬─ users (via memberships+roles)
           ├─ api_keys
           ├─ webhook_endpoints ── webhook_deliveries
           ├─ customers ── charges ── refunds
           ├─ payment_links / checkout_sessions
           ├─ plans ── subscriptions ── invoices ── (charges)
           ├─ payouts / payout_batches ── payout_items
           ├─ settlements
           └─ ledger: accounts ── postings ── journal_entries
providers (mtn_momo, orange_money) ── provider_transactions ── reconciliation_items
events (immutable) · audit_logs (append-only) · kyb_documents
ops: staff_users ── staff_webauthn_credentials · approval_requests (dual-control)
     risk_rules ── risk_flags ── risk_cases ── risk_case_notes ── str_packs
     merchant_limits ── merchant_limits_history · merchant_notes · ops_settings
```

## 2. Core Tables (abridged)

### merchants
`id, name, legal_name, tier (0|1|2), status (pending|active|suspended), country='CM', default_settlement_channel, settlement_schedule, kyb_status, risk_flags jsonb`

### users / memberships
`users(id, phone, email, password_hash, totp_secret, locale)` ·
`memberships(user_id, merchant_id, role: owner|admin|developer|finance|viewer)`

### api_keys
`id, merchant_id, mode (test|live), kind (secret|publishable), prefix, hash, last_used_at, revoked_at` — store only a hash; show full key once.

### charges
```
id, merchant_id, mode, customer_id, amount, fee, net, currency,
channel (mtn_momo|orange_money), status (pending|succeeded|failed|expired|refunded),
failure_code, reference (unique per merchant+mode), description, metadata jsonb,
provider_txn_id FK, payment_link_id?, checkout_session_id?, invoice_id?,
idempotency_key, expires_at, succeeded_at
```
Index: `(merchant_id, created_at desc)`, `(merchant_id, reference)` unique, `(status) where status='pending'` partial for pollers.

### provider_transactions
```
id, provider (mtn_momo|orange_money), direction (collection|disbursement),
provider_ref uuid, our_ref, amount, phone, status, raw_request jsonb, raw_response jsonb,
last_polled_at, poll_count
```
One row per provider call chain — the audit trail of what we actually sent/received.

### payouts / payout_batches / payout_items
Batch: `status (draft|pending_approval|processing|completed|partially_failed), created_by, approved_by, approval_required bool, totals`. Item: `amount, channel, beneficiary_phone, beneficiary_name, status, failure_code, provider_txn_id`.

### plans / subscriptions / invoices
Subscription: `status (active|past_due|paused|canceled), current_period_start/end, retry_state jsonb (attempt #, next_attempt_at)`. Invoice: `amount, status (open|paid|uncollectible), attempt_count`.

### payment_links / checkout_sessions
Link: `slug unique, title, amount, amount_type (fixed|open), reusable, active, expires_at, qr_object_key`. Session: cart snapshot, `success_url, cancel_url, status`.

### webhook_endpoints / webhook_deliveries / events
`events(id, merchant_id, type, payload jsonb)` immutable. `webhook_deliveries(event_id, endpoint_id, attempt, status_code, next_retry_at, delivered_at)`.

### Outbox
`outbox(id, aggregate, aggregate_id, event_type, payload, processed_at)` — written **in the same DB transaction** as the state change; a relay publishes to the queue. Guarantees no event is ever lost between DB and queue.

## 2b. Ops Console Tables (docs/13)

### approval_requests
```
id, kind ('adjustment'|'limit_change'|'suspension'|'circuit_breaker'|'kyb_tier2'),
subject_type, subject_id,       -- e.g. ('merchant','m_...'), ('provider','orange')
payload jsonb,                  -- the full proposed change, applied atomically on approval
requested_by (staff FK), approver (staff FK, must ≠ requested_by),
status (pending|approved|rejected|expired|canceled), expires_at, decided_at
```
Single dual-control queue: nothing in `payload` is applied until a second staff member approves (WebAuthn re-auth). Expiry enforced by sweeper (SLA 4 h default).

### risk_rules / risk_flags / risk_cases
`risk_rules(key unique, params jsonb, enabled)` — velocity, amount, pattern rules.
`risk_flags(rule_key, target_type ('merchant'|'charge'), target_id, score, status (open|dismissed|escalated))`.
`risk_cases(merchant_id, status (open|closed|str_filed), opened_by, closed_at)` with `risk_case_notes` — append-only, `created_by`, no UPDATE/DELETE grants.

### str_packs
`id, risk_case_id, file_key, generated_by, filed_at, filed_by, anif_reference` — suspicious-transaction-report bundles; marking filed is manual (ANIF acknowledgement number required).

### staff_users / staff_webauthn_credentials
`staff_users(id, email, name, role (ops|ops-admin|finance-admin|compliance|viewer), idp_subject, status)` ·
`staff_webauthn_credentials(staff_id, credential_id, public_key, sign_count, last_used_at)` — SSO + hardware-key step-up for sensitive actions.

### merchant_limits / merchant_limits_history
`merchant_limits(merchant_id unique, per_txn_max, daily_max, monthly_max, payout_daily_max, source ('tier_default'|'custom'))`. Every change writes a `merchant_limits_history` row (old/new jsonb, changed_by, approval_request_id) — changes above tier defaults go through dual control.

### ops_settings
`key unique, value jsonb, updated_by` — queue SLAs, security policy, notification routing; every write also lands in `audit_logs` (old → new value).

### Append-only notes pattern
`merchant_notes`, `risk_case_notes` (and recon resolution notes): `id, subject FK, body, created_by, created_at` — INSERT-only, no UPDATE/DELETE grants; corrections are new notes.

## 3. Ledger (double-entry) — the heart of the system

### Tables

```sql
-- Chart of accounts. One row per (owner, type, currency).
accounts (
  id, owner_type ('merchant'|'platform'|'provider'),
  owner_id,               -- merchant_id, or NULL for platform accounts
  type,                   -- see chart below
  currency 'XAF',
  UNIQUE (owner_type, owner_id, type, currency)
)

journal_entries (
  id, kind,               -- 'charge_succeeded','payout_sent','fee','settlement','adjustment','refund','topup'
  reference_type, reference_id,   -- e.g. ('charge','ch_...') — links entry to business object
  description, created_by, created_at
)                          -- APPEND-ONLY: no UPDATE/DELETE grants

postings (
  id, journal_entry_id, account_id,
  amount BIGINT,          -- signed: positive = credit, negative = debit
  CHECK enforced by trigger: SUM(amount) per journal_entry = 0
)
```

### Chart of accounts (initial)

| Account | Owner | Meaning |
|---|---|---|
| `provider_float:mtn` / `provider_float:orange` | platform | Money we hold at each telco |
| `merchant_available` | merchant | Settleable balance |
| `merchant_pending` | merchant | Succeeded but inside settlement hold |
| `merchant_payout_wallet` | merchant | Pre-funded wallet for disbursements |
| `platform_fees` | platform | Our revenue |
| `settlement_clearing` | platform | In-flight settlements |
| `payout_inflight` | platform | Clearing account for payouts between initiation and provider confirmation |
| `adjustments` | platform | Ops corrections (dual-control only) |

### Canonical entries

**Charge succeeds (5 000 gross, 100 fee):**
```
debit  provider_float:mtn        5000
credit merchant_pending          4900
credit platform_fees              100
```
**Settlement matures:** `debit merchant_pending / credit merchant_available`.
**Payout of 250 000:** initiation `debit merchant_payout_wallet 250000 / credit payout_inflight 250000`; on provider success `debit payout_inflight 250000 / credit provider_float:orange 250000` (reversed back to the wallet on failure).

### Rules

1. Postings are written **in the same transaction** as the charge/payout status change.
2. Balances are `SUM(postings.amount)` per account — materialized into `account_balances` (refreshed transactionally via trigger) for fast reads, but the sum is always the truth.
3. Never mutate: corrections are new reversing entries.
4. Nightly job asserts: every account sums correctly, every journal entry balances, `provider_float` matches telco statement (→ `reconciliation_items` queue on mismatch).

## 4. Reconciliation

```
provider_reports (provider, period, file_key, imported_at)
reconciliation_items (report_id, provider_ref, amount, matched_charge_id?,
                      status: matched|missing_ours|missing_theirs|amount_mismatch,
                      resolved_by, resolution_note)
```
Matcher runs on import: exact match on provider_ref, fallback fuzzy (phone+amount+time window). Unmatched items page ops.

## 5. Multi-tenancy & Data Safety

- Every business table carries `merchant_id`; all queries scoped via repository layer (and optionally Postgres RLS as defense-in-depth).
- Test/live separation by `mode` column + key mode enforcement at the gateway — a `sk_test_` key can never read live rows.
- PII (customer phones/names) encrypted at rest (pgcrypto or app-level envelope encryption); audit_logs append-only.
- Retention: events 90 days hot; charges/ledger forever (regulatory); provider raw payloads 2 years then cold storage.
