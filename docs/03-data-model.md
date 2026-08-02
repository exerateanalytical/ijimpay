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
| `adjustments` | platform | Ops corrections (dual-control only) |

### Canonical entries

**Charge succeeds (5 000 gross, 100 fee):**
```
debit  provider_float:mtn        5000
credit merchant_pending          4900
credit platform_fees              100
```
**Settlement matures:** `debit merchant_pending / credit merchant_available`.
**Payout of 250 000:** `debit merchant_payout_wallet 250000 / credit provider_float:orange 250000` (posted on provider success; a `payout_inflight` clearing account holds it while processing).

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
