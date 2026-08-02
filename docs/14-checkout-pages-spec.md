# Ijim Pay — Customer-Facing Payment Pages: Exhaustive Spec (pay.ijimpay.com)

Status: v1 — **supersedes `docs/09-checkout-screens.md`**. Complements `docs/07-brand-design-system.md` (tokens, icons, components) and `docs/02-api-spec.md` (contracts).

The payer is NOT our user. Assumptions baked into every decision below: low-end Android 8+ phone, 3G, Facebook/WhatsApp in-app webview majority, possibly first online payment ever. FR default, EN via `globe` toggle. One column, max 420px, centered. Merchant logo + name always visible in header. Footer on every screen: "Sécurisé par Ijim Pay" with `lock` icon / EN "Secured by Ijim Pay". Hosted checkout is **light-only** (design system §2). Amounts always `12 500 FCFA` (thin non-breaking space, currency after). No `accent-500` anywhere on this surface.

## Global chrome (all screens)

- **Header**: merchant logo (40px, fallback: initials circle in `brand-100`) · merchant display name (16px `ink-900`) · language toggle `globe` "FR | EN" (top-right, 48px touch target, persists in localStorage + `?lang=` param).
- **Footer**: `lock` "Sécurisé par Ijim Pay" / "Secured by Ijim Pay" · link "Qu'est-ce que Ijim Pay ?" / "What is Ijim Pay?" → ijimpay.com (opens new tab; suppressed in widget mode).
- **Test-mode ribbon** (test-mode links/sessions only): full-width `warning-600`-tinted banner, `flask-conical` icon — FR "Mode test — aucun argent réel ne sera débité." / EN "Test mode — no real money will be charged."
- Global language toggle and footer links are chrome, not counted as per-screen actions.

## Routes

```
/l/{slug}          payment link landing (C-01, C-02, C-03, C-05, C-06, C-07)
/c/{session_id}    checkout session landing (C-04, C-06)
/pay/{charge_id}   in-flight charge status (C-09, C-10, C-11, C-12)
/r/{receipt_id}    public receipt (C-14)
/r/{receipt_id}.pdf  receipt PDF (C-15)
```

## Master flow

```
[C-01|C-02|C-03|C-04 landing] → [C-08 phone & channel] → POST /v1/charges → [C-09 waiting]
   ├─ succeeded → [C-10 success] → [C-14 receipt]
   ├─ failed    → [C-11 failure] → retry → C-08
   └─ expired   → [C-12 expired] → retry → C-08
Blocked landings: C-05 inactive · C-06 expired · C-07 already paid · C-13 rate-limited · C-21 not found · C-22 server error
Degraded entry: C-17 in-app browser · Widget: C-18 loader → C-19 modal → (blocked) C-20 fallback
```

## Failure-code → payer copy mapping (canonical, used on C-09/C-11/C-12)

Every stable error code from docs/02 §1 mapped. Codes marked *(internal)* should never reach a payer; they map to the generic string as a safety net.

| `failure_code` / error code | FR (exact) | EN (exact) |
|---|---|---|
| `insufficient_payer_funds` | Solde insuffisant sur ce compte. Rechargez votre compte puis réessayez. | Insufficient balance on this account. Top up, then try again. |
| `payer_rejected` | Le paiement a été refusé sur le téléphone. | The payment was declined on the phone. |
| `payer_timeout` | La demande a expiré sans réponse. | The request expired without a response. |
| `payer_not_found` | Ce numéro n'a pas de compte {MTN Mobile Money\|Orange Money}. | This number has no {MTN Mobile Money\|Orange Money} account. |
| `channel_unavailable` | {MTN Mobile Money\|Orange Money} est indisponible actuellement. Réessayez plus tard ou utilisez l'autre opérateur. | {MTN Mobile Money\|Orange Money} is currently unavailable. Try again later or use the other operator. |
| `provider_error` | Erreur chez l'opérateur — cela ne vient pas de vous. Réessayez. | Operator error — this is not your fault. Please try again. |
| `rate_limited` | Une demande est déjà en cours sur ce numéro — validez-la sur votre téléphone ou attendez 5 minutes. | A request is already pending on this number — approve it on your phone or wait 5 minutes. |
| `invalid_request` *(internal)* | Une erreur est survenue. Réessayez ou contactez le commerçant. | Something went wrong. Try again or contact the merchant. |
| `authentication_failed` *(internal)* | same generic as above | same |
| `permission_denied` *(internal)* | same generic | same |
| `idempotency_conflict` *(internal — treat as duplicate, show existing charge status)* | same generic | same |
| `insufficient_balance` *(merchant-side, never shown to payer)* | same generic | same |
| `payout_limit_exceeded` *(payout-only, impossible here)* | same generic | same |
| charge status `expired` | La demande a expiré. Vous pouvez réessayer. | The request expired. You can try again. |

## Per-telco USSD instruction cards (CONFIG-DRIVEN — single source of truth in ops config, never hardcoded; codes verified with telcos before launch)

Card anatomy: ChannelChip (logo dot + label, 24px) · instruction text 16px · USSD code in 20px bold monospace, tap-to-dial `tel:` link where webview allows.

- **MTN MoMo** — FR: "Validez la notification reçue sur votre téléphone, **ou** composez **\*126\*1#** puis confirmez avec votre code MoMo." · EN: "Approve the notification on your phone, **or** dial **\*126\*1#** and confirm with your MoMo PIN."
- **Orange Money** — FR: "Composez **\*150\*4\*4#** pour approuver, puis confirmez avec votre code Orange Money." · EN: "Dial **\*150\*4\*4#** to approve, then confirm with your Orange Money PIN."

## Page-weight budgets (gzipped, first load, cold cache)

| Screen | Budget | Notes |
|---|---|---|
| C-01…C-07 landings | ≤ 150 KB total, ≤ 45 KB JS | System-font fallback while Inter loads; merchant image lazy, ≤ 60 KB, WebP |
| C-08 phone & channel | ≤ +10 KB over landing (same bundle) | No extra route download |
| C-09 waiting | ≤ 150 KB; polling payloads ≤ 1 KB | SSE optional enhancement |
| C-10/C-11/C-12 terminals | same bundle | Success animation = CSS only, no Lottie |
| C-13/C-21/C-22 error pages | ≤ 60 KB, **zero JS required** | Server-rendered static |
| C-14 receipt | ≤ 100 KB, **zero JS required** | Must render with JS disabled |
| C-17 in-app degraded | same as target landing + 3 KB banner | |
| C-18 widget loader | ≤ 30 KB (hard SDK contract) | |
| C-19 widget iframe | same budgets as hosted screens | |

## Payer-side accessibility rules (all screens)

1. WCAG 2.1 AA contrast; status never by color alone (icon + label, design system §8).
2. All touch targets ≥ 48px; primary CTA full-width, min height 56px.
3. `lang` attribute switches with the language toggle; all strings ICU-format.
4. Phone/amount inputs: `inputmode=numeric`, `autocomplete=tel` on phone, visible labels (never placeholder-only), errors announced via `aria-live="polite"` and linked with `aria-describedby`.
5. Status changes on C-09 announced via `aria-live="assertive"` region; countdown ring has text equivalent ("Expire dans 4 min 12 s").
6. Focus rings visible (2px `brand-600` offset 2px); focus moved to heading on route change.
7. Body text ≥ 16px; amounts hero 40px tabular; no text in images.
8. Works at 200% zoom and 320px viewport without horizontal scroll.
9. Motion respects `prefers-reduced-motion` (success scale-in becomes fade).
10. Receipt (C-14) and error pages fully functional with JS disabled.

---

# Screens

### C-01 — Lien de paiement, montant fixe / Payment link landing, fixed amount
- **Route**: `/l/{slug}` (link `amount_type=fixed`, active) · **Icon**: lucide `link` · **Access**: public (payer, no auth) · **Purpose**: present the merchant's fixed-amount payment request and start payment in one tap.
- **Layout zones**: header (global chrome) · content: merchant block → link title (20px) → optional description + image → **amount hero** (40px tabular `ink-900`) → accepted-channels row (both ChannelChips, or single chip + note when merchant accepts one channel) → sticky CTA · footer.
- **Tabs**: none.
- **Data displayed**: merchant name/logo (payment_link → merchant profile) · `payment_link.title`, `description`, image (docs/02 §2.2) · `payment_link.amount` · accepted channels (merchant config; see API additions) · fee line if merchant passes fees on: FR "dont frais : {X} FCFA" / EN "incl. fees: {X} FCFA".
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Payer 15 000 FCFA (CTA; EN "Pay 15 000 FCFA") | — | public | → C-08, amount carried; emits `amount_entered` (fixed) |

- **Modals/drawers/sheets opened**: none.
- **States**: **loading** — skeleton (logo circle, 2 text lines, amount block); **error** (link fetch fails) → C-22; single-channel note FR "Ce commerçant accepte uniquement {MTN Mobile Money\|Orange Money}." / EN "This merchant only accepts {…}."; inactive → C-05; expired → C-06; already paid (single-use) → C-07; not found → C-21.
- **Events/notifications**: analytics `link_viewed` `{slug, merchant_id, amount_type:"fixed", channel_options, referrer, in_app_browser:bool, lang}`; `cta_pay_clicked` `{slug}`.

### C-02 — Lien de paiement, montant libre / Payment link landing, open amount
- **Route**: `/l/{slug}` (`amount_type=open`) · **Icon**: lucide `link` · **Access**: public · **Purpose**: let the payer type the amount themselves (counter QR / donation / "pay what you owe" case).
- **Layout zones**: as C-01, amount hero replaced by **amount input zone** (large field + fixed "FCFA" suffix + min hint below).
- **Tabs**: none.
- **Data displayed**: as C-01 minus fixed amount; min amount hint from link config: FR "Minimum : 100 FCFA" / EN "Minimum: 100 FCFA".
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Montant à payer | numérique (`inputmode=numeric`, 32px tabular, thin-space grouping as-you-type) | entier ≥ min (défaut 100), ≤ 5 000 000; chiffres uniquement | vide | « Entrez un montant d'au moins 100 FCFA. » · au-dessus du max : « Le montant maximum est 5 000 000 FCFA. » (EN: "Enter an amount of at least 100 FCFA." / "The maximum amount is 5 000 000 FCFA.") |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Payer (devient « Payer 12 500 FCFA » dès saisie valide; EN "Pay …") | — | public | disabled until valid; → C-08; emits `amount_entered` `{amount, amount_type:"open"}` |

- **Modals/drawers/sheets opened**: none.
- **States**: loading skeleton as C-01; inline validation error (above); inactive/expired/paid/not-found → C-05/C-06/C-07/C-21; error → C-22.
- **Events/notifications**: `link_viewed` `{amount_type:"open", ...}`, `amount_entered`, `amount_validation_error` `{reason:"below_min"|"above_max"}`.

### C-03 — Lien catalogue avec quantité / Catalog link with quantity
- **Route**: `/l/{slug}` (catalog link: product name, image, unit price) · **Icon**: lucide `shopping-cart` · **Access**: public · **Purpose**: buy N units of one product from a catalog link.
- **Layout zones**: header · content: product image (16:9, ≤ 60 KB) → product name → unit price "3 500 FCFA / unité" (EN "/ unit") → **quantity stepper** (− qty +, 48px buttons) → computed total hero "Total : 10 500 FCFA" → CTA · footer.
- **Tabs**: none.
- **Data displayed**: product name/image/unit price (payment_link catalog fields — see API additions) · quantity (local) · total = qty × unit price, live.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Quantité | stepper + champ numérique éditable | entier 1–99 | 1 | « Quantité entre 1 et 99. » (EN "Quantity between 1 and 99.") |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Augmenter la quantité | `plus-circle` | public | qty+1 (max 99), recompute total, `quantity_changed` |
| Diminuer la quantité | `minus-circle` *(icon addition)* | public | qty−1 (min 1), recompute total |
| Payer 10 500 FCFA (EN "Pay …") | — | public | → C-08 with qty in charge `metadata.quantity`; `amount_entered` `{amount, quantity}` |

- **Modals/drawers/sheets opened**: none.
- **States**: as C-01 (loading/inactive/expired/paid/not-found/error); image load failure → grey placeholder block with `shopping-cart` glyph, no broken-image icon.
- **Events/notifications**: `link_viewed` `{amount_type:"catalog"}`, `quantity_changed` `{quantity}`, `amount_entered`.

### C-04 — Session de paiement e-commerce / Checkout session landing
- **Route**: `/c/{session_id}` · **Icon**: lucide `shopping-cart` · **Access**: public (unguessable id) · **Purpose**: pay a cart handed off by a merchant site via redirect.
- **Layout zones**: header · content: merchant block → **order summary accordion** (collapsed by default: "Votre commande — 3 articles" + `chevron-down` *(icon addition)*) → amount hero (session total) → CTA → cancel link · footer.
- **Tabs**: none.
- **Data displayed**: `checkout_session` cart snapshot: per-item name, qty, line amount; total (docs/03 sessions table) · merchant name/logo · session expiry countdown text when < 5 min left: FR "Cette session expire dans {m} min" / EN "This session expires in {m} min".
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Payer 42 000 FCFA (EN "Pay …") | — | public | → C-08 |
| Voir le détail de la commande (EN "View order details") | `chevron-down` | public | opens CM-06 order sheet (mobile) / expands accordion (desktop) |
| Retour à {merchant} (EN "Back to {merchant}") | `arrow-left` | public | opens CM-04 cancel confirm; on confirm redirect to `cancel_url`, session marked abandoned |

- **Modals/drawers/sheets opened**: CM-04, CM-06.
- **States**: loading skeleton; **session expired** → C-06 variant with FR "Cette session a expiré. Retournez à la boutique pour recommencer." / EN "This session has expired. Return to the shop to start again." + button to `cancel_url`; session already completed → redirect to `/pay/{charge_id}` (C-10); not found → C-21; error → C-22.
- **Events/notifications**: `link_viewed` `{surface:"session", session_id}`, `order_summary_opened`, `checkout_canceled` `{stage:"landing"}`; abandonment fires merchant webhook per docs/09 flow (server-side).

### C-05 — Lien inactif / Inactive (deactivated) link
- **Route**: `/l/{slug}` (link `active=false`) · **Icon**: lucide `circle-x` (`ink-500`, never danger — not the payer's fault) · **Access**: public · **Purpose**: dead-end gracefully with a path to the merchant.
- **Layout zones**: header · content: icon in `paper` circle → heading → body → optional contact block · footer.
- **Tabs**: none.
- **Data displayed**: merchant name/logo; merchant support contact (phone/WhatsApp) if configured.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Contacter le commerçant (EN "Contact the merchant") — shown only if contact configured | `message-circle` | public | opens CM-05 |

- **Modals/drawers/sheets opened**: CM-05.
- **States**: this page **is** a state. Copy — FR heading "Ce lien n'est plus actif" · body "Le commerçant a désactivé ce lien de paiement. Aucun montant n'a été débité." / EN "This link is no longer active" · "The merchant has deactivated this payment link. Nothing was charged." Zero-JS server render.
- **Events/notifications**: `link_viewed` `{blocked_reason:"inactive"}`.

### C-06 — Lien ou session expiré / Expired link or session
- **Route**: `/l/{slug}` (past `expires_at`) or `/c/{id}` expired · **Icon**: lucide `timer-off` (`ink-500`) · **Access**: public · **Purpose**: explain expiry, route payer back.
- **Layout zones**: as C-05.
- **Tabs**: none.
- **Data displayed**: merchant name/logo; expiry date FR "Expiré le 02 août 2026, 14:05" / EN "Expired on 02 Aug 2026, 14:05"; for sessions: `cancel_url`.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Retour à la boutique (sessions only; EN "Back to the shop") | `arrow-left` | public | redirect `cancel_url` |
| Contacter le commerçant (links, if contact configured) | `message-circle` | public | opens CM-05 |

- **Modals/drawers/sheets opened**: CM-05.
- **States**: page is the state. Copy — FR "Ce lien de paiement a expiré" · body "Demandez au commerçant de vous envoyer un nouveau lien. Aucun montant n'a été débité." / EN "This payment link has expired" · "Ask the merchant to send you a new link. Nothing was charged." Session variant body per C-04. Zero-JS.
- **Events/notifications**: `link_viewed` `{blocked_reason:"expired"}`.

### C-07 — Lien déjà payé / Single-use link already paid
- **Route**: `/l/{slug}` (single-use, one succeeded charge) · **Icon**: lucide `circle-check` (`success-600`) · **Access**: public · **Purpose**: reassure that the payment already went through and expose the receipt.
- **Layout zones**: as C-05, plus receipt summary card (amount, date).
- **Tabs**: none.
- **Data displayed**: merchant name/logo · paid amount + date from the succeeded charge · receipt URL `/r/{receipt_id}`.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Voir le reçu (EN "View receipt") | `receipt` | public | → C-14 |

- **Modals/drawers/sheets opened**: none.
- **States**: page is the state. Copy — FR "Ce lien a déjà été payé" · body "Ce lien à usage unique a été réglé le {date}. Si ce n'était pas vous, contactez le commerçant." / EN "This link has already been paid" · "This single-use link was paid on {date}. If this wasn't you, contact the merchant." Zero-JS.
- **Events/notifications**: `link_viewed` `{blocked_reason:"already_paid"}`.

### C-08 — Numéro et opérateur / Phone & channel
- **Route**: `/l/{slug}#pay` or `/c/{id}#pay` (client step, back-button safe) · **Icon**: lucide `hand-coins` · **Access**: public · **Purpose**: capture the payer's mobile-money number, detect/confirm the channel, create the charge.
- **Layout zones**: header (adds `arrow-left` back to landing) · content: amount recap line ("Vous payez 15 000 FCFA à {merchant}" / EN "You are paying … to {merchant}") → phone field → channel detection zone → fee line (if passed on) → CTA · footer.
- **Tabs**: none.
- **Data displayed**: amount + merchant recap · detected channel per prefix rules (design system §8: MTN 650–654/67x/680–684; Orange 655–659/69x/685–689): chip + FR "Numéro MTN détecté" / "Numéro Orange détecté" (EN "MTN number detected" / "Orange number detected") · fee line "dont frais : 300 FCFA" / EN "incl. fees: 300 FCFA".
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Numéro de téléphone | tel, préfixe fixe « +237 », format national espacé à la saisie « 6 70 00 00 00 », 24px | 9 chiffres, commence par 6, préfixe valide CM | vide | vide/incomplet : « Entrez votre numéro Mobile Money (9 chiffres). » · préfixe inconnu : « Ce numéro ne semble pas être un numéro camerounais valide. » (EN "Enter your Mobile Money number (9 digits)." / "This doesn't look like a valid Cameroonian number.") |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer le paiement (EN "Confirm payment") | — | public | client lock (double-tap safe) + `Idempotency-Key`; `POST /v1/charges` `{amount, currency:"XAF", channel, customer:{phone}}` via checkout backend; 202 → `/pay/{charge_id}` (C-09); errors: see states |
| Changer (d'opérateur) (EN "Change") | `pencil` | public | opens CM-01 channel picker |
| Retour (EN "Back") | `arrow-left` | public | back to landing, amount preserved |

- **Modals/drawers/sheets opened**: CM-01.
- **States**: **loading** (CTA spinner replaces label, width fixed, all inputs disabled); **unknown prefix** → detection zone forces manual pick (both chips shown, none selected, CTA disabled with helper FR "Choisissez votre opérateur." / EN "Choose your operator."); **channel not accepted by merchant** → FR "Ce commerçant n'accepte pas {Orange Money}. Utilisez un numéro {MTN}." / EN equivalent; **`channel_unavailable`** inline banner `triangle-alert` FR "Orange Money est indisponible actuellement. Réessayez plus tard ou utilisez un numéro MTN." (channel names swapped as appropriate); **`rate_limited`** → C-13; **network failure** → inline retry banner FR "Connexion impossible. Vérifiez votre réseau et réessayez." / EN "Could not connect. Check your network and try again."; other 4xx/5xx → generic FR "Une erreur est survenue. Réessayez ou contactez le commerçant."
- **Events/notifications**: `phone_submitted` `{channel, channel_detected:bool, channel_overridden:bool}` · `charge_created` `{charge_id, channel, amount}` · `charge_create_failed` `{error_code}` · `channel_changed`.

### C-09 — En attente d'approbation / Waiting for approval
- **Route**: `/pay/{charge_id}` (status `pending`) · **Icon**: lucide `clock` (pulsing, in 96px `brand-100` circle) · **Access**: public · **Purpose**: hold the payer's hand while they approve the USSD push — the make-or-break screen.
- **Layout zones**: header · hero (pulsing clock + amount + sent-to line) · **USSD instruction card** (per-telco, config-driven, see canonical cards above) · countdown ring (to `expires_at`, default 5 min, with text "Expire dans 4 min 12 s" / EN "Expires in 4 min 12 s") · progressive-help zone · escape-hatch links · footer.
- **Tabs**: none.
- **Data displayed**: `charge.amount`, `channel`, masked-none phone "6 70 00 00 00" (from the payer's own submission), `expires_at`, live `status` via 3 s polling of public status endpoint (SSE upgrade when supported) — resource `charge` per docs/02 §2.1.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Renvoyer la demande (appears at 2 min, allowed once; EN "Resend the request") | `repeat` | public | opens CM-03; on confirm: cancel + recreate charge (see API additions), stay on C-09 with reset countdown |
| Changer de numéro (appears at 2 min; EN "Change number") | `pencil` | public | opens CM-02 |
| Annuler le paiement (sessions only; EN "Cancel payment") | `arrow-left` | public | opens CM-04; on confirm → `cancel_url` |
| Un problème ? Contactez {merchant} (EN "A problem? Contact {merchant}") | `life-buoy` | public | opens CM-05 |

- **Modals/drawers/sheets opened**: CM-02, CM-03, CM-04, CM-05.
- **States** (progressive copy is the state machine of this screen):
  - **0 s**: FR "**Une demande de paiement a été envoyée au 6 70 00 00 00**" / EN "**A payment request was sent to 6 70 00 00 00**" + USSD card.
  - **45 s**: adds line (aria-live polite) FR "Toujours en attente — vérifiez vos notifications." / EN "Still waiting — check your notifications."
  - **2 min**: adds FR "Rien reçu ? Vous pouvez renvoyer la demande ou changer de numéro." / EN "Nothing received? You can resend the request or change the number." + reveals « Renvoyer la demande » and « Changer de numéro ».
  - **succeeded** → replace with C-10 (in place, no reload) · **failed** → C-11 · **expired** → C-12.
  - **polling failure** (3 consecutive misses): banner FR "Connexion instable — le paiement continue sur votre téléphone. Cette page se met à jour dès que possible." / EN "Unstable connection — the payment continues on your phone. This page will update as soon as possible."
  - **static QR counter context** (open-amount link source): success adds "Montrez cet écran au commerçant" — see C-10.
- **Events/notifications**: `waiting_viewed` `{charge_id, channel}` · `waiting_45s_reached` · `waiting_2min_reached` · `charge_resent` `{attempt:2}` · `number_changed_midflight` · `status_result` `{status, failure_code, elapsed_ms, poll_count}`. Merchant side: terminal states fire `charge.succeeded|failed|expired` webhooks + app push (server-side, not from this page).

### C-10 — Paiement réussi / Success
- **Route**: `/pay/{charge_id}` (status `succeeded`) · **Icon**: lucide `circle-check` (`success-600`, scale-in 200 ms + `brand-600` flash; fade under `prefers-reduced-motion`; no sound on web) · **Access**: public · **Purpose**: the signature success moment; give proof and route onward.
- **Layout zones**: header · success hero (check + FR "**Paiement reçu**" / EN "**Payment received**" + amount 40px) · details card (merchant, référence, date `fr-CM` "02 août 2026, 14:05", ChannelChip) · actions stack · counter-QR notice zone (conditional) · footer.
- **Tabs**: none.
- **Data displayed**: `charge.amount`, `reference`, `succeeded_at`, `channel`, merchant name · receipt id/url. For sessions: auto-redirect notice FR "Retour à {merchant} dans 3 s…" / EN "Returning to {merchant} in 3 s…" (redirect to `success_url` with `charge_id` + signature params).
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Télécharger le reçu (EN "Download receipt") | `download` | public | GET `/r/{receipt_id}.pdf` (see API additions) |
| Voir le reçu en ligne (EN "View receipt online") | `receipt` | public | → C-14 |
| Retourner à {merchant} (sessions; EN "Return to {merchant}") | `arrow-left` | public | immediate redirect `success_url` (cancels 3 s timer) |

- **Modals/drawers/sheets opened**: none.
- **States**: **counter/static-QR variant** — banner FR "**Montrez cet écran au commerçant.** Il reçoit aussi une confirmation sur son téléphone." / EN "**Show this screen to the merchant.** They also receive a confirmation on their phone." (Merchant materials say: "attendez la sonnerie, pas l'écran du client."); receipt-pdf fetch failure → toast FR "Téléchargement impossible — réessayez." / EN "Download failed — try again."
- **Events/notifications**: `succeeded` funnel event `{charge_id, channel, elapsed_ms}` · `receipt_downloaded` · `redirected_to_merchant`. Server: `charge.succeeded` webhook, merchant app push + sound, optional payer SMS/WhatsApp receipt (docs/08 §C).

### C-11 — Paiement échoué / Failure
- **Route**: `/pay/{charge_id}` (status `failed`) · **Icon**: lucide `circle-x` (`danger-600` tint circle) · **Access**: public · **Purpose**: explain the failure in human words and offer a retry path.
- **Layout zones**: header · failure hero (icon + FR "**Le paiement n'a pas abouti**" / EN "**The payment did not go through**" + amount `ink-500` struck none — plain) · reason card (mapped `failure_code` copy, canonical table above) · actions stack · footer.
- **Tabs**: none.
- **Data displayed**: `charge.failure_code` → mapped copy · amount, merchant, channel, reference.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer (EN "Try again") | `repeat` | public | → C-08 prefilled (same phone/channel/amount); new charge on submit |
| Changer de numéro ou d'opérateur (EN "Change number or operator") | `pencil` | public | → C-08 with phone cleared, CM-01 opened |
| Retour à la boutique (sessions; EN "Back to the shop") | `arrow-left` | public | redirect `cancel_url` |
| Contacter le commerçant (EN "Contact the merchant") | `life-buoy` | public | opens CM-05 |

- **Modals/drawers/sheets opened**: CM-01 (via change), CM-05.
- **States**: page is the state; `insufficient_payer_funds` adds helper line FR "Vous pouvez recharger votre compte puis réessayer — ce lien reste valable." / EN "You can top up your account and try again — this link is still valid."; `channel_unavailable` failure swaps « Réessayer » label to FR "Réessayer avec {l'autre opérateur}" when the merchant accepts both.
- **Events/notifications**: `failed` funnel event `{charge_id, failure_code, channel}` · `retry_clicked` `{from:"failure"}`. Server: `charge.failed` webhook.

### C-12 — Demande expirée / Charge expired
- **Route**: `/pay/{charge_id}` (status `expired`) · **Icon**: lucide `timer-off` (`ink-500` circle) · **Access**: public · **Purpose**: neutral (not the payer's fault) expiry message with retry.
- **Layout zones**: as C-11 with neutral tint.
- **Tabs**: none.
- **Data displayed**: amount, merchant, channel; copy FR "**La demande a expiré**" · body "Vous n'avez pas validé à temps — cela arrive. Aucun montant n'a été débité." / EN "**The request expired**" · "You didn't approve in time — it happens. Nothing was charged."
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer (EN "Try again") | `repeat` | public | → C-08 prefilled; new charge |
| Changer de numéro (EN "Change number") | `pencil` | public | → C-08, phone cleared |
| Contacter le commerçant (EN "Contact the merchant") | `life-buoy` | public | opens CM-05 |

- **Modals/drawers/sheets opened**: CM-05.
- **States**: page is the state.
- **Events/notifications**: `expired` funnel event `{charge_id, channel}` · `retry_clicked` `{from:"expired"}`. Server: `charge.expired` webhook.

### C-13 — Trop de demandes / Rate-limited
- **Route**: shown in place of C-09 when `POST /charges` returns 429 `rate_limited` (3 pending per payer phone per merchant, docs/02 §4) · **Icon**: lucide `clock` (static, `warning-600` circle) · **Access**: public · **Purpose**: stop USSD spam without losing the sale.
- **Layout zones**: header · icon + heading + body · countdown zone (from `Retry-After`) · actions · footer.
- **Tabs**: none.
- **Data displayed**: `Retry-After` seconds → live countdown FR "Vous pourrez réessayer dans {m}:{ss}" / EN "You can try again in {m}:{ss}".
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Actualiser (enabled when countdown hits 0; EN "Refresh") | `repeat` | public | retries `POST /charges` with a new Idempotency-Key; success → C-09 |
| Contacter le commerçant (EN "Contact the merchant") | `life-buoy` | public | opens CM-05 |

- **Modals/drawers/sheets opened**: CM-05.
- **States**: page is the state. Copy — FR heading "**Une demande est déjà en cours sur ce numéro**" · body "Validez la demande reçue sur votre téléphone, ou attendez 5 minutes avant d'en envoyer une nouvelle." / EN "**A request is already pending on this number**" · "Approve the request on your phone, or wait 5 minutes before sending a new one."
- **Events/notifications**: `rate_limited_viewed` `{retry_after_s}` · `rate_limited_retry`.

### C-14 — Reçu public / Public receipt
- **Route**: `/r/{receipt_id}` (public, unguessable, no auth, **zero JS required**) · **Icon**: lucide `receipt` · **Access**: public (payer, merchant, anyone verifying) · **Purpose**: tamper-proof payment proof — the anti-fake-receipt story; QR target on thermal receipts.
- **Layout zones**: header · **verified banner** (`badge-check` on `brand-100`): FR "Paiement vérifié par Ijim Pay" / EN "Payment verified by Ijim Pay" · receipt card · items table (session payments only) · anti-fraud note · actions · footer.
- **Tabs**: none.
- **Data displayed** (all from receipt resource — see API additions; mirrors `charge`): merchant name + logo · amount (40px) · StatusBadge (`succeeded` normally; `refunded` — `undo-2` `ink-500` — shown with FR "Remboursé le {date}" / EN "Refunded on {date}" when a linked refund exists) · référence (`charge.reference`) · Ijim Pay ref (`charge.id` short) · date `fr-CM` · ChannelChip · payer number partially masked "6 70 •• •• 00" · session items table (Article | Qté | Montant). Anti-fraud note: FR "Vérifiez toujours vos reçus sur pay.ijimpay.com — un vrai reçu Ijim Pay s'ouvre toujours sur cette adresse." / EN "Always verify your receipts on pay.ijimpay.com — a genuine Ijim Pay receipt always opens at this address."
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Télécharger le PDF (EN "Download PDF") | `download` | public | GET `/r/{receipt_id}.pdf` (C-15) |
| Copier le lien du reçu (EN "Copy receipt link") | `copy` | public | clipboard (JS-enhanced; hidden when no JS) |
| Signaler un problème (EN "Report a problem") | `life-buoy` | public | mailto/WhatsApp to Ijim Pay support with receipt id prefilled |

- **Modals/drawers/sheets opened**: none.
- **States**: **not found** → C-21 variant copy FR "Reçu introuvable. Méfiez-vous des faux reçus : un vrai reçu s'ouvre toujours sur pay.ijimpay.com." / EN "Receipt not found. Beware of fake receipts: a genuine receipt always opens on pay.ijimpay.com."; loading: server-rendered, none.
- **Events/notifications**: `receipt_viewed` `{receipt_id, source: "qr"|"link"|"direct"}` (source from `?src=`) · `receipt_pdf_downloaded`.

### C-15 — Reçu PDF & impression thermique / Receipt PDF & thermal print layout
- **Route**: `/r/{receipt_id}.pdf` (A6 PDF) + 80 mm thermal template used by the merchant app POS print · **Icon**: lucide `printer` · **Access**: public (PDF) / merchant app (thermal) · **Purpose**: printable, monochrome, verifiable proof.
- **Layout zones**: single column, monochrome `ink-900` on white; line-by-line layout below is normative for BOTH A6 PDF and 80 mm thermal (thermal: 32-char monospace fallback; PDF: Inter).

```
 1  [Ijim Pay logo mono, centered, 18mm / 24 dots high]
 2  {MERCHANT NAME}                (centered, bold, uppercase)
 3  {merchant city · merchant phone}   (centered, small)
 4  --------------------------------  (rule)
 5  RECU DE PAIEMENT / PAYMENT RECEIPT (centered, small caps)
 6  --------------------------------
 7  Montant / Amount:
 8  {15 000 FCFA}                  (centered, XXL — largest line)
 9  Statut / Status:  PAYE / PAID   (or REMBOURSE / REFUNDED)
10  Date:             02 août 2026, 14:05
11  Réf. commerçant:  ORDER-1042
12  Réf. Ijim Pay:    ch_01J9XK…    (short id)
13  Opérateur:        MTN MoMo | Orange Money
14  Payeur:           6 70 •• •• 00
15  --------------------------------
16  (sessions only) 1 ligne / article: {Qté}x {Nom}   {montant}
17  --------------------------------
18  Vérifiez ce reçu / Verify this receipt:
19  [QR code, centered, ≥ 25 mm, target pay.ijimpay.com/r/{id}?src=qr]
20  pay.ijimpay.com/r/{id8}        (short URL, centered)
21  --------------------------------
22  Sécurisé par Ijim Pay          (centered, small)
```

- **Tabs**: none. · **Data displayed**: same receipt resource as C-14. · **Inputs**: none. · **Actions**: none (static document).
- **Modals/drawers/sheets opened**: none.
- **States**: refunded charges print line 9 as "REMBOURSE / REFUNDED" plus line 9b "Remboursé le / Refunded on {date}". PDF ≤ 80 KB; QR error-correction level M.
- **Events/notifications**: server logs `receipt_pdf_generated` `{receipt_id}`.

### C-16 — Affiche QR comptoir A6 / Static counter QR — A6 print layout
- **Route**: generated from dashboard/app link QR modal ("print A6 counter card", docs/08 A4); asset `qr_png_url` + A6 PDF template · **Icon**: lucide `qr-code` · **Access**: merchant prints; payer scans · **Purpose**: printed table-stand that opens the merchant's open-amount link.
- **Layout zones** (A6 105×148 mm, portrait, normative top-to-bottom):

```
 1  Header band, brand-900, 22 mm: Ijim Pay logo (white) left ·
    "Payez par MTN MoMo & Orange Money" white text right (EN version on request)
 2  {MERCHANT NAME} — 20pt bold, centered, ink-900
 3  "Scannez pour payer" / "Scan to pay" — 14pt, centered
 4  QR code — 62×62 mm, centered, quiet zone 4 modules,
    target: pay.ijimpay.com/l/{slug}?src=counter_qr
 5  Fallback line: "ou tapez : pay.ijimpay.com/l/{slug}" — 10pt monospace, centered
 6  ChannelChips row — MTN + Orange chips with labels, centered, 8 mm
 7  Footer, paper #F7F9F8 band: lock glyph + "Sécurisé par Ijim Pay" — 8pt centered
```

- **Tabs**: none. · **Data displayed**: `payment_link.slug`, merchant name, `qr_png_url` (docs/02 §2.2). · **Inputs**: none. · **Actions**: none (print asset). · **Modals**: none.
- **States**: monochrome printer variant (header band becomes black rule); minimum print size enforced A6 — QR never below 40 mm.
- **Events/notifications**: scans arrive as `link_viewed` `{source:"counter_qr"}`.

### C-17 — Mode dégradé navigateur intégré / In-app-browser degraded mode
- **Route**: any landing when UA sniff detects Facebook/Instagram/WhatsApp/Messenger webview AND a degraded capability (blocked `tel:` links, blocked downloads, or forced viewport) · **Icon**: lucide `triangle-alert` (`info-600` styling — informational, not error) · **Access**: public · **Purpose**: keep the payment possible inside the webview while offering escape to a real browser.
- **Layout zones**: dismissible top banner injected above the normal landing content (landing otherwise unchanged) · rest = the underlying screen.
- **Tabs**: none.
- **Data displayed**: detected app name when known (FR "Vous êtes dans le navigateur de WhatsApp").
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir dans le navigateur (EN "Open in browser") | `external-link` *(icon addition)* | public | Android intent `intent://…#Intent;scheme=https;end` → Chrome; iOS: instruction sheet "Touchez ⋯ puis « Ouvrir dans Safari »" |
| Copier le lien (EN "Copy the link") | `copy` | public | clipboard + toast FR "Lien copié" / EN "Link copied" |
| Continuer quand même (EN "Continue anyway") | — | public | dismisses banner (session-scoped), stays in webview |

- **Modals/drawers/sheets opened**: none.
- **States**: banner copy — FR "**Pour une meilleure expérience, ouvrez cette page dans votre navigateur.** Le paiement fonctionne aussi ici, mais la numérotation \*126# peut être bloquée." / EN "**For a better experience, open this page in your browser.** Payment also works here, but dialing \*126# may be blocked." Degradations applied while in webview: USSD codes rendered as copyable text (`copy` button) instead of `tel:` links; PDF download replaced by "open receipt link" instruction; auto-redirects (C-10 sessions) always paired with a visible manual button.
- **Events/notifications**: `inapp_browser_detected` `{app, capabilities:{tel:bool,download:bool}}` · `open_in_browser_clicked` · `inapp_continue_anyway`.

### C-18 — Widget : chargeur / Widget loader
- **Route**: none (merchant page; `@ijimpay/checkout` script ≤ 30 KB, `IjimPay.open({publishableKey, amount|linkSlug, onSuccess, onClose})`) · **Icon**: — · **Access**: public, publishable key (`pk_…`, may only create checkout sessions per docs/02 §1) · **Purpose**: instant perceived response between the merchant's button click and the iframe being ready.
- **Layout zones**: full-viewport overlay `rgba(16,24,40,0.6)` · centered white card 88×88px with brand spinner (CSS) · below: FR "Ouverture du paiement sécurisé…" / EN "Opening secure payment…" (white, 14px).
- **Tabs**: none. · **Data displayed**: none. · **Inputs**: none. · **Actions**: none (Escape/backdrop inert during load; max 8 s before fallback).
- **Modals/drawers/sheets opened**: transitions to C-19.
- **States**: **timeout/failure** (8 s or iframe `onerror` or CSP frame-ancestors block) → C-20; script loaded but `pk_` invalid → widget calls `onClose({error:"invalid_key"})` and shows nothing payer-facing (merchant console error only).
- **Events/notifications**: `widget_opened` `{merchant_origin, mode}` · `widget_load_failed` `{reason}`.

### C-19 — Widget : modal iframe / Widget modal
- **Route**: iframe `https://pay.ijimpay.com/c/{session_id}?widget=1` inside overlay · **Icon**: — · **Access**: public · **Purpose**: the full hosted flow (C-04 → C-08 → C-09 → terminals) without leaving the merchant page.
- **Layout zones**: dimmed backdrop · sheet: mobile = full-screen; ≥ 768px = centered card 420×min(720px, 92vh), 16px radius · close button `x` *(icon addition)* top-right inside the frame chrome · iframe content renders the standard screens with footer link suppressed and back-to-merchant actions replaced by "close".
- **Tabs**: none.
- **Data displayed**: whatever inner screen shows; postMessage events to parent: `ijimpay:success {charge_id, signature}` · `ijimpay:failure {failure_code}` · `ijimpay:closed`.
- **Inputs**: none at this layer (inner screens own their inputs).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Fermer (aria-label "Fermer le paiement" / EN "Close payment") | `x` | public | if charge `pending` → inner CM-04 confirm first; else close overlay, postMessage `ijimpay:closed`, fire `onClose` |

- **Modals/drawers/sheets opened**: inner screens' modals (CM-01…CM-06).
- **States**: backdrop click = same as close button; Escape = close (same confirm rule); focus trapped inside modal, returned to the triggering button on close (a11y); iframe navigation locked to pay.ijimpay.com.
- **Events/notifications**: `widget_session_started` · funnel events from inner screens carry `{surface:"widget"}` · `widget_closed` `{stage, charge_status}`.

### C-20 — Widget : iframe bloqué / Widget blocked-iframe fallback
- **Route**: rendered by the loader script in place of the modal when the iframe cannot load · **Icon**: lucide `external-link` · **Access**: public · **Purpose**: never lose the sale — degrade to full-page redirect.
- **Layout zones**: small centered card on the overlay: icon · one line of copy · one CTA.
- **Tabs**: none. · **Data displayed**: none. · **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Continuer vers la page de paiement (EN "Continue to the payment page") | `external-link` | public | `window.location = https://pay.ijimpay.com/c/{session_id}`; return via `success_url`/`cancel_url` |

- **Modals/drawers/sheets opened**: none.
- **States**: copy — FR "Impossible d'ouvrir le paiement ici. Continuez sur notre page sécurisée." / EN "Couldn't open the payment here. Continue on our secure page." Auto-redirect after 4 s if no tap; card is the state.
- **Events/notifications**: `widget_fallback_redirect` `{reason:"iframe_blocked"|"timeout"}`.

### C-21 — Page introuvable / Not found (404)
- **Route**: any unknown slug/id · **Icon**: lucide `search` (`ink-500`) · **Access**: public · **Purpose**: safe dead-end that also protects against typo-phishing.
- **Layout zones**: neutral header (Ijim Pay logo only — no merchant, we don't know one) · icon + heading + body · footer.
- **Tabs**: none. · **Data displayed**: none. · **Inputs**: none. · **Actions**: none (no destination we can trust). · **Modals**: none.
- **States**: page is the state. Copy — FR "**Page introuvable**" · body "Ce lien de paiement n'existe pas. Vérifiez l'adresse reçue, ou demandez au commerçant de renvoyer le lien." / EN "**Page not found**" · "This payment link doesn't exist. Check the address you received, or ask the merchant to resend the link." Zero-JS, HTTP 404.
- **Events/notifications**: `not_found_viewed` `{path_shape:"l"|"c"|"pay"|"r"|"other"}`.

### C-22 — Erreur ou maintenance / Server error & maintenance
- **Route**: any route on 5xx or maintenance window · **Icon**: lucide `triangle-alert` (`warning-600`) · **Access**: public · **Purpose**: honest failure with a support handle; never blame the payer.
- **Layout zones**: neutral header · icon + heading + body + request id line (`ink-500`, 12px) · action · footer.
- **Tabs**: none. · **Data displayed**: `request_id` when available: FR "Code d'assistance : req_8fK2…" / EN "Support code: req_8fK2…". · **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer (EN "Try again") | `repeat` | public | full reload of the intended route |

- **Modals/drawers/sheets opened**: none.
- **States**: **5xx** — FR "**Un problème de notre côté**" · body "Nos serveurs rencontrent un souci. Votre argent n'a pas été débité tant que vous n'avez pas confirmé sur votre téléphone. Réessayez dans un instant." / EN "**A problem on our side**" · "Our servers hit a snag. No money leaves your account until you confirm on your phone. Try again in a moment." · **maintenance** — FR "**Maintenance en cours**" · "Nous revenons dans quelques minutes." / EN "**Maintenance in progress**" · "We'll be back in a few minutes." Zero-JS, HTTP 503 with Retry-After.
- **Events/notifications**: `server_error_viewed` `{status, request_id}` (client-side beacon when JS available).

---

# Modals & Drawers/Sheets

All are bottom sheets on mobile (< 768px) and centered modals otherwise; backdrop `rgba(16,24,40,0.6)`; focus-trapped, Escape + backdrop close (except where a confirm is pending); close buttons are 48px targets. Money-adjacent confirms restate amount + destination (design system §5); confirm buttons never default-focused.

### CM-01 — Changer d'opérateur / Change channel sheet
- Opened from: C-08 (« Changer »), C-11.
- Content: title FR "Choisissez votre opérateur" / EN "Choose your operator" · two radio rows (48px each): ChannelChip MTN `#FFCC00` + "MTN Mobile Money" · ChannelChip Orange `#FF7900` + "Orange Money"; selected row = `brand-100` background + `circle-check`; channels the merchant doesn't accept or that are `down` per `GET /v1/channels` are disabled with FR "Indisponible" / EN "Unavailable" label (`triangle-alert`).
- Inputs: none (radio selection is the action).
- Actions: **Sélectionner MTN Mobile Money** (radio → applies + closes) · **Sélectionner Orange Money** (idem) · **Annuler** / EN "Cancel" (closes, no change).
- Confirm rules: none — non-destructive.
- Events: `channel_changed` `{from, to, reason:"manual"}`.

### CM-02 — Changer de numéro / Change number sheet
- Opened from: C-09 (at 2 min).
- Content: title FR "Changer de numéro" / EN "Change number" · warning line FR "La demande en cours sur le 6 70 00 00 00 sera annulée." / EN "The pending request on 6 70 00 00 00 will be cancelled." · phone field (identical spec + validation + error copy as C-08 input) · channel auto-detect zone (same as C-08).
- Inputs:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nouveau numéro | tel, +237 fixe, format national | identique à C-08 | vide | identiques à C-08 |

- Actions: **Confirmer le nouveau numéro** / EN "Confirm new number" (cancels pending charge, creates new one with new phone — API additions; → C-09 reset) · **Annuler** / EN "Cancel" (closes; pending charge untouched).
- Confirm rules: primary button disabled until valid; restates FR "Une nouvelle demande sera envoyée au {numéro}." / EN "A new request will be sent to {number}."
- Events: `number_changed_midflight` `{charge_id_old, charge_id_new}`.

### CM-03 — Renvoyer la demande / Resend request confirm
- Opened from: C-09 (« Renvoyer la demande », once only).
- Content: title FR "Renvoyer la demande ?" / EN "Resend the request?" · body FR "L'ancienne demande sera annulée et une nouvelle sera envoyée au 6 70 00 00 00. Vous ne pouvez renvoyer qu'une seule fois." / EN "The old request will be cancelled and a new one sent to 6 70 00 00 00. You can only resend once." · amount restated "Montant : 15 000 FCFA".
- Inputs: none.
- Actions: **Renvoyer** / EN "Resend" (`repeat`) — cancel + recreate charge, C-09 countdown resets, button permanently removed after use · **Annuler** / EN "Cancel".
- Confirm rules: cancel button default-focused (money-adjacent).
- Events: `charge_resent` `{attempt:2}`.

### CM-04 — Annuler le paiement / Cancel payment confirm (sessions)
- Opened from: C-04 (« Retour à {merchant} »), C-09 (sessions), C-19 close while pending.
- Content: title FR "Annuler le paiement ?" / EN "Cancel the payment?" · body FR "Votre commande chez {merchant} ne sera pas payée." / EN "Your order with {merchant} will not be paid." · if a charge is `pending`, adds FR "Si vous avez déjà validé sur votre téléphone, le paiement peut quand même aboutir — vous recevrez un reçu." / EN "If you already approved on your phone, the payment may still complete — you will receive a receipt."
- Inputs: none.
- Actions: **Continuer le paiement** / EN "Keep paying" (primary, default-focused; closes sheet) · **Oui, annuler** / EN "Yes, cancel" (secondary-destructive outline; redirect `cancel_url` or postMessage `ijimpay:closed`; session marked abandoned server-side → merchant webhook).
- Confirm rules: destructive is never primary/default.
- Events: `checkout_canceled` `{stage, charge_status}`.

### CM-05 — Contacter le commerçant / Contact merchant sheet
- Opened from: C-05, C-06, C-09, C-11, C-12, C-13.
- Content: title FR "Contacter {merchant}" / EN "Contact {merchant}" · rows built from merchant support config (see API additions); rows absent when not configured; when nothing configured the opener link is hidden entirely.
- Inputs: none.
- Actions: **WhatsApp** (`message-circle`) — `wa.me/{phone}` with prefilled FR "Bonjour, j'ai une question sur mon paiement {reference} de {amount} FCFA." · **Appeler** / EN "Call" (`message-square` is SMS — calling uses `phone` *(icon addition)*) — `tel:` link (copyable text in webviews per C-17) · **Copier le numéro** / EN "Copy number" (`copy`) · **Fermer** / EN "Close".
- Confirm rules: none.
- Events: `merchant_contact_opened` `{from_screen, method}`.

### CM-06 — Détail de la commande / Order details sheet
- Opened from: C-04 accordion tap on mobile.
- Content: title FR "Votre commande" / EN "Your order" · items table: columns Article | Qté | Montant (tabular nums, right-aligned amounts) · subtotal row · fee row if passed on ("Frais") · bold total row "Total : 42 000 FCFA".
- Inputs: none.
- Actions: **Fermer** / EN "Close" (sheet dismiss; swipe-down also closes).
- Confirm rules: none.
- Events: `order_summary_opened`.

---

# Analytics — canonical event dictionary

Common properties on every event: `merchant_id, mode(test|live), surface(hosted|widget), lang, in_app_browser, device_class, session_ulid`. Funnel spine: `link_viewed → amount_entered → phone_submitted → charge_created → succeeded|failed|expired|abandoned` (abandoned = server-derived timeout). Screen-specific events are listed in each screen's "Events" line; full list: `link_viewed, cta_pay_clicked, amount_entered, amount_validation_error, quantity_changed, order_summary_opened, phone_submitted, channel_changed, charge_created, charge_create_failed, waiting_viewed, waiting_45s_reached, waiting_2min_reached, charge_resent, number_changed_midflight, status_result, succeeded, failed, expired, retry_clicked, rate_limited_viewed, rate_limited_retry, receipt_viewed, receipt_downloaded, receipt_pdf_downloaded, redirected_to_merchant, checkout_canceled, merchant_contact_opened, inapp_browser_detected, open_in_browser_clicked, inapp_continue_anyway, widget_opened, widget_load_failed, widget_session_started, widget_closed, widget_fallback_redirect, not_found_viewed, server_error_viewed`. These power the link-conversion stat in dashboard (`docs/08` A4) and app.

# API additions needed (not in docs/02 — never invented silently inline)

| Method + path | Purpose |
|---|---|
| `GET /v1/public/links/{slug}` | Public link resolution for landings (title, amount config, catalog fields, merchant display profile, accepted channels, support contact, state active/expired/paid) |
| `GET /v1/public/checkout_sessions/{id}` | Public session resolution (cart snapshot, urls, state) |
| `POST /v1/public/charges` | Payer-side charge creation from hosted page (server-brokered; publishable-key/session scoped) |
| `GET /v1/public/charges/{id}` | Public charge status poll (status, failure_code, expires_at only) |
| `GET /v1/public/charges/{id}/stream` | SSE status stream (progressive enhancement over polling) |
| `POST /v1/public/charges/{id}/resend` | Cancel + recreate (CM-02/CM-03; once per charge) |
| `GET /v1/public/receipts/{receipt_id}` | Receipt resource for C-14 |
| `GET /v1/public/receipts/{receipt_id}/pdf` | Receipt PDF (C-15) |
| — plus fields | `payment_link`: catalog fields (`product_name,image_url,unit_price,max_quantity`), `min_amount` for open links; merchant public profile: `support_phone, support_whatsapp, fee_passthrough:bool, accepted_channels[]` |

# Icon additions needed (not in docs/07 map)

`minus-circle` (quantity decrement — pairs with existing `plus-circle`) · `chevron-down` (accordion) · `external-link` (open in browser / fallback redirect) · `x` (widget modal close) · `phone` (call merchant in CM-05).

```
INVENTORY: pages=22 tabs=0 modals=6 forms=4 tables=2 actions=54
```
