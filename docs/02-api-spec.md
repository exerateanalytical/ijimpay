# IjimPay — API Specification (v1 Draft)

Base URL: `https://api.ijimpay.com/v1` · Format: JSON · Auth: Bearer secret keys.
This document defines the contract; the OpenAPI 3.1 file will be generated from it and kept in `openapi/`.

---

## 1. Conventions

- **Auth**: `Authorization: Bearer sk_test_...` or `sk_live_...`. Publishable keys (`pk_...`) only for the checkout widget (create checkout sessions, nothing else).
- **Idempotency**: all POST money operations require `Idempotency-Key` header (UUID, stored 24h). Replay returns the original response.
- **Amounts**: integer XAF (no decimals). `"amount": 5000` = 5 000 FCFA.
- **Phones**: E.164 `2376XXXXXXXX`.
- **Pagination**: cursor-based — `?limit=20&starting_after=<id>`; responses include `has_more`.
- **Versioning**: path version `/v1`; breaking changes → `/v2`. Additive changes are non-breaking.
- **Errors**: HTTP status + problem body:

```json
{
  "error": {
    "code": "insufficient_payer_funds",
    "message": "The payer's mobile money balance is insufficient.",
    "doc_url": "https://docs.ijimpay.com/errors#insufficient_payer_funds",
    "request_id": "req_8fK2..."
  }
}
```

Stable error codes (initial set): `invalid_request`, `authentication_failed`, `permission_denied`, `rate_limited`, `idempotency_conflict`, `channel_unavailable`, `payer_not_found`, `insufficient_payer_funds`, `payer_rejected`, `payer_timeout`, `insufficient_balance`, `payout_limit_exceeded`, `provider_error`.

Additional codes for the dashboard/app/public surfaces: `otp_invalid`, `otp_expired`, `totp_invalid`, `step_up_required` (action needs a fresh 2FA elevation — see `POST /auth/step_up`), `charge_not_cancelable` (charge already terminal), `resend_limit_reached` (public resend allowed once per charge), `link_inactive`, `invite_expired`, `last_owner` (cannot remove/downgrade the last Owner), `settlement_account_cooldown` (change pending 24 h hold), `export_expired` (signed download URL past 72 h).

- **Internal ops API**: the ops console's internal surface lives at `https://api.ijimpay.com/internal/v1/*` and is specified separately in `docs/17-internal-api.md`; it is not part of this contract.
- **Auth surfaces**: §2.9–2.14 (`/auth`, `/me`, `/merchant`, team, developer tooling, app support) are the first-party dashboard/mobile-app surface — session-token auth (cookie on web, refresh/access tokens in the app), not API keys. §2.15 (`/v1/public/*`) is unauthenticated/session-scoped for payers on `pay.ijimpay.com`. Everything else uses secret keys as above.

## 2. Resources & Endpoints

### 2.1 Charges (collections)

```
POST   /charges              create a charge (async)
GET    /charges/{id}         retrieve
GET    /charges              list (filter: status, channel, created[gte|lte], customer_phone, payment_link)
POST   /charges/{id}/refund  refund full/partial (where supported)
POST   /charges/{id}/cancel  cancel a still-pending charge (merchant-side resend path) → 409 charge_not_cancelable if terminal
POST   /charges/{id}/receipt/send   send/re-send the receipt (SMS/WhatsApp/email per notification prefs)
GET    /receipts/{charge_id}.pdf    receipt PDF for a succeeded charge
```

Create request:

```json
{
  "amount": 5000,
  "currency": "XAF",
  "channel": "mtn_momo",          // or "orange_money" | "auto"
  "customer": { "phone": "237670000000", "name": "Ama N." },
  "reference": "ORDER-1042",       // merchant's own ref, unique per merchant
  "description": "Order #1042",
  "metadata": { "cart_id": "abc" },
  "callback_url": "https://shop.example/webhooks/ijimpay"  // optional override
}
```

Response `202 Accepted`:

```json
{
  "id": "ch_01J9XK...",
  "object": "charge",
  "status": "pending",             // pending | succeeded | failed | expired | refunded
  "amount": 5000, "fee": 100, "net": 4900,
  "channel": "mtn_momo",
  "provider_ref": "a1b2c3-...",
  "failure_code": null,
  "expires_at": "2026-08-02T12:10:00Z",
  "created_at": "2026-08-02T12:05:00Z"
}
```

State machine: `pending → succeeded | failed(failure_code) | expired`. Terminal states are final; refunds create a linked `refund` object, never mutate the charge amount.

### 2.2 Checkout Sessions & Payment Links

```
POST /checkout_sessions      one-time hosted page for a cart (e-commerce redirect flow)
GET  /checkout_sessions/{id}
POST /payment_links          reusable or single-use link
GET  /payment_links / {id}   list, retrieve
PATCH /payment_links/{id}    edit (title, amount, catalog fields, expiry) and re-activate (active: true)
POST /payment_links/{id}/deactivate
GET  /payment_links/{id}/stats   views, charges created/succeeded, conversion, volume
```

Payment link create: `{ "title": "Gâteau d'anniversaire", "amount": 15000, "amount_type": "fixed" | "open", "reusable": true, "expires_at": null }` → returns `url` (`https://pay.ijimpay.com/l/abc123`) and `qr_png_url`.

Optional catalog fields (product-style links, quantity picker on the hosted page): `product_name`, `image_url`, `unit_price`, `max_quantity`. Open-amount links (`amount_type: "open"`) accept `min_amount`.

### 2.3 Payouts (disbursements)

```
POST /payouts                single payout
POST /payout_batches         bulk (array of items or uploaded file token)
GET  /payout_batches/{id}    batch with per-item statuses
POST /payout_batches/{id}/approve   (maker–checker; requires approver role)
GET  /payouts/{id} , /payouts
```

Payout item: `{ "amount": 250000, "channel": "orange_money", "beneficiary": { "phone": "237690000000", "name": "J. Fotso" }, "reference": "SAL-2026-07-jfotso" }`. Batch lifecycle: `draft → pending_approval → processing → completed | partially_failed`.

### 2.4 Subscriptions

```
POST /plans                  { name, amount, interval: "week"|"month"|"year" }
POST /subscriptions          { plan, customer: {phone, name}, start_date? }
GET  /subscriptions/{id}, /subscriptions
POST /subscriptions/{id}/cancel | /pause | /resume
GET  /subscriptions/{id}/invoices
```

Each cycle generates an `invoice` → charge attempts per retry ladder → invoice `paid | uncollectible`; subscription `active → past_due → canceled` after N failed cycles.

### 2.5 Balances, Ledger & Settlements

```
GET /balance                          { available: [...], pending: [...] } per currency
GET /balance_transactions             ledger lines visible to merchant (charge, fee, payout, adjustment)
GET /settlements , /settlements/{id}  T+1 transfers to merchant's own account, with included transactions
POST /topups                          fund payout wallet (instructions + auto-match)
```

### 2.6 Customers (light CRM)

```
POST/GET /customers            phone-keyed; auto-created from charges
```

### 2.7 Webhooks & Events

```
POST/GET/DELETE /webhook_endpoints     { url, enabled_events: ["charge.succeeded", ...] }
GET /events , /events/{id}             immutable event log, 90-day retention
POST /webhook_endpoints/{id}/ping
```

Event types (initial): `charge.succeeded`, `charge.failed`, `charge.expired`, `refund.succeeded`, `payout.succeeded`, `payout.failed`, `payout_batch.completed`, `invoice.paid`, `invoice.payment_failed`, `subscription.past_due`, `subscription.canceled`, `settlement.paid`, `balance.topup.received`.

Delivery: POST JSON `{ id, type, created, data: { object } }` with headers:

```
X-Ijimpay-Signature: t=1722600000,v1=hex(hmac_sha256(secret, t + "." + body))
```

Reject if |now − t| > 5 min (replay protection). Retries: 1m, 5m, 30m, 2h, 6h, then every 12h up to 72h. Endpoint auto-disabled after 7 days of failures (email + push warning first).

### 2.8 Misc

```
GET /channels        provider health/availability (mtn_momo: operational | degraded | down)
GET /events/verify   SDK helper endpoints as needed
```

## 3. Test Mode

- Separate key space (`sk_test_`), separate data, identical API surface.
- **Magic phone numbers** on the simulated provider:
  - `237670000001` → succeeds after 5 s
  - `237670000002` → fails `insufficient_payer_funds`
  - `237670000003` → payer rejects
  - `237670000004` → stays pending until expiry
- Test webhooks fire identically; dashboard has a "send test event" button.

## 4. Rate Limits

Per key: 50 req/s sustained, burst 200 (429 + `Retry-After`). Charge creation additionally limited per customer phone (anti-spam of USSD pushes): default 3 pending charges per payer phone per merchant.

## 5. SDK Plan

| SDK | Priority | Notes |
|---|---|---|
| JS/TS (`@ijimpay/node`, `@ijimpay/checkout` widget) | Launch | Generated from OpenAPI + hand-written webhook verify |
| PHP (`ijimpay/ijimpay-php`) | Launch | WooCommerce plugin depends on it |
| Python | Fast-follow | Odoo module depends on it |
| Dart/Flutter client (internal) | With mobile app | Same OpenAPI generation used by our own app |
