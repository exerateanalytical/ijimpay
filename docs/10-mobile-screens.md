# IjimPay — Mobile App: Every Screen, Every Flow, Every Action

> **Superseded for detail**: fully expanded in [docs/15](15-mobile-app-screens-spec.md). This doc remains the overview.

Expands `04-mobile-app.md` into a full screen inventory. Icons = Lucide (24px). Bottom nav: Accueil `layout-dashboard` · Liens `link` · **Encaisser `hand-coins`** (raised center) · Activité `arrow-left-right` · Menu `settings`. All flows FR-first; strings shown are the actual v1 copy draft.

---

## Screen inventory (route-style ids)

```
AUTH:   splash · welcome · phone-entry · otp · create-pin · biometric-optin · login-pin
ONBRD:  business-info · doc-capture (xN) · settlement-account · verification-status
HOME:   home · notifications · provider-status-detail
CHARGE: amount-pad · channel-payer · charge-waiting · charge-success · charge-failed · qr-display
LINKS:  links-list · link-create · link-created(share) · link-detail · link-qr
ACT:    activity-list · tx-detail · receipt-preview/share
PAYOUT: payouts-home · approval-detail · approval-confirm · beneficiary-list/add · payout-single · topup-instructions
MENU:   menu · balance-detail · settlements-list/detail · team-list/invite · business-profile ·
        verification-docs · security(pin/biometric/devices) · notifications-prefs · language · help/support · about
GLOBAL: offline-banner · force-update · maintenance · error-sheet
```

## 1. Auth & first run

| Screen | Layout / actions | States & rules |
|---|---|---|
| Splash → Welcome | logo, tagline "Encaissez, simplement.", [Créer un compte] [Se connecter], language `globe` top-right | force-update screen if app version below minimum |
| Phone entry | `+237` fixed, big numeric field, ToS/privacy links | invalid prefix error |
| OTP | 6-digit auto-read (SMS Retriever), resend 30s countdown | wrong/expired; 5 attempts → cooldown |
| Create PIN | 4-digit, twice; explainer "Votre PIN protège l'application" | — |
| Biometric opt-in | `fingerprint` illustration, [Activer] [Plus tard] | skipped → PIN only |
| Login (returning) | avatar + name, PIN pad or biometric prompt; "changer de compte" | 5 wrong PINs → OTP re-auth; session JWT refresh silent |

Onboarding (new merchant): business-info form → guided doc capture (frame overlay, blur/glare detection, retake loop, per-doc checklist RCCM/ID/proof) → settlement account → **verification-status** screen (tier badge, per-doc status chips, "En attente d'examen ~24h", push on decision). User lands in app in **test mode** immediately: amber banner "Mode test" with switch-to-live CTA once Tier 1 approved.

## 2. Home (Accueil)

- Header: merchant name + switcher (if multi), `bell` with badge, test/live pill.
- Hero card: **Encaissé aujourd'hui** amount XL, tx count, mini success-rate ring; tap → Activity filtered today.
- Balance row: Disponible + En attente (tap → balance-detail). Provider strip: MTN ●, Orange ● (tap → provider-status-detail with plain-language status + history).
- Quick actions: Encaisser (primary) · Créer un lien · Voir le QR du comptoir (merchant's static QR fullscreen for scanning).
- Pending-approval card (approver roles): "2 lots de paiement attendent votre validation →".
- Latest 5 TxRows. Pull-to-refresh; skeletons; cached render < 0.5s offline (`cloud-off` chip "données de 14:02").
- Notifications screen: grouped by day, deep-link per item; mark-all-read.

## 3. Encaisser (charge at counter) — the core loop, target < 15 s

```
[amount-pad] full-screen numpad, live-formatted "12 500 FCFA", recent amounts chips
   → [channel-payer]  ── entered phone → auto ChannelChip + [Envoyer la demande]
                      └─ [Afficher le QR] → qr-display (dynamic QR for this amount)
   → [charge-waiting] pulsing clock, payer number, per-channel USSD instruction card,
                      expiry ring, progressive copy @45s/@2min, [Renvoyer] [Annuler]
   → [charge-success] full-screen brand flash + ✓ + SOUND + haptic, amount XL,
                      [Partager le reçu `share-2`] [Nouveau encaissement] — auto-returns to amount-pad in 5s
   or [charge-failed] reason in plain FR + [Réessayer] [Changer de canal] [QR à la place]
```

Rules: phone field remembers recent payers (local, opt-out in settings); provider-down channel is disabled with tooltip; double-submit locked; charge created offline is refused with clear message (money actions require network — only *reads* are offline-tolerant); success screen reachable from notification even if app was closed mid-wait.
qr-display: brightness auto-max, amount huge above QR, "Le client scanne avec son appareil photo", cancel returns to channel-payer.

## 4. Liens (Payment links)

- links-list: cards (title, amount/`libre`, collected total, paid count, active dot); search; FAB `plus`. Row swipe/long-press: partager, QR, désactiver.
- link-create (sheet, 1 screen): title, montant fixe/libre toggle, description optional, réutilisable toggle, [Créer]. → link-created: URL big, [WhatsApp] [SMS] [Copier `copy`] [QR], prefilled FR share text "Payez {title} ici en toute sécurité 👉 {url}".
- link-detail: stats (collecté, vues→payés conversion), tx list for link, edit, deactivate (confirm sheet). link-qr: fullscreen + [Imprimer/PDF A6] via share sheet.
- Empty: "Créez votre premier lien en 30 secondes" + button.

## 5. Activité (Transactions)

- Segmented filter: Tout · Réussis · En attente · Échoués; date chips (Aujourd'hui/7j/Mois/Perso); channel filter; search (phone/ref). Sticky day headers with day totals.
- tx-detail (sheet): amount + StatusBadge + ChannelChip, timeline, customer, frais/net breakdown, link/invoice origin, actions: **Partager le reçu** (receipt-preview → image/PDF to WhatsApp/print `printer` v1.2), **Rembourser** (Finance/Admin: amount, reason, PIN/biometric confirm), Copier ref, Signaler un problème (support with tx context prefilled).
- Pending items live-update; failed show human reason; export month CSV (menu → email it).

## 6. Paiements sortants (Payouts) — v1.1

- payouts-home: wallet balance card + [Approvisionner] · tabs Validations (badge) / Historique / Bénéficiaires.
- **approval-detail**: batch summary (total XL, count, créé par, motif), scrollable item list, anomaly hints ("⚠ 2 nouveaux bénéficiaires", "montant 3× supérieur au lot habituel") → [Approuver `check-check`] / [Rejeter + raison]. approval-confirm: restates "Approuver 4 250 000 FCFA vers 50 bénéficiaires ?" → **biometric/PIN required**, enrolled devices only; success screen with batch tracking link.
- payout-single: beneficiary pick/add (phone, name, channel auto), amount, motif, confirm (restate + biometric). Blocked states: solde insuffisant (→ top-up), provider down, role lacking permission (hidden entirely).
- topup-instructions: amount intent → reference code + exact USSD steps per channel; screen auto-updates to success when top-up matched (push + confetti-free ✓).
- beneficiaries: list w/ verified-name `badge-check`, add/edit/delete (confirm), payroll groups read-only v1.1 (managed on web).

## 7. Menu

balance-detail (3 balances + ledger rows) · settlements (list → detail → share PDF) · team (list, invite via contact picker + role sheet w/ plain-language role descriptions, remove w/ confirm) · business-profile (logo, name — logo shows on checkout) · verification-docs (status, re-upload rejected) · security (change PIN, biometric toggle, **appareils** list with revoke, active sessions) · notifications-prefs (matrix from doc 08§C, per-event toggles, sound on/off) · language FR/EN · help (WhatsApp support deep-link, FAQ, tutoriels vidéo) · about (version, ToS, licences) · se déconnecter `log-out` (confirm).

## 8. Global behaviors

- **Offline**: top banner `cloud-off` "Hors ligne — affichage des dernières données"; reads cached (Drift); creates queued ONLY for non-money items (link creation queues; charges/payouts never queue). Queue indicator + auto-flush toast "1 lien créé hors ligne a été publié".
- **Errors**: single error-sheet component — icon, plain FR sentence, request_id small, [Réessayer] [Contacter le support]. Never raw codes.
- **Deep links / notifications routing**: payment success → tx-detail; approval request → approval-detail; KYB decision → verification-status; settlement → settlement-detail. All survive cold start.
- **FLAG_SECURE** on: balance, payout, security screens. Root check on launch → warning + payouts disabled.
- **Empty/loading**: every list has skeleton + designed empty state (see design backlog 07§9).
- Accessibility: TalkBack labels on all actions, min 48dp targets, font-scale up to 1.3 without breakage.

## 9. Traceability — PRD coverage map

| PRD §9 requirement | Screens |
|---|---|
| Accept payment at counter | amount-pad → charge-waiting → success (+ qr-display) |
| Instant sound + push | charge-success, FCM high-priority channel |
| Today's totals | home hero |
| Link creation/sharing | link-create → link-created |
| Payout approvals | approval-detail → approval-confirm |
| Balance view | home balance row, balance-detail |
| Offline tolerance | §8 offline rules |
| FR/EN | language screen, ICU strings |
