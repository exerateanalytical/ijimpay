# IjimPay — Web Dashboard & Ops Console: Every Page, Every Action

Complements `07-brand-design-system.md` (tokens, icons, components). Route map, layout, actions, states for each page. Icons are Lucide names.

---

## A. Merchant Dashboard (`app.ijimpay.com`)

### Route map

```
/login  /signup  /verify-otp  /forgot-password  /accept-invite
/onboarding/*            (business info → documents → account → done)
/                        Home
/transactions            /transactions/:id (drawer)
/links                   /links/new  /links/:id
/checkout-sessions/:id   (read-only detail)
/customers               /customers/:id
/payouts                 /payouts/new  /payouts/batches/:id  /payouts/beneficiaries
/subscriptions           /subscriptions/plans  /subscriptions/:id
/balance                 (balance + balance transactions + top-up)
/settlements             /settlements/:id
/developers/keys  /developers/webhooks  /developers/webhooks/:id  /developers/events  /developers/logs
/team                    /settings/business  /settings/verification  /settings/notifications  /settings/security
```

Global chrome: sidebar groups — Aperçu, Encaissements (Transactions, Liens, Clients), Décaissements (Paiements sortants, Bénéficiaires), Abonnements, Finances (Solde, Règlements), Développeurs, Équipe, Paramètres. Topbar: merchant switcher · **Test/Live toggle** (in test: persistent amber banner `flask-conical` "Mode test — aucun argent réel") · ⌘K search (tx ref, phone, link title) · `bell` notification tray · avatar menu (profile, language, sign out).

Role visibility: Viewer = read-only everywhere (buttons hidden, not disabled); Developer = + Développeurs section; Finance = + payouts create/top-up; Admin/Owner = everything incl. approvals, team, settings.

### A1. Auth & Onboarding

| Page | Layout & content | Actions | States |
|---|---|---|---|
| Login | Centered card, logo, phone-or-email + password, "Se connecter", link to signup/forgot | submit; language toggle | error (bad creds), 2FA step (TOTP input) for enrolled users, rate-limited |
| Signup | Same card; phone → OTP → set password → business name | send OTP `message-square`; resend (30s cooldown) | invalid phone, OTP wrong/expired |
| Onboarding wizard (4 steps, progress bar) | 1. Infos entreprise (name, legal form, sector, address) · 2. Documents (RCCM, ID — drag-drop + camera on mobile web, `badge-check`) · 3. Compte de règlement (MoMo/OM/bank + proof) · 4. Terminé | save & continue each step; "compléter plus tard" (docs step only) | each step resumable; step 4 shows tier status + "Explorer en mode test" CTA |
| Accept invite | Shows inviter, merchant, role; set password if new user | accept / decline | expired invite |

### A2. Home `/` — `layout-dashboard`

- **Header row**: "Bonjour {name}" + date · KYB banner if tier < requested ("Vérification en cours — volume limité", link to /settings/verification).
- **Stat tiles** (4): Encaissé aujourd'hui (AmountText XL + vs hier %) · Transactions (count + success-rate ring) · Solde disponible (`wallet`, click → /balance) · En attente de règlement.
- **Provider health strip** (`activity`): MTN ● opérationnel / Orange ● dégradé → warning Banner when degraded with copy "Les paiements Orange peuvent être lents actuellement."
- **Volume chart** (14 days, stacked by channel) + **Latest 8 transactions** (TxRow, → drawer) + **Quick actions card**: Créer un lien `link` · Encaisser `hand-coins` (opens charge modal: amount, phone, channel — same semantics as app flow) · Payer `send` (if role permits).
- Empty (new merchant): setup checklist card — Vérifiez votre entreprise → Créez votre premier lien → Testez l'API → Passez en mode réel; each with icon + done-state.

### A3. Transactions `/transactions` — `arrow-left-right`

- Filters bar: date range, status multi (StatusBadge chips), channel, amount min/max, search (ref/phone/name). Saved views (Aujourd'hui, Échecs récents). Export `download` → CSV (async email if > 10k rows).
- Table (TxRow columns) + pagination (cursor). Row → **detail drawer**: header (amount XL, StatusBadge, ChannelChip) · timeline (créé → en attente → résultat, with provider ref, poll count) · customer card (phone, name, link to /customers/:id) · fees breakdown (brut/frais/net) · related objects (link, invoice, refund) · webhook deliveries for this charge (status, retry `rotate-cw` button) · Actions: **Rembourser** `undo-2` (modal: amount ≤ net, reason, confirm restates amount — Finance/Admin only) · Renvoyer le reçu (WhatsApp/SMS/email) · Copier ref.
- States: pending rows live-update (websocket/poll); empty state "Aucune transaction — créez un lien de paiement"; failed rows show failure_code human text on hover.

### A4. Payment Links `/links` — `link`

- List: card grid or table toggle — title, QR mini, amount (or "montant libre"), collected total, times paid, active toggle, created. Actions per row: copy URL `copy`, QR `qr-code` (modal: download PNG/SVG, print A6 counter card), share `share-2` (WhatsApp prefilled FR message), edit `pencil`, deactivate (confirm).
- `/links/new` (also modal from Home): title, amount type (fixe/libre with min), description, image (catalog SHOULD), reusable vs single-use, expiry, success message override. Live phone-frame **preview** of the hosted page on the right. Submit → success screen with URL + QR + share buttons.
- Link detail `/links/:id`: stats (collected, conversion views→paid), transactions filtered to link, edit panel.
- Empty: illustration + "Créez votre premier lien en 30 secondes" CTA.

### A5. Customers `/customers` — `user-round`

List (phone, name, total spent, tx count, last seen) → detail: profile, transactions, active subscriptions, notes. Action: créer un encaissement pré-rempli.

### A6. Payouts `/payouts` — `send`

- Tabs: **Paiements** (all payout items list w/ status incl. `pending_approval` accent badge) · **Lots** (batches) · **Bénéficiaires**.
- `/payouts/new`: two paths — **Individuel** (beneficiary picker or new phone + name-verify hint, amount, motif) · **En masse**: CSV upload (template download, column mapper, validation report screen listing per-row errors before anything is created) or pick saved payroll list. Review screen: total, fees, per-item table, funding check ("Solde du portefeuille de paiement : 1 200 000 FCFA — suffisant ✅ / insuffisant → CTA Approvisionner"). Submit → creates `draft` or `pending_approval`.
- Batch detail `/payouts/batches/:id`: status header, progress bar processing, per-item table (retry failed `rotate-cw`, per-item failure_code), report download. **Approval panel** (Admin/Owner, and not the maker): summary + "Approuver `check-check`" (2FA re-prompt) / "Rejeter" with reason. Audit strip: créé par X, approuvé par Y.
- Beneficiaries: CRUD, verified-name badge `badge-check`, payroll lists (named groups with amounts) for one-click monthly runs.
- States: insufficient wallet (blocking banner + top-up CTA), provider down (submit disabled for that channel with `triangle-alert` note), partial failure summary ("47/50 réussis — 3 échecs à revoir").

### A7. Subscriptions `/subscriptions` — `repeat`

- Plans tab: CRUD (name, amount, interval). Subscriptions tab: list (customer, plan, StatusBadge active/past_due/paused/canceled, next billing). Detail: invoice history with per-invoice charge attempts (retry ladder visualized), actions pause/resume/cancel, "Envoyer un lien de paiement" manual dunning `message-circle`.
- Past-due list has bulk "relancer" action.

### A8. Balance `/balance` — `wallet` & Settlements `/settlements` — `landmark`

- Balance: Disponible / En attente / Portefeuille de paiement (three AmountText tiles) + balance_transactions table (type icon per row: charge `hand-coins`, frais, paiement `send`, règlement `landmark`, ajustement, approvisionnement `plus-circle`) + **Approvisionner** flow (instructions with reference code, auto-match confirmation push when received).
- Settlements: list (period, amount, destination, status paid/in-transit) → detail: included transactions, downloadable relevé (PDF/CSV). Settings link for schedule (T+1/weekly) & destination account (change requires 2FA + cooldown banner "actif dans 24h" — anti-account-takeover).

### A9. Developers — `key-round` `webhook` `scroll-text`

- **API keys**: test & live sections; create (name it), reveal-once modal with copy, last used, revoke (confirm). Publishable keys shown inline.
- **Webhooks**: endpoints list (URL, events, status healthy/failing with `triangle-alert`), create/edit (URL, event picker, secret reveal-once), endpoint detail: delivery log (event, code, attempts, next retry) with redeliver button, "Envoyer un ping".
- **Events**: searchable event log, JSON viewer with copy.
- **Logs**: API request log (method, path, status, request_id, key used) — filter by status ≥ 400.
- Docs banner linking docs.ijimpay.com quickstart `book-open`.

### A10. Team `/team` — `users-round` & Settings

- Team: members table (name, phone, role `shield`, 2FA status, last active), invite modal (phone/email + role with role-explainer), change role, remove (confirm; cannot remove last Owner).
- Settings: Business (profile, logo upload — appears on checkout/receipts), Verification (tier status, document list w/ per-doc status & re-upload on rejection reason), Notifications (matrix per event × channel push/email/SMS, per-user), Security (password, TOTP enroll with QR, active sessions list + revoke, enrolled mobile devices list + revoke `smartphone`).

### A11. Global states

- 404 page (brand illustration, back home), 500 (request_id shown for support), maintenance page, forced-logout (session expired) toast, offline detection banner. Websocket-less fallback: 15s polling on Home/Transactions.

---

## B. Ops Console (`ops.ijimpay.com` — internal, separate app & SSO + hardware keys)

Route map: `/queue/kyb` · `/merchants` `/merchants/:id` · `/transactions` (cross-merchant search) · `/reconciliation` · `/providers` · `/risk` · `/adjustments` · `/audit` · `/staff`.

| Page | Purpose & key actions |
|---|---|
| KYB queue `badge-check` | Cards with docs viewer side-by-side against form data; approve tier / reject with reason templates (FR) that go to merchant; SLA timer per item |
| Merchant detail | Full profile, tier & limits editor (dual-control above Tier 1), balances, recent activity, risk flags, impersonate-read-only (audited), suspend (dual-control + reason) |
| Transactions search | Any field cross-merchant; detail mirrors merchant drawer + raw provider payloads viewer (JSONB), manual re-poll button |
| Reconciliation `scale` (use Lucide `scale`) | Import provider report (upload/SFTP status), match run results, discrepancy queue: missing_ours / missing_theirs / amount_mismatch tabs, each item → investigate panel (linked charge, provider raw), resolve with note (creates adjustment entry when needed, dual-control) |
| Providers `activity` | Health dashboards (success rate, latency, pending age per provider), synthetic charge results, float balances vs ledger `provider_float`, manual circuit-breaker toggle ("pause Orange collections" — dual-control, sets `/channels` to down + merchant banners) |
| Risk | Rules list (thresholds editable), flagged merchants/transactions review queue, case notes, STR export pack for ANIF filing |
| Adjustments | Create manual journal entry: debit/credit account pickers, amount, reason, reference — **requires second approver**; list with audit trail |
| Audit | Append-only viewer of all staff actions, filterable |
| Staff | Roles (ops, ops-admin, compliance, finance-admin), hardware-key enrollment status |

Every ops money action shows the resulting journal entry preview before confirm.

---

## C. Notifications & Email Matrix (fills gap — no doc covered this)

| Event | Merchant push (app) | Email | SMS | In-dashboard bell |
|---|---|---|---|---|
| charge.succeeded | ✅ (sound) | — (daily digest) | — | ✅ |
| charge.failed | ✅ | — | — | ✅ |
| payout_batch pending_approval | ✅ approvers | ✅ approvers | — | ✅ |
| payout_batch completed / partially_failed | ✅ | ✅ report attached | — | ✅ |
| settlement.paid | ✅ | ✅ relevé | ✅ | ✅ |
| balance top-up received | ✅ | ✅ | — | ✅ |
| KYB approved / rejected | ✅ | ✅ | ✅ | ✅ |
| webhook endpoint failing 24h | dev role ✅ | ✅ | — | ✅ |
| new team member / role change | — | ✅ | — | ✅ |
| security: new device / password change | ✅ | ✅ | ✅ | ✅ |
| subscription invoice failed (to merchant) | ✅ | digest | — | ✅ |
| Customer-facing (from us, merchant-branded): receipt after payment | — | optional | ✅/WhatsApp | — |
| Customer dunning (subscription) | — | — | ✅/WhatsApp link | — |

Email templates (FR+EN, brand header/footer per design system): welcome, OTP, invite, KYB result, payout report, settlement relevé, webhook failing, security alert, daily digest. All transactional; digest opt-out per user.
