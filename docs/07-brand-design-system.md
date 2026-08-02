# IjimPay — Brand & Design System

Status: Draft v1 · Applies to: dashboard, hosted checkout, mobile app, marketing site, emails, receipts.

---

## 1. Brand Foundations

**Name**: **Ijim Pay** (two words in prose and marketing; wordmark and code namespace `ijimpay`; domains ijimpay.com / pay.ijimpay.com). "Ijim" evokes the Ijim highlands of the North-West; grounded, Cameroonian, trustworthy. Never write "IJIMPAY" or "ijim-pay".
**Tagline**: EN "Get paid, simply." · FR **"Encaissez, simplement."** (FR is the lead language everywhere.)

**Personality**: trustworthy > friendly > modern. Never playful with money states; celebratory only on success moments. Voice: short sentences, no fintech jargon, always FR-first with EN parity. Say "Paiement reçu", never "Transaction OK".

## 2. Color

Palette (WCAG AA against stated backgrounds; verified pairs only):

| Token | Hex | Usage |
|---|---|---|
| `brand-900` | `#0B3B2E` | Deep forest — headers, hero backgrounds |
| `brand-600` (primary) | `#0E7C5A` | Primary buttons, links, active states |
| `brand-100` | `#E3F4EE` | Selected/background tints, success surfaces |
| `accent-500` | `#F5A623` | Highland amber — marketing highlights and CTAs on dark surfaces ONLY. Never used for statuses (see collision rules below) |
| `ink-900` | `#101828` | Primary text |
| `ink-500` | `#667085` | Secondary text |
| `surface` | `#FFFFFF` / `paper #F7F9F8` | Cards / page background |
| `success-600` | `#12B76A` | Succeeded states |
| `warning-600` | `#D97706` | Pending, degraded provider |
| `danger-600` | `#D92D20` | Failed, destructive actions |
| `info-600` | `#175CD3` | Informational banners |

Channel colors (used ONLY as small identity chips, never as UI semantics): MTN `#FFCC00` chip with black text; Orange `#FF7900` chip with white text. Status color always wins over channel color.

### Color-collision rules (mandatory)

Three color families in this product are dangerously close: our amber accent (`#F5A623`), the pending/warning amber (`#D97706`), and MTN's brand yellow (`#FFCC00`); likewise Orange's brand orange (`#FF7900`) sits near danger red (`#D92D20`). To keep them unambiguous:

1. **One meaning per hue in any given view.** Status ambers (`warning-600`) and channel yellow (MTN) may appear in the same row ONLY because channel chips always carry the operator logo dot + text label — a chip is never a bare color swatch.
2. `accent-500` is banned from transactional UI (tables, badges, banners, buttons in dashboard/app/checkout). It lives only in marketing pages, empty-state illustrations, and dark-hero CTAs. The former use of accent for `pending_approval` badges is replaced by `warning-600` outline style (filled = pending, outlined = pending_approval).
3. Channel chips never tint a row, card, button, or background — 24px chip max, always logo + label, `ink-900`/white text per chip, identical size for both operators (no visual favoritism).
4. Danger red is reserved for failed/destructive. The Orange chip may never be enlarged, glowed, or used as a button color, so it cannot be mistaken for an error affordance.
5. Success green vs brand green: `success-600 #12B76A` is only for StatusBadge/status text; interactive elements use `brand-600 #0E7C5A`. A green button always means "action", a green badge always means "succeeded".
6. Dark mode: statuses switch to 400-weight variants (`success-400 #32D583`, `warning-400 #FDB022`, `danger-400 #F97066`, `info-400 #528BFF`) — all AA on `ink-900`; channel chips keep their light-mode look (they are logos, not UI).
7. Color-blind safety: every status is icon + label + color (already required in §4); pending vs succeeded differ by shape (`clock` vs `circle-check`), never by hue alone.

Dark mode: mobile app and dashboard support dark (`ink-900` surfaces, `brand-400 #34D399` primary); hosted checkout is light-only v1 (predictability on cheap screens).

## 3. Typography

- UI: **Inter** (variable). Display/marketing: **Bricolage Grotesque**. Numeric: Inter tabular-nums for ALL amounts and tables.
- Scale (px): 32/24/20 headings · 16 body · 14 secondary · 12 caption. Mobile app minimum body 16.
- Amounts: always `12 500 FCFA` (non-breaking thin space thousands, currency after, never "XAF" in UI). Big-money moments (success screen, balance) use 40px tabular.

## 4. Iconography — Lucide (canonical set)

Library: **lucide** (web `lucide-react`, app `lucide_flutter` or exported SVGs). Stroke 2px, size 20 (web) / 24 (app), `ink-500` default, `brand-600` active. One icon = one meaning, fixed across all surfaces:

| Concept | Lucide icon | Concept | Lucide icon |
|---|---|---|---|
| Home / overview | `layout-dashboard` | Collect / charge | `hand-coins` |
| Transactions | `arrow-left-right` | Payment link | `link` |
| QR code | `qr-code` | Checkout / cart | `shopping-cart` |
| Payouts / send | `send` | Payroll | `users` + `banknote` (composite header) |
| Batch | `layers` | Approval (maker–checker) | `check-check` |
| Balance / wallet | `wallet` | Settlement | `landmark` |
| Top-up | `plus-circle` | Refund | `undo-2` |
| Subscription | `repeat` | Invoice | `receipt` |
| Customer | `user-round` | Team | `users-round` |
| Roles/permissions | `shield` | API keys | `key-round` |
| Webhooks | `webhook` | Events/logs | `scroll-text` |
| Docs | `book-open` | Sandbox/test mode | `flask-conical` |
| Success | `circle-check` | Pending | `clock` (animated pulse) |
| Failed | `circle-x` | Expired | `timer-off` |
| Warning/degraded | `triangle-alert` | Provider health | `activity` |
| Notifications | `bell` | Share | `share-2` |
| WhatsApp share | `message-circle` | SMS | `message-square` |
| Copy | `copy` | Download/export | `download` |
| Filter | `list-filter` | Search | `search` |
| Settings | `settings` | KYB/verification | `badge-check` |
| Security/PIN | `lock` | Biometric | `fingerprint` |
| Sign out | `log-out` | Help/support | `life-buoy` |
| Analytics | `chart-line` | Printer (POS) | `printer` |
| Offline/queued | `cloud-off` | Language | `globe` |
| Danger/destructive | `trash-2` | Edit | `pencil` |
| Back | `arrow-left` | More actions | `ellipsis-vertical` |
| Retry / redeliver / re-poll | `rotate-cw` | Reveal / hide (passwords, amounts) | `eye` / `eye-off` |
| Upload | `upload` | Document capture (mobile) | `camera` |
| CSV import | `file-up` | Column mapping | `table` |
| Validation report | `list-checks` | Date / period picker | `calendar` |
| Invite member | `user-round-plus` | Enrolled device | `smartphone` |
| Close modal/drawer/widget | `x` | Maintenance | `wrench` |
| Re-sign-in | `log-in` | Subscription pause / resume | `circle-pause` / `circle-play` |
| Reconciliation & discrepancies | `scale` | Risk (queue, cases, flags) | `siren` |
| Suspend / circuit-breaker | `octagon-pause` | Manual journal entries | `book-text` |
| Quantity decrement | `minus-circle` | Accordion | `chevron-down` |
| Open in browser | `external-link` | Call by phone (merchant, voice OTP) | `phone` |
| Ops read-only impersonation | `venetian-mask` | Image | `image` |
| Torch (scanner) | `flashlight` | Keypad backspace | `delete` |
| List chevron | `chevron-right` | Contact picker | `book-user` |
| Printer pairing | `bluetooth` | Add (app FAB) | `plus` |
| Video tutorials | `play-circle` | Business switcher | `building-2` |
| Statements | `file-text` | | |

**Arbitrations** (cross-doc conflicts, decided here — these names are canonical):

- "Call by phone" uses `phone` everywhere (calling the merchant, voice OTP fallback). `phone-call` is **banned**.
- `eye` / `eye-off` mean ONLY reveal/hide (passwords, amounts). Ops read-only impersonation uses `venetian-mask` — never `eye`.
- `cloud-off` is the only offline/queued icon; `wifi-off` is **banned** (docs/12 alternative rejected).
- The `draft` pill (payout batches) is NOT a 9th StatusBadge status — it is an outline `ink-500` chip only; the 8 canonical statuses in §5 are unchanged.

Rules: never two different icons for the same concept; never repurpose a status icon for navigation; status icons always pair with a text label (no color/icon-only meaning — accessibility).

### Icon rendering rules (no shadows, no effects)

- **No drop shadows, glows, or bevels on icons — ever.** Lucide is a 2px-stroke line system; effects blur strokes at 20–24px and look broken on low-DPI Android screens. Elevation belongs to containers (cards/sheets), never to glyphs.
- No filled/duotone variants mixed in: stroke-only, single color per icon, `currentColor` so icons inherit text color and never drift from their label's color.
- Icons never scale past 24px in UI (large visual moments use the illustration set, not blown-up icons); never below 16px (strokes alias).
- Touch targets: the tappable area is 48px even when the glyph is 24px — padding, not a bigger icon.
- The raised "Encaisser" center button in the app bottom nav is a `brand-600` circular container with a white icon; the container may have elevation-2 shadow — the icon itself still has none.
- Exports for Flutter: SVGs from the same Lucide version pinned in web `package.json`; no re-drawn or off-library icons without adding them to this map first.

## 5. Core Components (shared vocabulary web + app)

- **StatusBadge**: pill with icon + label — `succeeded` (success-600 tint), `pending` (warning + pulsing `clock`), `failed`, `expired`, `refunded` (ink-500), `processing`, `pending_approval` (accent-500). Same 8 statuses, same colors, everywhere.
- **AmountText**: tabular, sign-aware (`+4 900 FCFA` green for money in, `−250 000 FCFA` ink for money out), size variants.
- **ChannelChip**: MTN/Orange mini chip with logo dot + label.
- **TxRow**: [StatusBadge] [customer phone/name] [description] [ChannelChip] [AmountText] [time] — identical anatomy in dashboard table and app list.
- **EmptyState**: illustration + one sentence + one primary action (every list screen defines its empty state; specified per screen in docs 08–10).
- **Banner**: info/warning/danger page-level (provider degraded, KYB pending, webhook endpoint failing).
- **ConfirmSheet/Modal**: destructive and money-moving confirmations always restate amount + beneficiary; approve buttons are never default-focused.
- Buttons: primary (brand-600), secondary (outline), destructive (danger-600), min touch target 48px in app; loading state = spinner replaces label, button stays same width.

## 6. Layout

**Dashboard (web)**: left sidebar 240px (collapsible to icons) · topbar with merchant switcher, **test/live toggle** (amber `flask-conical` banner across top whenever in test mode), search (⌘K), bell, avatar · content max-width 1200px, 24px grid gap. Tables: sticky header, row click → right-side detail drawer (never lose list context); full page only for editors.

**Hosted checkout**: single column, max 420px, centered; merchant logo top; amount as hero; one action per screen; total page weight < 150 KB, system-font fallback while Inter loads.

**Mobile app**: bottom nav 5 tabs — Accueil `layout-dashboard` · Encaisser `hand-coins` (center, raised 64px FAB-style) · Liens `link` · Activité `arrow-left-right` · Menu `settings`. 16px screen padding, cards 12px radius, sheet-based sub-flows (amount pad, share, confirm) rather than deep stacks.

**Receipts** (image/PDF/print): 80mm-thermal-friendly monochrome layout — logo, merchant name, amount XXL, status, ref, date, QR verify link (`pay.ijimpay.com/r/{id}`).

## 7. Motion & Feedback

- Durations 150–250ms; ease-out. No motion on checkout critical path except the pending-state pulse.
- **Success moment** (app + checkout): check-circle scale-in + brand-600 flash + cash-register sound (app only, toggleable) + haptic. This is the signature interaction — identical everywhere.
- Pending: pulsing `clock` + progress copy that changes at 15s/45s ("Demandez au client de valider…" → "Toujours en attente — le client a-t-il reçu la demande ?").
- Skeletons for lists, never spinners > 400ms full-screen.

## 8. Accessibility & Localization

- AA contrast minimum; visible focus rings; all interactive elements labeled for screen readers; status never conveyed by color alone.
- FR default, EN via `globe` switcher; all strings in ICU message format from day 1; dates in `fr-CM` style (`02 août 2026, 14:05`).
- Phone input: national format display `6 70 00 00 00` with `+237` fixed prefix; validate MTN (650–654, 67x, 680–684) vs Orange (655–659, 69x, 685–689) prefixes for channel auto-detect, always overridable.

## 9. Assets To Produce (design backlog)

- [ ] Logo suite (wordmark, mark, mono, favicon, app icon adaptive) — brief: geometric "ij" ligature forming an upward path/mountain ridge
- [ ] Figma library: tokens above + components §5 (web + app variants)
- [ ] Empty-state illustration set (6: transactions, links, payouts, team, webhooks, search)
- [ ] Success sound (≤ 1s, distinctive, pleasant at market-stall volume)
- [ ] Receipt + statement PDF templates; email template (header/footer) in brand
- [ ] Play Store / App Store listing assets (FR + EN screenshots)
