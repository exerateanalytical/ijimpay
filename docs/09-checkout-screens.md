# IjimPay — Hosted Checkout, Payment Links & Receipts: Every Screen & State

> **Superseded for detail**: fully expanded in [docs/14](14-checkout-pages-spec.md). This doc remains the overview.

Customer-facing surface (`pay.ijimpay.com`). The payer is NOT our user — assume a low-end Android phone, 3G, possibly first time paying online. One column, max 420px, FR default with EN toggle (`globe`), merchant logo + name always visible, "Sécurisé par IjimPay `lock`" footer on every screen. Page weight < 150 KB; no JS required for the read-only states (receipt, expired).

---

## Routes

```
/l/{slug}          payment link page (reusable or single-use)
/c/{session_id}    checkout session (e-commerce redirect)
/pay/{charge_id}   in-flight charge status page (also dunning/QR target)
/r/{receipt_id}    public receipt / verification page
```

## Flow (master state diagram)

```
[Landing /l or /c]
   → (open amount? enter amount) → [Phone & channel]
   → POST charge → [Waiting for approval]
        ├─ succeeded → [Success + receipt]
        ├─ failed    → [Failure + retry]
        └─ expired   → [Expired + retry]
Abandon at any point → link stays reusable / session marked abandoned (webhook to merchant)
```

## Screen specs

### 1. Landing — payment link `/l/{slug}`
- Content: merchant logo/name · link title · description/image (if set) · **amount hero** (fixed) or amount input (open: numpad-friendly `inputmode=numeric`, min hint, FCFA suffix fixed) · quantity stepper (catalog links) · CTA **"Payer {amount}"**.
- States: **inactive/deactivated** ("Ce lien n'est plus actif" + merchant contact if provided) · **expired** (`timer-off`) · **single-use already paid** (shows receipt link) · amount-below-min inline error.

### 2. Landing — checkout session `/c/{id}`
- Same skeleton + order summary accordion (items, total) + "Retour à {merchant}" cancel link (→ `cancel_url`). Session expired state → "Retournez à la boutique pour recommencer".

### 3. Phone & channel
- Phone field: `+237` fixed prefix, national formatting as-you-type, large font. Channel auto-detected from prefix → shows detected ChannelChip ("Numéro MTN détecté") with **"Changer"** override revealing both chips (MTN `#FFCC00`, Orange `#FF7900`, radio behavior). Unknown prefix → manual pick required.
- Fee display: if merchant passes fees on (config), show "dont frais : X FCFA" line; otherwise silent.
- CTA "Confirmer le paiement" → creates charge. Errors inline: invalid number, `channel_unavailable` (provider down → suggest the other channel if payer has one: "Orange Money est indisponible actuellement"), rate-limited (3 pending per phone: "Une demande est déjà en cours sur ce numéro — validez-la ou attendez 5 min").

### 4. Waiting for approval `/pay/{charge_id}` — the make-or-break screen
- Hero: pulsing `clock` in brand-100 circle · amount · "**Une demande de paiement a été envoyée au 6 70 00 00 00**".
- Instruction card per channel (exact, tested copy):
  - MTN: "Validez la notification sur votre téléphone, ou composez **\*126\*1#** puis confirmez avec votre code MoMo."
  - Orange: "Composez **\*150\*4\*4#** pour approuver, puis confirmez avec votre code Orange Money." *(USSD codes verified with telcos before launch — single source of truth in config, not hardcoded.)*
- Live status via 3s polling (SSE if available); countdown ring to expiry (default 5 min).
- Progressive copy: at 45s "Toujours en attente — vérifiez vos notifications" · at 2 min adds **"Renvoyer la demande"** (allowed once, cancels + recreates charge) and **"Changer de numéro"**.
- Never dead-ends: "Un problème ? Contactez {merchant}" link.

### 5. Success
- Signature success moment (check scale-in, brand flash — no sound on web). Amount, merchant, ref, date.
- Actions: **Télécharger le reçu** `download` · voir le reçu `/r/{id}` · e-commerce: auto-redirect to `success_url` after 3s with manual "Retourner à {merchant}" button (redirect carries `charge_id` + signature params for merchant verification).

### 6. Failure
- `circle-x` danger tint + human reason mapping: `insufficient_payer_funds` → "Solde insuffisant sur ce compte" · `payer_rejected` → "Le paiement a été refusé" · `payer_timeout`/`expired` → "La demande a expiré" · `payer_not_found` → "Ce numéro n'a pas de compte {channel}" · `provider_error` → "Erreur chez l'opérateur — réessayez".
- Actions: **Réessayer** (same details, new charge) · Changer de numéro/canal · back to merchant (cancel_url on sessions).

### 7. Receipt `/r/{receipt_id}` (public, no auth, unguessable id)
- Verified banner "✔ Paiement vérifié par IjimPay" · merchant, amount, status, ref, date, channel · PDF download · anti-fraud note "Vérifiez toujours vos reçus sur pay.ijimpay.com".
- This page is the QR target printed on thermal receipts — it's our anti-fake-receipt story for market merchants.

### 8. Embeddable widget (`@ijimpay/checkout`)
- `IjimPay.open({ publishableKey, amount|linkSlug, onSuccess, onClose })` → full-screen iframe modal rendering the same screens; postMessage events `success|failure|closed`; < 30 KB loader. Fallback: plain redirect if iframe blocked.

## Static QR flow (counter)
Merchant's printed static QR → `/l/{slug}` open-amount link → payer enters amount themselves → same flow. Success screen instructs "Montrez cet écran au commerçant" + merchant simultaneously gets app push (that's the real confirmation — copy on merchant materials: "attendez la sonnerie, pas l'écran du client").

## Analytics (per screen)
Funnel events: `link_viewed → amount_entered → phone_submitted → charge_created → succeeded|failed|abandoned` with channel + failure_code dimensions; powers the link conversion stat in dashboard and app.

## Edge cases checklist
- [ ] Payer phone = merchant's own number (allowed in live, flagged in risk rules; blocked in obvious self-dealing patterns)
- [ ] Double-tap on "Payer" → idempotent (client-side lock + server idempotency key)
- [ ] Link opened in Facebook/WhatsApp in-app browsers (test matrix — majority of traffic will be in-app webviews)
- [ ] Payer on MTN opening an Orange-only merchant config → clear "moyens acceptés" messaging on landing
- [ ] Charge succeeds after page abandoned → receipt still reachable, merchant webhook unaffected
- [ ] RTL not needed; but test FR diacritics on cheap-device system fonts
