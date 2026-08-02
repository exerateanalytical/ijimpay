# IjimPay — Merchant Mobile App Specification

Status: Draft v1 · The app is the primary surface for small merchants and the companion surface for everyone else.

---

## 1. Positioning

The mobile app is a **merchant tool**, not a consumer wallet (customers pay with their existing MTN/Orange wallets — we never hold consumer accounts). It turns any Android phone into:

1. A **mini-POS** — accept a payment at the counter in under 15 seconds.
2. A **payment-link machine** — create and share links/QRs to WhatsApp.
3. A **business monitor** — real-time notifications, daily totals, balance.
4. An **approval device** — approve payouts/payroll from anywhere (maker–checker).

## 2. Platform & Technology

**Recommendation: Flutter.**

| Option | Verdict |
|---|---|
| **Flutter (Dart)** | ✅ Chosen. Single codebase Android+iOS, excellent performance on low-end Android (the Cameroonian reality: Android 8–12, 2–3 GB RAM), first-class offline/local-DB tooling, small team can ship both platforms. |
| React Native | Viable; chooses JS consistency with backend over low-end-device performance. Acceptable fallback if hiring dictates. |
| Native Kotlin + Swift | Best per-platform quality but doubles the team; not justified at this stage. |

Decisions:
- **Android-first release** (≈ 95% of the market); iOS from the same codebase one cycle later.
- Min Android 8.0 (API 26); APK size budget < 25 MB; cold start < 2.5 s on a 2 GB-RAM device.
- **State/data**: Riverpod + Drift (SQLite) for local cache; `dio` client generated from the same OpenAPI spec as the public SDKs — the app consumes the public API plus a small `/app`-scoped API (device registration, push tokens).
- **Push**: Firebase Cloud Messaging; payment-received notifications must arrive < 3 s after webhook event (dedicated high-priority channel with custom cash-register sound 🔔 — this sound is a feature; merchants leave the phone by the till).
- **Offline tolerance**: read screens render from local cache instantly, then refresh; write actions (create link, initiate charge) queue with retry when data is flaky — never lose merchant input. Clear "waiting for network" states.
- **Auth**: phone + OTP login, then PIN/biometric unlock; device binding (payout approval only from registered devices); role-aware UI (a Viewer sees no payout button).
- **i18n**: French default, English toggle. All amounts formatted `5 000 FCFA`.

## 3. Feature Set by Release

### v1.0 — "Get Paid" (ships with platform Phase 2)

| Feature | Detail |
|---|---|
| **Charge at counter** | Numpad amount → choose MTN/Orange (or auto from payer phone) → enter payer phone OR show QR → live status screen ("Client is approving on their phone…") → success screen + sound. Target: < 15 s merchant effort. |
| **QR at counter** | Static reusable QR (open amount) printable from the app; dynamic QR per charge. |
| **Payment links** | Create (title, amount fixed/open), share sheet (WhatsApp/SMS/copy), list with per-link earnings, deactivate. |
| **Transactions** | Live list, filters (today/week/channel/status), detail with timeline, receipt share (image/PDF to WhatsApp). |
| **Home** | Today's collected total, count, success rate, balance (available/pending), provider health banner (e.g. "Orange Money is currently slow"). |
| **Notifications** | Push on every succeeded/failed charge; daily summary at close of business. |
| **Onboarding** | Signup, KYB document capture with camera (guided shots), verification status tracker. |

### v1.1 — "Money Out & Team"

- Payout approvals (review batch → biometric confirm), single payout to a saved beneficiary, balance top-up instructions.
- Team management (invite by phone, assign role), multi-business switcher.
- Settlement history + statement download/share.

### v1.2 — "Grow"

- Mini product catalog (name, price, photo) → catalog payment links.
- Analytics screens (weekly trends, best days, channel split).
- Subscription monitor (active/past-due counts, tap-to-nudge sends dunning link).
- Bluetooth receipt-printer support (common thermal printers) — makes it a real POS.

### Later / MAY

- Offline "payment pending" QR flows, kiosk mode for cashier staff (restricted per-device role), tablet layout, iOS-specific polish, in-app support chat.

## 4. Key Screen Flows (v1.0)

```
[Home] ── "Encaisser" (big primary button)
   └─▶ [Amount numpad] → [Channel: MTN | Orange | QR]
         ├─ phone entered ─▶ POST /charges ─▶ [Waiting screen: animated, 
         │                     "Demandez au client de composer son code"]
         │                     ├─ webhook/poll: succeeded ─▶ [✅ + sound + share receipt]
         │                     └─ failed/expired ─▶ [Retry | Switch channel]
         └─ QR shown ─▶ customer scans → hosted checkout on their phone → same status screen
```

Design principles: huge tap targets, works one-handed, every state has a French sentence a non-technical merchant understands, no jargon ("En attente du client", not "PENDING_PROVIDER").

## 5. App Architecture

```
lib/
  core/        api client (OpenAPI-generated), auth/session, push, offline queue, i18n, theme
  features/
    home/  charge/  links/  transactions/  payouts/  team/  onboarding/  settings/
  shared/      widgets, formatting (FCFA, phone), error mapping
```

- Feature-first folders, each with `data / domain / ui` layers; Riverpod providers per feature.
- All money displayed from server-provided integers; the app never computes fees.
- Crash/perf: Sentry + Firebase Performance; analytics events for funnel (charge started → succeeded).
- Release: fastlane; Play internal → closed track (pilot merchants) → production; feature flags via simple remote config for kill-switches.

## 6. Security (app-specific)

- Secret keys never in the app — the app authenticates as a **user session** (JWT, short-lived + refresh), scoped by role; certificate pinning to api.ijimpay.com.
- PIN/biometric gate for launch after timeout and for every payout approval; screenshots blocked on sensitive screens (FLAG_SECURE).
- Device registry: payout approval requires an enrolled device; remote revoke from dashboard.
- Root/jailbreak detection → warn + disable payout approval.

## 7. Mobile Team & Estimate

| Phase | Scope | Effort (2 Flutter devs + shared designer) |
|---|---|---|
| Design system + auth + onboarding | 3–4 wks |
| v1.0 core (charge, links, transactions, push) | 6–8 wks |
| Pilot hardening (offline, perf, FR copy polish) | 2–3 wks |
| v1.1 | 4–5 wks |

v1.0 on Play Store ≈ **3–3.5 months** after backend charge API is stable in sandbox (app dev can start against the simulated provider from week 1).
