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
GET  /payout_batches         list (filter: status, created[gte|lte])
GET  /payout_batches/{id}    batch with per-item statuses
POST /payout_batches/validate       server-side pre-check of items/CSV rows (no batch created) → per-row errors (invalid phone, non-integer amount, amount < 100, duplicate reference, unknown channel prefix, missing name)
POST /payout_batches/{id}/approve   (maker–checker; requires approver role, ≠ creator)
POST /payout_batches/{id}/reject    checker rejection { "reason": "..." } → batch back to draft, reason surfaced
GET  /payout_batches/{id}/report    CSV/PDF result report (per-item outcome)
POST /payouts/{id}/retry            retry a failed item (Idempotency-Key required)
GET  /payouts/{id} , /payouts
```

Payout item: `{ "amount": 250000, "channel": "orange_money", "beneficiary": { "phone": "237690000000", "name": "J. Fotso" }, "reference": "SAL-2026-07-jfotso" }`. Batch lifecycle: `draft → pending_approval → processing → completed | partially_failed`.

#### 2.3.1 Beneficiaries & payroll lists

```
GET/POST /beneficiaries , PATCH/DELETE /beneficiaries/{id}   saved payees { phone, name, channel } (operator name-check where available)
GET/POST /payroll_lists , GET/PATCH/DELETE /payroll_lists/{id}   named, reusable lists of beneficiary+amount rows
POST /payroll_lists/{id}/schedule    { cadence, next_run_date } — at run date the system creates a draft batch, checks funding, moves it to pending_approval
```

#### 2.3.2 Fees & internal transfers

```
GET  /fees                   fee schedule per channel/direction (collection & payout), for review screens
POST /balance_transfers      instant internal transfer available balance → payout wallet { amount }
```

### 2.4 Subscriptions

```
POST /plans                  { name, amount, interval: "week"|"month"|"year" }
GET  /plans                  list
PATCH /plans/{id}            edit (name; amount changes apply to future cycles)
POST /plans/{id}/archive     archive — running subscriptions continue, no new signups
POST /subscriptions          { plan, customer: {phone, name}, start_date? }
GET  /subscriptions/{id}, /subscriptions   (filter: status, customer)
POST /subscriptions/{id}/cancel | /pause | /resume
GET  /subscriptions/{id}/invoices
POST /invoices/{id}/send_link       generate + send a single-use dunning payment link (SMS/WhatsApp) for an open invoice
POST /invoices/send_links           bulk dunning { invoice_ids: [...] } — max 1 manual dunning / 24 h / customer
```

Each cycle generates an `invoice` → charge attempts per retry ladder → invoice `paid | uncollectible`; subscription `active → past_due → canceled` after N failed cycles.

### 2.5 Balances, Ledger & Settlements

```
GET /balance                          { available: [...], pending: [...], payout_wallet: [...] } per currency
GET /balance_transactions             ledger lines visible to merchant (charge, fee, payout, adjustment)
GET /settlements , /settlements/{id}  T+1 transfers to merchant's own account, with included transactions
GET /settlements/{id}/statement.pdf   settlement statement (PDF)
GET /settlements/{id}/statement.csv   settlement statement (CSV)
POST /topups                          fund payout wallet (instructions + auto-match)
```

### 2.6 Customers (light CRM)

```
POST/GET /customers            phone-keyed; auto-created from charges
GET  /customers/{id}           retrieve (profile + totals)
PATCH /customers/{id}          edit name/metadata
POST /customers/{id}/notes     append an internal note (append-only)
```

### 2.7 Webhooks & Events

```
POST/GET/DELETE /webhook_endpoints     { url, enabled_events: ["charge.succeeded", ...] }
PATCH /webhook_endpoints/{id}          edit url / enabled_events
POST /webhook_endpoints/{id}/enable    re-enable an auto-disabled endpoint
POST /webhook_endpoints/{id}/roll_secret   new signing secret (revealed once)
GET  /webhook_endpoints/{id}/deliveries    delivery log (event, HTTP code, attempt #, next_retry_at)
POST /webhook_deliveries/{id}/redeliver    manual redelivery of one delivery
GET /events , /events/{id}             immutable event log, 90-day retention
POST /webhook_endpoints/{id}/ping
```

Event types (initial): `charge.succeeded`, `charge.failed`, `charge.expired`, `refund.succeeded`, `payout.succeeded`, `payout.failed`, `payout_batch.completed`, `payout_batch.pending_approval` (maker–checker: batch awaits an approver), `payout_batch.partially_failed` (batch finished with ≥ 1 failed item), `invoice.paid`, `invoice.payment_failed`, `subscription.past_due`, `subscription.canceled`, `settlement.paid`, `balance.topup.received`, `kyb.decision` (KYB approved/rejected, with tier), `security.new_device` (new device/session on the account), `team.member_changed` (member added, role changed, or removed).

The last five types double as mobile push notification triggers (routing per docs/15's push table).

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

### 2.9 Auth & sessions (dashboard + app)

First-party surface: session cookie (web) / refresh + access tokens (app). Not available to API keys.

```
POST /auth/signup/otp        start signup — send OTP to phone (or email on web)
POST /auth/signup            complete signup with verified OTP + password
POST /auth/otp               send a login/verification OTP (rate-limited)
POST /auth/otp/verify        verify an OTP code
POST /auth/login             phone/email + password → session (or 2FA challenge)
POST /auth/login/totp        complete login with TOTP code
POST /auth/password/forgot   start password reset (OTP/email)
POST /auth/password/reset    set new password with reset token
POST /auth/recovery          account recovery with identity re-verification (lost phone/2FA)
POST /auth/step_up           re-prompt 2FA for sensitive actions → elevated token (5 min validity); actions requiring it fail step_up_required
```

App-only:

```
POST /auth/pin/set           set/replace app PIN (payout approvals locked 24 h after a PIN reset)
POST /auth/token/refresh     rotate access token
POST /auth/logout            revoke current session
```

### 2.10 Current user (`/me`)

```
GET/PATCH /me                profile (name, phone, email, locale)
POST /me/password            change password → revokes other sessions + security email
POST /me/totp · POST /me/totp/verify · DELETE /me/totp   TOTP enrollment / verify / disable (disable forbidden for Owner/Admin at Tier ≥ 1)
GET /me/sessions · DELETE /me/sessions/{id}              active sessions, revoke one
GET /me/devices · DELETE /me/devices/{id}                enrolled devices, revoke one
POST /devices                app: register device + FCM push token
POST /devices/{id}/enroll    app: enroll device for approvals (PIN/biometric)
GET /me/notifications        in-product notification feed (type, title, body, read state, linked object)
POST /me/notifications/mark_all_read
GET/PATCH /me/notification_preferences   per-event channel toggles (push/email/SMS/WhatsApp)
GET /me/memberships          businesses this user belongs to (role per membership)
POST /me/switch              switch active business context { membership_id }
DELETE /me/memberships/{id}  leave a business (forbidden for its last Owner → last_owner)
```

(App path aliases `GET /notifications`, `POST /notifications/mark_all_read`, `GET /sessions`, `GET /devices` resolve to the `/me/...` endpoints above — one canonical set.)

### 2.11 Merchant profile & KYB

```
GET/PATCH /merchant          business settings incl. payout approval threshold (threshold change requires step-up 2FA)
PUT  /merchant/profile       resumable onboarding draft (business info, saved per step)
POST /merchant/logo          upload/replace logo (multipart)
PATCH /merchant/settlement_account   change settlement account — takes effect after 24 h cooldown (settlement_account_cooldown while pending)
GET/POST /merchant/kyb_documents     list docs (type, status sent/approved/rejected + reason) · upload/replace a doc
POST /merchant/kyb/submit    submit dossier for review
POST /merchant/kyb/request_tier      request Tier 2 → returns list of additional required docs
GET/PUT /merchant/golive_checklist   persisted go-live checklist state
```

### 2.12 Team & invitations

```
GET /members · PATCH /members/{id} · DELETE /members/{id}   list, change role, remove (Owner required to touch an Admin; last Owner protected)
GET/POST /invites            list, invite { phone/email, role } — expire after 7 days
POST /invites/{id}/resend    resend (60 s cooldown) · DELETE /invites/{id}  cancel
GET  /invites/{token}        invitee-side: resolve invite (merchant, role, inviter)
POST /invites/{token}/accept | /decline
```

### 2.13 Developer tooling

```
GET/POST /api_keys · GET /api_keys/{id} · DELETE /api_keys/{id}   list/create/retrieve/revoke (secret revealed once at creation)
POST /api_keys/{id}/rotate   rotate with grace window (0 / 24 h / 72 h — old key auto-revoked after)
GET /api_logs · GET /api_logs/{id}   recent API request log (method, path, status, latency, key prefix, redacted bodies)
POST /test_events            fire a sample event at an endpoint (test mode "send test event")
```

### 2.14 Platform & app support

```
GET /app/config              app bootstrap: min supported version, maintenance flag, USSD codes per channel, FAQ manifest, feature flags
GET /channels/{channel}/history      provider status history (uptime/degradations)
POST /support/tickets        create a support ticket { subject, body, attachments? }
GET /stats/volume            aggregated charge volume time-series (period, granularity, stacked by channel) for dashboards
GET /search?q=               federated search (charges, customers, links, payouts, batches) — grouped results, max 5 per group
POST /exports · GET /exports/{id}    async export jobs (statements/CSV; ≤ 10 000 rows returns a direct URL, larger jobs email a signed URL expiring after 72 h → export_expired)
```

### 2.15 Public payer surface (`pay.ijimpay.com`)

Unauthenticated / checkout-session-scoped endpoints consumed by the hosted pages and widget. Server-brokered — no secret key ever reaches the browser; responses expose only payer-safe fields.

```
GET  /v1/public/links/{slug}             resolve a payment link landing: title, amount config (incl. min_amount), catalog fields, merchant display profile, accepted channels, support contact, state (active | expired | paid)
GET  /v1/public/checkout_sessions/{id}   resolve a checkout session (cart snapshot, urls, state)
POST /v1/public/charges                  payer-side charge creation from the hosted page (publishable-key/session scoped)
GET  /v1/public/charges/{id}             status poll — status, failure_code, expires_at only
GET  /v1/public/charges/{id}/stream      SSE status stream (text/event-stream; progressive enhancement over polling)
POST /v1/public/charges/{id}/resend      cancel + recreate the pending charge (optionally with a new payer phone); allowed once per charge → resend_limit_reached
GET  /v1/public/receipts/{receipt_id}    receipt resource (mirrors charge: merchant, amount, status, refunds, session items)
GET  /v1/public/receipts/{receipt_id}/pdf   receipt PDF (short alias `GET /r/{receipt_id}.pdf` on pay.ijimpay.com)
```

Merchant public display profile (embedded in link/session resolutions): name, logo, `support_phone`, `support_whatsapp`, `fee_passthrough` (bool — when true the hosted page shows the "dont frais" line), `accepted_channels[]`.

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
