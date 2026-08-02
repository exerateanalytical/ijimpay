# IjimPay — Internal Ops API Specification (v1 Draft)

Base URL: `https://api.ijimpay.com/internal/v1` · Format: JSON · Consumer: the ops console (`ops.ijimpay.com`, docs/13) **only** — never exposed to merchants or the public internet beyond the staff IP allowlist (O-20 §Sécurité).

This document specifies every endpoint declared in docs/13 §"API additions needed" plus the few the ops pages (O-01…O-24) need but that list omitted. Conventions mirror docs/02; entities per docs/03.

---

## 1. Conventions

- **Auth**: staff SSO session cookie (OIDC via corporate IdP, 8 h lifetime — configurable O-20) obtained through the login flow (§2). No API keys, no Bearer tokens. All requests require the session cookie + CSRF token header `X-Ijimpay-Csrf`.
- **WebAuthn step-up**: endpoints marked **DC** (dual-control) or **step-up** require a fresh WebAuthn assertion passed as `webauthn_assertion` in the request body (validated server-side, ≤ 2 min old). Missing/stale assertion → `401 webauthn_required`.
- **IDs**: ULIDs, prefixed per resource (`rev_`, `apr_`, `adj_`, `rci_`, `rc_`, `strp_`, `stf_`, `note_`…), same style as docs/02 (`ch_`, `po_`).
- **Amounts**: integer XAF, no decimals (docs/02 §1). **Phones**: E.164 `2376XXXXXXXX`.
- **Pagination**: cursor-based — `?limit=20&starting_after=<id>`; responses include `has_more`. All list endpoints paginate this way.
- **Errors**: HTTP status + problem body identical to docs/02 §1:

```json
{
  "error": {
    "code": "approver_is_maker",
    "message": "The requester cannot approve their own request.",
    "doc_url": "https://docs.ijimpay.com/internal-errors#approver_is_maker",
    "request_id": "req_8fK2..."
  }
}
```

- **Error codes**: all docs/02 codes remain valid where applicable (`invalid_request`, `authentication_failed`, `permission_denied`, `rate_limited`…), plus the internal-only set:

| Code | HTTP | Meaning |
|---|---|---|
| `approver_is_maker` | 403 | The approver of a dual-control request is the same staff user as the maker. |
| `approval_request_expired` | 409 | The approval request expired (24 h; adjustments 72 h) before the decision call. |
| `unbalanced_journal_entry` | 422 | Submitted postings do not sum to zero — the entry is rejected before any write. |
| `webauthn_required` | 401 | A fresh WebAuthn assertion is required (step-up) and was missing or stale. |
| `last_ops_admin` | 409 | The action would deactivate or demote the last active ops-admin. |

- **Roles**: `ops`, `ops-admin`, `compliance`, `finance-admin` (docs/13). Each endpoint states the minimum role; unauthorized roles get `403 permission_denied`.
- **Audit**: every mutating call — and every flagged read (PII reveal, document download, exports) — writes `audit_logs` (append-only, docs/03).
- **Live-only**: the console is live-only; test-mode objects are returned read-only with `"mode": "test"` and mutations on them are rejected `invalid_request`.
- **Exports**: CSV export endpoints return the file inline up to 10 000 rows; beyond, they return `202` `{ "delivery": "email" }` and mail a signed link. Every export is audit-logged.

## 2. Auth & Staff (O-01, O-19, O-24, global chrome)

```
GET    /auth/sso/start                  begin OIDC redirect to the corporate IdP (?login_hint=)
POST   /auth/webauthn/verify            second factor after IdP return → 8 h session
POST   /auth/logout                     revoke the current session (topbar "Se déconnecter")
GET    /staff/me                        current staff profile
GET    /staff                           list staff users                      [ops-admin; others: self only]
POST   /staff/invites                   invite a staff member                 [ops-admin]
PATCH  /staff/{id}                      edit name / non-admin role change     [ops-admin]
POST   /staff/{id}/role_change_requests DC — role change to/from ops-admin    [ops-admin]  DC
POST   /staff/{id}/deactivate           deactivate account, revoke sessions   [ops-admin]
POST   /staff/{id}/webauthn/enroll      begin+finish hardware-key enrollment  [self w/ existing key, or ops-admin]
DELETE /staff/{id}/webauthn/credentials/{credId}  revoke a security key       [self, ops-admin]  step-up
GET    /notifications                   bell feed for the current staff user
POST   /notifications/{id}/read         mark a bell notification read
```

- `POST /auth/webauthn/verify` — request `{ assertion }` (from `navigator.credentials.get`); response `{ staff: {...}, session_expires_at }`. Failures: `authentication_failed`; no key enrolled → `403` with FR guidance (O-01 no-key state). Rate-limited (`rate_limited`, retry 5 min).
- `GET /staff/me` — `{ id, name, email, role, webauthn_credentials: [{ id, nickname, enrolled_at, last_used_at }], last_login_at }`.
- `GET /staff` — rows add `status: active|deactivated|invited`, `last_login_at` (O-19 table). Non-ops-admin callers receive only their own row.
- `POST /staff/invites` — `{ email, name, role }`; email must be `@ijimpay.com`, unique (`invalid_request` otherwise). Invitee must enroll a key on first login.
- `POST /staff/{id}/role_change_requests` **DC** — `{ role, reason }`; creates an `approval_request` of type `staff_role_change` (§10). Direct `PATCH` of a role to/from `ops-admin` is rejected `invalid_request` — this endpoint is mandatory for those transitions.
- `POST /staff/{id}/deactivate` — restated confirm client-side; server refuses to deactivate the last active ops-admin → `409 last_ops_admin`. Sessions revoked immediately.

## 3. Queues summary (O-02)

```
GET /queues/summary        counts + worst SLA per queue + "my items"
```

Response: `{ queues: [{ key: "kyb"|"reconciliation"|"risk"|"approvals", count, oldest_at, worst_sla_pct, assigned_count }], my_items: [{ type, id, reference, sla_pct }] }`. Provider health comes from the merchant API `GET /channels` (docs/02 §2.8), not duplicated here.

## 4. KYB queue (O-03, O-04)

```
GET  /kyb/reviews                       list queue (?status=new|in_review|info_requested&tier=&assignee=&sort=sla)
GET  /kyb/reviews/{id}                  full dossier: documents + declared data + history
POST /kyb/reviews/{id}/claim            assign the review to me
POST /kyb/reviews/{id}/assign           reassign to another staff user        [ops-admin]
POST /kyb/reviews/{id}/approve          approve tier (tier 2 → DC)            [ops, compliance]  DC (tier 2)
POST /kyb/reviews/{id}/reject           reject with reason templates          [ops, compliance]
POST /kyb/reviews/{id}/request_info     request more info; SLA paused         [ops, compliance]
GET  /kyb/reviews/export                CSV export of the queue               [ops-admin]
GET  /kyb/documents/{id}/download       5-min signed URL; download audit-logged
```

- Review object: `{ id, merchant: { id, name, legal_name, tier }, requested_tier, status, documents: [{ id, type, file_name, uploaded_at, status, reject_reason }], declared: { legal_name, legal_form, sector, address, owner_id_name, owner_id_number, settlement_channel, settlement_number }, assignee, submitted_at, sla: { target_hours, elapsed_pct, paused } , history: [...] }`.
- `approve` — `{ tier, limits: { standard: true } | { collection_daily, payout_daily, per_transaction, max_tx_daily }, checklist: { legal_name, rccm, owner_identity, settlement_account } }` (all checklist values must be `true` — else `invalid_request`). **Tier 1**: applied immediately, merchant notified. **Tier 2**: the same call creates an `approval_request` type `kyb_tier2` (§10) covering tier **and** limits atomically; response `202` `{ approval_request: {...} }`; nothing applies until second approval (règle DC, F-047).
- `reject` — `{ template_codes: ["doc_illisible", ...], custom_text?, document_ids: [] }` (templates per OM-01). Closes the dossier; rejected documents flip to "resend" merchant-side.
- `request_info` — `{ items: ["rccm"|"owner_id"|"account_proof"|"address_proof"|"other"], message }` (20–1 000 chars). Status → `info_requested`, SLA paused until the merchant uploads again.

## 5. Merchants (O-05, O-06)

```
GET   /merchants                          search/browse (?query=&status=&tier=&flagged=)
GET   /merchants/{id}                     full internal profile (all O-06 tabs' header data)
GET   /merchants/export                   CSV export                          [ops-admin, finance-admin]
GET   /merchants/{id}/balances            ledger balances from postings
GET   /merchants/{id}/limits              current limits + consumed today + change history
PATCH /merchants/{id}/limits              apply a non-DC limit change (any decrease; increase ≤ 20 % at tier ≤ 1)  [ops-admin]
POST  /merchants/{id}/limit_change_requests  DC — limit change (tier > 1 or increase > 20 %)  [ops-admin]  DC
POST  /merchants/{id}/suspension_requests    DC — suspend or reactivate       [ops-admin]  DC
POST  /merchants/{id}/impersonate         30-min read-only dashboard session  [ops-admin]  step-up
POST  /merchants/{id}/notes               append-only internal note
GET   /merchants/{id}/api-logs            recent API calls ≥ 400 (O-06 tab 8)
```

- `GET /merchants` row: `{ id, name, legal_name, tier, status, volume_30d, available_balance, risk_flags, created_at }`.
- `GET /merchants/{id}` embeds per-tab data: profile (docs/03 merchants fields), verification (`kyb_status`, documents, decision history), team (`memberships`, API-key metadata — prefix/mode/last_used_at only, **never** the key), webhook endpoints + failing deliveries, active DC banners (`pending_approvals: [...]`), suspension record (`{ suspended_at, scope, maker, checker, reason }`). Activity uses `GET /transactions?merchant_id=` (§6); journal entries use `GET /ledger/journal_entries?merchant_id=` (§9).
- `GET /merchants/{id}/balances` — `{ available, pending, payout_wallet }`, each `SUM(postings)` on `merchant_available` / `merchant_pending` / `merchant_payout_wallet` (docs/03 §3).
- `GET /merchants/{id}/limits` — `{ current: { collection_daily, payout_daily, per_transaction, max_tx_daily }, consumed_today: {...}, history: [{ changed_at, by, approved_by?, old, new }] }`.
- `limit_change_requests` **DC** — `{ collection_daily?, payout_daily?, per_transaction?, max_tx_daily?, reason }` (≥ 20 chars) → `approval_request` type `limit_change`. Decreases are always applied immediately via `PATCH` (protective).
- `suspension_requests` **DC** — `{ action: "suspend"|"reactivate", scope: "collections"|"collections_and_payouts"|"full_freeze", reason_code, reason_detail, emergency_freeze_4h?: bool }` → `approval_request` type `merchant_suspension`. `emergency_freeze_4h` cuts the API conservatively pending approval; auto-lifted on reject/expiry. Balances are frozen, never seized — no ledger entry.
- `impersonate` **step-up** — `{ reason }` (≥ 15 chars) → `{ url, expires_at }`: single-use token URL to app.ijimpay.com, server-side read-only, 30 min, fully audit-logged.
- `notes` — `{ text }` (3–2 000 chars) → append-only; never editable or deletable.

## 6. Transactions (O-07, O-08)

```
GET  /transactions                        cross-merchant search of charges / payouts / refunds
GET  /transactions/export                 CSV of current search               [ops-admin, finance-admin, compliance]
GET  /transactions/{id}                   full internal detail
POST /transactions/{id}/repoll            manual provider re-poll             [ops]
POST /webhook_deliveries/{id}/redeliver   re-send a webhook delivery          [ops]
```

- Search params: `?query=&provider_ref=&phone=&amount[gte|lte]=&created[gte|lte]=&status=&channel=&merchant_id=&type=charge|payout|refund` (period ≤ 366 days). Row: `{ type, merchant: { id, name }, status, channel, amount, reference, provider_ref, phone, failure_code, created_at }` — field semantics per docs/02 §2.1/§2.3.
- `GET /transactions/{id}` embeds: the docs/02-shaped object, `provider_transactions` (incl. `raw_request` / `raw_response` with PII masked; unmasking via `?reveal_pii=true` is audit-logged), ledger postings (linked `journal_entries`), `webhook_deliveries`, linked objects (link, invoice, refund, batch), timeline (`last_polled_at`, `poll_count`).
- `repoll` — no body; rate limit 1/min/transaction (`429 rate_limited` with `Retry-After`). Response `{ provider_status }`; a status change runs the normal state machine, merchant webhooks included.
- `redeliver` — re-enqueues the delivery immediately; response `202`.

## 7. Reconciliation (O-09, O-10)

```
GET  /reconciliation/reports              imported provider reports + run summaries
POST /reconciliation/reports              multipart import (provider, period, file) → auto-match run  [finance-admin]
POST /reconciliation/reports/{id}/rematch re-run matching                     [finance-admin]
GET  /reconciliation/items                discrepancy queue (?status=missing_ours|missing_theirs|amount_mismatch)
GET  /reconciliation/items/{id}           item + embedded charge + raw provider payloads
POST /reconciliation/items/{id}/claim     assign to me
POST /reconciliation/items/{id}/resolve   resolve the discrepancy             [finance-admin]
POST /reconciliation/items/{id}/notes     append-only note (+ attachments)
```

- Report row: `{ id, provider, period: { from, to }, file_name, imported_at, line_count, match_rate, status: matched|running|failed }`. Import validates required columns (`provider_ref, amount, phone, timestamp`), ≤ 100 MB; overlap with an existing report is a non-blocking warning — matching dedupes.
- Item (docs/03 §4): `{ id, report_id, status, provider_ref, amount, phone, provider_timestamp, matched_charge_id, sla, assignee, resolved_by, resolution_note, charge?: {docs/02 shape}, raw: {...} }`.
- `resolve` — `{ resolution_type: "manual_match"|"provider_error"|"adjustment_required"|"duplicate_ignore", matched_charge_id?, note (≥ 20 chars), provider_ticket_ref? }`. `manual_match` requires an exact amount match (`invalid_request` otherwise) and posts the catch-up journal entry (canonical entries, docs/03 §3). `adjustment_required` leaves the item `resolving` until the linked adjustment (§9) is second-approved.

## 8. Providers (O-11)

```
GET  /providers/health                    per-provider metrics, float vs ledger, breaker states
POST /providers/{provider}/synthetic_check  run a synthetic charge probe      [ops]
POST /providers/{provider}/breaker_requests DC — trip/restore circuit-breaker [ops-admin]  DC
```

- `health` — per provider (`mtn_momo`, `orange_money`): `{ status, success_rate_1h, success_rate_24h, latency_p95_ms, pending_median_age_s, float: { ledger_balance, provider_declared, delta, last_reconciled_at }, synthetic_checks: [{ at, direction, duration_ms, result }], breakers: { collections: { state, since, maker, checker, reason }, disbursements: {...} } }`. `delta ≠ 0` auto-creates a reconciliation item (O-09).
- `synthetic_check` — `{ direction: "collection"|"disbursement" }`; magic numbers in test env, 1 FCFA internal probe in prod. Result appended to the checks table.
- `breaker_requests` **DC** — `{ action: "trip"|"restore", directions: ["collections","disbursements"], reason_code, reason_detail, auto_restore_after?: "1h"|"4h"|"12h" }` → `approval_request` type `provider_breaker` (urgent: immediate bell + email to all ops-admins). On approval: `GET /channels` (docs/02) returns `down` for the cut direction and new charges/payouts on it fail `channel_unavailable`.

## 9. Risk (O-12, O-13, O-14)

```
GET   /risk/rules                         detection rules
PATCH /risk/rules/{id}                    edit threshold / severity / active  [compliance]
GET   /risk/flags                         flagged merchants/transactions (?target=merchant|transaction&status=)
POST  /risk/flags/{id}/dismiss            dismiss with short reason           [compliance]
GET   /risk/cases                         cases list
POST  /risk/cases                         open a case (from a flag or a merchant)  [compliance]
GET   /risk/cases/{id}                    case + linked items + notes
POST  /risk/cases/{id}/items              attach a transaction/flag/document  [compliance]
POST  /risk/cases/{id}/notes              append-only note (+ attachments)    [compliance, ops-admin]
POST  /risk/cases/{id}/close              close with decision                 [compliance]
POST  /risk/cases/{id}/str_packs          generate STR evidence pack (async)  [compliance]
GET   /risk/str_packs/{id}                pack status + 24 h signed download URL  [compliance]
POST  /risk/str_packs/{id}/mark_filed     record ANIF filing                  [compliance]
```

- Rule: `{ id, name, description, threshold: { value, window: "1h"|"24h"|"7d" }, severity: info|medium|high, active, triggers_7d }`. `PATCH` accepts `{ threshold?, severity?, active? }` (threshold: positive integer).
- Flag: `{ id, rule: { id, name }, target_type, target_id, observed_value, threshold_value, raised_at, status: new|in_case|dismissed }`. `dismiss` requires `{ reason }`.
- Case: `{ id, number, merchant_id, severity, status: open|investigating|closed_unfounded|closed_founded|str_filed, assignee, sla, items: [{ type, reference_id, added_by, added_at }], notes: [...] }`. `close` — `{ decision: "closed_unfounded"|"closed_founded"|"escalate_str", summary (≥ 100 chars) }`; `escalate_str` routes to the STR pack builder.
- `str_packs` create — `{ period: { from, to }, suspicion_summary (200–5 000 chars), include_kyb_documents: bool, include_raw_payloads: bool, item_ids: [], format: "pdf_csv_zip" }` → `202` `{ id, status: "generating" }`; bell + email when ready. `mark_filed` — `{ anif_ack_ref }` → case status `str_filed`.

## 10. Ledger & Adjustments (O-15, O-16, OM-10)

```
GET  /ledger/accounts                     chart of accounts + current balances (account picker, O-16)
GET  /ledger/journal_entries              list (?merchant_id=&kind=&reference_id=)
GET  /ledger/journal_entries/{id}         entry + postings + before/after balances (OM-10 read-only)
GET  /adjustments                         manual adjustments with DC trail (?status=&account=&created_by=)
POST /adjustments                         DC — submit a manual journal entry  [finance-admin]  DC
```

- Account: `{ id, type ("provider_float:mtn"|"merchant_available"|…, docs/03 §3 chart), owner_type, owner_id, balance }`.
- Journal entry: `{ id, kind, description, reference_type, reference_id, created_by, postings: [{ account: { type, owner }, amount }], balances_after: [...] }` — signed amounts, sum always 0.
- `POST /adjustments` **DC** — `{ debit_account_id, credit_account_id, amount (> 0, ≤ 50 000 000), reason_code: "recon_correction"|"goodwill"|"internal_error"|"fraud_recovery"|"other", linked_reference?, description (≥ 20 chars) }`. Server re-validates balance: postings not summing to zero → `422 unbalanced_journal_entry` (nothing is written). Creates an `approval_request` type `adjustment` (expiry **72 h**, not 24 h); the entry posts only on second approval. Debiting into negative is allowed with a warning client-side, never blocked server-side.

## 11. Approval requests — dual-control lifecycle (O-17, OM-11)

The single handshake behind every **DC** endpoint above. Canonical rules (docs/13 "règle DC"):

1. **Make** — a DC endpoint (`suspension_requests`, `limit_change_requests`, `breaker_requests`, `role_change_requests`, `adjustments`, tier-2 `kyb/reviews/{id}/approve`) creates an `approval_request` `{ id, type, payload (full intended effect, immutable snapshot), maker, status: "pending_approval", created_at, expires_at }` and returns `202` with it. Nothing is applied yet.
2. **Approve** — a different staff user holding the required approver role calls `POST /approval_requests/{id}/approve` with a fresh `webauthn_assertion`. The server enforces: approver ≠ maker (`403 approver_is_maker`), role eligibility (`403 permission_denied`), assertion freshness (`401 webauthn_required`), non-expiry (`409 approval_request_expired`). On success the payload's effect executes **atomically** (ledger posting / limits / suspension / breaker / role / tier) and the maker is notified.
3. **Reject** — `POST /approval_requests/{id}/reject` `{ reason }` (≥ 10 chars, required). Same approver checks minus WebAuthn effect execution.
4. **Expire** — unactioned requests expire server-side after 24 h (adjustments: 72 h); status `expired`, maker notified.
5. **Cancel** — the maker (only) may `POST /approval_requests/{id}/cancel` while pending.
6. Every transition (create/approve/reject/expire/cancel) writes `audit_logs`.

```
GET  /approval_requests                   queue (?status=pending&type=&maker=&sort=sla)
GET  /approval_requests/{id}              full request (payload + journal-entry preview if monetary)
POST /approval_requests/{id}/approve      DC decision: execute effect          [eligible approver ≠ maker]  step-up
POST /approval_requests/{id}/reject       DC decision: reject with reason      [eligible approver ≠ maker]  step-up
POST /approval_requests/{id}/cancel       maker withdraws their own request
```

Approver roles per type: `adjustment`, `limit_change` → finance-admin or ops-admin; `merchant_suspension`, `provider_breaker`, `kyb_tier2`, `staff_role_change` → ops-admin. Monetary types embed `journal_entry_preview` (OM-10 shape) in the GET response.

## 12. Audit log (O-18)

```
GET /audit_logs                           search (?created[gte|lte]=&staff_id=&action=&merchant_id=&query=)
GET /audit_logs/export                    CSV, ≤ 90-day window per export     [ops-admin, compliance]
```

Row: `{ id, at, staff: { id, name }, action, target_type, target_id, ip, detail: {...} }`. Non-privileged roles (`ops`, `finance-admin`) are server-side restricted to `staff_id = self`. The export itself is audit-logged (meta-logging).

## 13. Ops settings (O-20)

```
GET   /settings                           current settings + change history
PATCH /settings                           save changes                        [ops-admin; compliance: SLA fields only]
```

Fields: `sla: { kyb_hours (4–72), recon_hours (8–168), risk_hours (24–336), approvals_hours (1–24) }, security: { session_hours (1–12), allowed_cidrs: [] (≥ 1 valid CIDR) }, alerts: { discrepancy_threshold_xaf (≥ 100 000) }, staff_notifications: { <event>: { <role>: ["bell","email"] } }`. WebAuthn step-up requirement and DC expiry windows are read-only constants. Each change is historized `{ key, old, new, by, at }` and bells ops-admins.

---

## Cross-reference: pages → endpoints

O-01 §2 · O-02 §3 (+ docs/02 `GET /channels`) · O-03/O-04 §4 · O-05/O-06 §5 (+ §6, §9-list, §10) · O-07/O-08 §6 · O-09/O-10 §7 · O-11 §8 · O-12/O-13/O-14 §9 · O-15/O-16 §10 · O-17 §11 · O-18 §12 · O-19 §2 · O-20 §13 · O-21/O-22/O-23/O-24 — no endpoints (static/edge). Endpoints added beyond the docs/13 list (pages needed them): `POST /auth/logout`, `GET /notifications` + `POST /notifications/{id}/read` (topbar bell), `GET /transactions/export` (O-07), `GET /merchants/{id}/limits` (O-06 tab 3), `GET /ledger/journal_entries` + `/{id}` (O-06 tab 4, O-08, OM-10 read-only), `GET /approval_requests/{id}` (OM-11), `GET /risk/str_packs/{id}` (O-14 generation status/download), `POST /staff/{id}/webauthn/enroll` + `DELETE /staff/{id}/webauthn/credentials/{credId}` (O-19 key management).

```
INVENTORY: endpoints=77 error_codes=5
```
