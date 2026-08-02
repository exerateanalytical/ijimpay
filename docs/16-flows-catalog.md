# Ijim Pay — Catalogue exhaustif des parcours / End-to-End Flow Catalog

Status: Draft v1 · Complements docs 01–03 (contract), 07 (design system), 08–10 (screens).
Conventions: amounts `12 500 FCFA` · FR-first copy · routes per docs/08 (dashboard `app.ijimpay.com`), docs/09 (checkout `pay.ijimpay.com`), docs/10 (app screen slugs), ops per docs/08 §B (`ops.ijimpay.com`). API paths per docs/02; anything missing is in **API additions needed** at the end (never invented inline — flows reference `[API-ADD-n]`). StatusBadge statuses are only the 8 canonical ones (`succeeded, pending, failed, expired, refunded, processing, pending_approval, draft`). `pending_approval` renders warning-600 **outlined** (collision rule 07 §2.2 — never accent-500). Notifications reference Appendix A (`N-xx`); events reference docs/02 §2.7 types. Ledger entries per docs/03 §3 chart of accounts.

Actor groups: **Merchant** F-001–F-034 · **Payer** F-035–F-044 · **Ops** F-045–F-054 · **System** F-055–F-065 · **Developer** F-066–F-069.

---

## A. Merchant flows (web dashboard + mobile app)

### F-001 — Inscription marchand / Merchant signup
- **Actor(s)**: prospective merchant (any role → becomes Owner) · **Trigger**: taps "Créer un compte" (web `/signup`, app `welcome`) · **Preconditions**: none · **Frequency**: ~50/day at scale.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Marchand | Saisit téléphone `+237 6 70 00 00 00` (ou email sur web), accepte CGU | Validates E.164 + CM prefix; sends OTP `[API-ADD-1]` | `/signup` · app `phone-entry` |
| 2 | Marchand | Saisit OTP 6 chiffres | Verifies (5 attempts, 10 min TTL); creates `users` row | `/verify-otp` · app `otp` |
| 3 | Marchand | Définit mot de passe (web) / PIN 4 chiffres ×2 (app) | Stores `password_hash`; app stores PIN-derived key locally + device enrolled (F-031) | signup step · app `create-pin` |
| 4 | Marchand | Saisit nom commercial | Creates `merchants` (tier 0, status `pending`, kyb_status `not_started`), `memberships` role `owner`; issues `sk_test_`/`pk_test_` keys | signup step 3 |
| 5 | — | — | Lands in dashboard/app in **test mode**, amber `flask-conical` banner "Mode test — aucun argent réel", setup checklist card | `/` (Home) · app `home` |

- **Failure branches**: invalid prefix → inline "Numéro camerounais invalide (ex : 6 70 00 00 00)"; OTP wrong → "Code incorrect — {n} essais restants"; OTP expired → "Code expiré — renvoyez un code"; 5 wrong OTPs → 15 min cooldown "Trop d'essais — réessayez dans 15 minutes"; phone already registered → "Ce numéro a déjà un compte — connectez-vous" with login link; SMS delivery failure → resend after 30 s cooldown, after 3 resends offer voice call fallback.
- **Postconditions**: user + merchant (tier 0) + owner membership + test API keys exist. No ledger movement.
- **Events emitted**: none (internal audit_log `merchant.created`). **Notifications**: N-01 (welcome email), N-02 (OTP SMS).

### F-002 — Connexion (avec 2FA) / Login incl. TOTP
- **Actor(s)**: any merchant user · **Trigger**: visits `/login` / opens app · **Preconditions**: account exists · **Frequency**: thousands/day.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Utilisateur | Téléphone/email + mot de passe (web) ou PIN/biométrie (app) | Verifies credentials `[API-ADD-1]`; rate-limits 5/min/account | `/login` · app `login-pin` |
| 2 | Utilisateur | Si TOTP inscrit : saisit code 6 chiffres | Verifies `totp_secret` window ±1 step | `/login` (2FA step) |
| 3 | — | — | Session JWT issued; if new device/browser → security alert; lands on Home | `/` · app `home` |

- **Failure branches**: bad credentials → "Identifiants incorrects" (never says which field); rate-limited → "Trop de tentatives — réessayez dans 5 minutes"; wrong TOTP → "Code d'authentification incorrect"; 5 wrong PINs (app) → forced OTP re-auth (F-004); suspended merchant → "Compte suspendu — contactez le support" with `life-buoy` link; new device detected → login proceeds + N-20.
- **Postconditions**: active session; audit_log `user.login`. **Events**: none. **Notifications**: N-20 (new device) when applicable.

### F-003 — Mot de passe oublié / Forgot password
- **Actor(s)**: merchant user · **Trigger**: "Mot de passe oublié ?" on `/forgot-password` · **Preconditions**: account exists · **Frequency**: dozens/day.
- **Steps**: 1) enters phone/email → OTP sent `[API-ADD-1]` (N-02) → 2) enters OTP → 3) sets new password (min 8 chars, 1 digit; live strength meter) → 4) all other sessions revoked, redirected to `/login` with toast "Mot de passe modifié — reconnectez-vous". Screens: `/forgot-password`.
- **Failure branches**: unknown account → same neutral success copy "Si un compte existe, un code a été envoyé" (no enumeration); OTP failures as F-001; TOTP-enrolled users must ALSO pass TOTP before reset (account-takeover defense).
- **Postconditions**: new `password_hash`, sessions revoked, audit_log entry. **Notifications**: N-02, N-21 (password changed alert push+email+SMS).

### F-004 — Réinitialisation du PIN (app) / PIN reset
- **Actor(s)**: app user · **Trigger**: "PIN oublié" on `login-pin`, or 5 wrong PINs · **Preconditions**: enrolled device · **Frequency**: dozens/week.
- **Steps**: 1) OTP to the account phone `[API-ADD-1]` → 2) verify → 3) `create-pin` ×2 → 4) biometric re-opt-in offered → back to app. Payout approvals are locked for 24 h after a PIN reset (anti-takeover, banner "Approbations désactivées pendant 24 h après réinitialisation du PIN").
- **Failure branches**: OTP failures as F-001; SIM-swap suspicion (OTP requested from new device + PIN reset within 1 h) → risk flag raised (F-052) and payouts locked pending review.
- **Postconditions**: new local PIN key; audit_log. **Notifications**: N-02, N-21.

### F-005 — Vérification KYB (onboarding) / KYB submission
- **Actor(s)**: Owner/Admin · **Trigger**: setup checklist or `/onboarding/*` wizard (app `ONBRD` screens) · **Preconditions**: F-001 done · **Frequency**: ~50/day.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Marchand | Infos entreprise : nom légal, forme juridique, secteur, adresse | Saves draft each step (resumable) `[API-ADD-2]` | `/onboarding/business` · app `business-info` |
| 2 | Marchand | Documents : RCCM, pièce d'identité du dirigeant, preuve de compte MoMo/OM/banque — drag-drop ou caméra | Uploads to `kyb_documents` (status `submitted`); app doc-capture has blur/glare detection + retake loop | `/onboarding/documents` · app `doc-capture` |
| 3 | Marchand | Compte de règlement : canal + numéro + preuve | Stores `default_settlement_channel` | `/onboarding/account` · app `settlement-account` |
| 4 | Marchand | Soumet | `kyb_status → in_review`; enters ops queue (F-045); SLA timer starts | `/onboarding/done` · app `verification-status` |

- **Failure branches**: file too large (> 10 MB) → "Fichier trop volumineux (max 10 Mo)"; unsupported format → "Formats acceptés : JPG, PNG, PDF"; upload network failure → retry with kept form state; "compléter plus tard" on docs step only → checklist item stays open with warning Banner on Home "Vérification incomplète — volume limité".
- **Postconditions**: kyb_documents rows `submitted`, merchant `kyb_status=in_review`. **Notifications**: N-03 (submission received, email + push).

### F-006 — Nouvelle soumission après rejet / KYB resubmission
- **Actor(s)**: Owner/Admin · **Trigger**: KYB rejection notification (N-05) deep-links to `/settings/verification` (app `verification-docs`) · **Preconditions**: ≥1 document `rejected` with FR reason · **Frequency**: ~30 % of submissions.
- **Steps**: 1) screen lists per-doc status chips; rejected docs show the ops reason verbatim ("Photo illisible — reprenez la pièce à plat, sans reflet") → 2) merchant re-uploads only rejected docs (`pencil`/re-upload) → 3) submit → `kyb_status → in_review` again, back of queue with "resubmission" priority boost.
- **Failure branches**: same upload failures as F-005; 3 rejections of same doc → item flagged for ops call-back, copy "Notre équipe va vous contacter au {phone}".
- **Postconditions**: replaced doc rows `submitted`; prior rows retained (audit). **Notifications**: N-03.

### F-007 — Passage en mode réel / Go-live tier upgrade
- **Actor(s)**: Owner/Admin · **Trigger**: Tier 1 approved (N-04) → CTA "Passer en mode réel" on test banner / checklist · **Preconditions**: `kyb_status=approved`, tier ≥ 1 · **Frequency**: once per merchant (+ Tier 2 requests).
- **Steps**: 1) go-live checklist modal (see F-068 content: live keys created, webhook live endpoint set, settlement account confirmed) → 2) merchant flips Test/Live toggle → live `sk_live_`/`pk_live_` revealed once (`/developers/keys`) → 3) amber banner disappears in live mode; Tier 1 volume limits shown on `/settings/verification` → 4) later: "Demander le niveau 2" button → uploads any additional docs → ops F-047.
- **Failure branches**: toggle attempted at tier 0 → blocking modal "Vérifiez votre entreprise pour encaisser en réel" with CTA to F-005; live charge attempted with test key → API 403 `authentication_failed`.
- **Postconditions**: live keys active; merchant status `active`. **Notifications**: N-04 already sent; N-06 on Tier 2 approval.

### F-008 — Encaissement au comptoir par téléphone / Counter charge by phone
- **Actor(s)**: any role except Viewer (app-first; web via Home "Encaisser" modal) · **Trigger**: customer at counter · **Preconditions**: live (or test) mode, provider operational · **Frequency**: the core loop — hundreds/merchant/day.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Marchand | Tape le montant sur le pavé (live-format "12 500 FCFA") | — | app `amount-pad` · web Home charge modal |
| 2 | Marchand | Saisit le numéro du client ; ChannelChip auto-détecté (préfixes 07 §8), surchargeable | — | app `channel-payer` |
| 3 | Marchand | "Envoyer la demande" | `POST /charges` with Idempotency-Key; charge `pending`, `expires_at = +5 min`; USSD push to payer (F-055) | → `charge-waiting` |
| 4 | Client | Approuve sur son téléphone (voir F-035 step equiv.) | Provider callback/poll → `succeeded`; ledger entry posted (F-055) | — |
| 5 | Marchand | Voit l'écran succès (flash + son + haptique), montant XL | Push N-07 fired; auto-return to amount-pad in 5 s | app `charge-success` |

- **Failure branches**: `insufficient_payer_funds` → `charge-failed` "Solde insuffisant sur ce compte" + [Réessayer] [Changer de canal] [QR à la place]; `payer_rejected` → "Le paiement a été refusé"; `payer_timeout`/expired → "La demande a expiré" + resend; `payer_not_found` → "Ce numéro n'a pas de compte {channel}"; `channel_unavailable` → channel disabled at step 2 with `triangle-alert` tooltip "Orange Money est indisponible actuellement"; rate limit 3 pending/payer → "Une demande est déjà en cours sur ce numéro — validez-la ou attendez 5 min"; offline → charge refused "Connexion requise pour encaisser" (money actions never queue); app killed mid-wait → success push N-07 deep-links to `tx-detail`.
- **Postconditions** (success, 5 000 gross / 100 fee): journal `charge_succeeded`: debit `provider_float:mtn` 5 000 · credit `merchant_pending` 4 900 · credit `platform_fees` 100.
- **Events**: `charge.succeeded` | `charge.failed` | `charge.expired`. **Notifications**: N-07 (merchant push+bell), N-30 (payer receipt SMS/WhatsApp if enabled).

### F-009 — Encaissement au comptoir par QR / Counter charge by QR
- **Actor(s)**: merchant + payer · **Trigger**: customer prefers scanning · **Preconditions**: as F-008 · **Frequency**: high.
- **Steps**: 1) `amount-pad` → 2) `channel-payer` → **[Afficher le QR]** → `qr-display` (dynamic QR encoding a single-use open link for this amount; brightness auto-max, amount huge, "Le client scanne avec son appareil photo") → 3) payer scans → lands on `/l/{slug}` prefilled amount → payer flow F-035 steps 2–4 → 4) merchant screen live-updates to `charge-success` on webhook/push. Static-QR variant: merchant shows/prints the permanent counter QR (Home "Voir le QR du comptoir") → payer enters amount themselves (F-038).
- **Failure branches**: payer scan fails (dirty camera/low light) → merchant falls back to phone entry [Réessayer par numéro]; payer abandons → QR screen cancel returns to `channel-payer`, charge (if created) expires (F-056); all payer-side failures per F-035.
- **Postconditions/Events/Notifications**: identical to F-008. Success copy on payer screen: "Montrez cet écran au commerçant" — merchant materials say "attendez la sonnerie, pas l'écran du client".

### F-010 — Créer et partager un lien / Create & share payment link
- **Actor(s)**: all roles except Viewer · **Trigger**: `/links/new`, Home quick action, app `link-create` FAB · **Preconditions**: none (works in test) · **Frequency**: daily.
- **Steps**: 1) form — titre (required, ≤ 80 chars), montant **fixe** (≥ 100 FCFA) ou **libre** (min optional), description, image (catalog SHOULD), réutilisable vs usage unique, expiration, message de succès personnalisé; live phone-frame preview (web) → 2) `POST /payment_links` → returns `url` + `qr_png_url` → 3) success screen: URL big, [WhatsApp] [SMS] [Copier `copy`] [QR `qr-code`] with prefilled FR text "Payez {title} ici en toute sécurité 👉 {url}" → 4) QR modal: download PNG/SVG, print A6 counter card. Screens: `/links/new`, `/links/:id`; app `link-create` → `link-created` → `link-qr`.
- **Failure branches**: empty title → "Donnez un titre à votre lien"; fixed amount < 100 → "Montant minimum : 100 FCFA"; expiry in past → "La date d'expiration doit être future"; offline (app) → link creation **queues** (`cloud-off`), toast on flush "1 lien créé hors ligne a été publié".
- **Postconditions**: `payment_links` row active. **Events**: none until paid. **Notifications**: none.

### F-011 — Désactiver un lien / Deactivate link
- **Actor(s)**: Admin/Owner (creators may deactivate their own) · **Trigger**: toggle on `/links` row or `link-detail` · **Steps**: confirm sheet "Désactiver « {title} » ? Les clients ne pourront plus payer via ce lien." → `POST /payment_links/{id}/deactivate` → link `active=false`; page `/l/{slug}` now renders "Ce lien n'est plus actif". Re-activation via edit `[API-ADD-3]`. · **Frequency**: weekly.
- **Failure branches**: pending charges on the link at deactivation → they complete normally (deactivation only blocks new landings).
- **Postconditions**: link inactive; stats retained. **Events/Notifications**: none.

### F-012 — Remboursement / Refund
- **Actor(s)**: Finance/Admin/Owner · **Trigger**: "Rembourser" `undo-2` in tx drawer / app `tx-detail` · **Preconditions**: charge `succeeded`, channel supports refunds (else tracked manual refund), refundable remainder > 0 · **Frequency**: ~0.5 % of charges.
- **Steps**: 1) modal — montant (≤ net minus prior refunds, default full), raison (select: "Client insatisfait", "Erreur de montant", "Doublon", "Autre" + note) → 2) confirm restates "Rembourser 4 900 FCFA à 6 70 00 00 00 ?" (approve not default-focused; app: PIN/biometric) → 3) `POST /charges/{id}/refund` with Idempotency-Key → refund object `processing` → provider disbursement → `succeeded` → charge StatusBadge `refunded` (full) or stays `succeeded` with linked refund (partial).
- **Failure branches**: amount > refundable → "Montant supérieur au remboursable (4 900 FCFA)"; insufficient `merchant_available` → "Solde disponible insuffisant pour ce remboursement"; provider failure → refund `failed`, retry button, ops notified if 3 fails; channel unsupported → "Remboursement manuel requis" flow: marks charge with manual-refund note, no money moves via us.
- **Postconditions**: journal `refund` (4 900 refund): debit `merchant_available` 4 900 · credit `provider_float:{channel}` 4 900 (fee not returned v1; stated in modal "Les frais de 100 FCFA ne sont pas remboursés").
- **Events**: `refund.succeeded`. **Notifications**: N-08 (merchant), N-31 (payer SMS).

### F-013 — Exporter les relevés / Export statements
- **Actor(s)**: Finance/Admin/Owner/Viewer (read) · **Trigger**: `download` on `/transactions`, `/balance`, `/settlements/:id`; app menu "Exporter le mois (CSV par email)" · **Frequency**: monthly peaks.
- **Steps**: 1) pick period + format CSV/PDF + rail filter → 2) ≤ 10 000 rows → immediate download; > 10 000 → async job `[API-ADD-4]`, toast "Export en cours — vous recevrez un email" → 3) email N-09 with expiring signed URL (72 h).
- **Failure branches**: empty period → export produced with header only + note "Aucune transaction sur la période"; job failure → N-09 variant "L'export a échoué — réessayez" + retry link.
- **Postconditions**: none (read-only). **Events/Notifications**: N-09.

### F-014 — Paiement sortant individuel / Single payout
- **Actor(s)**: maker Finance/Admin/Owner; approver Admin/Owner (≠ maker) if above threshold · **Trigger**: `/payouts/new` (Individuel) / app `payout-single` · **Preconditions**: wallet funded, channel operational · **Frequency**: daily.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Finance | Choisit bénéficiaire (répertoire, badge `badge-check` si nom vérifié) ou nouveau numéro + nom | Name-verify hint via provider where available | `/payouts/new` · app `payout-single` |
| 2 | Finance | Montant + motif | Funding check against `merchant_payout_wallet` | idem |
| 3 | Finance | Confirme (restate "Envoyer 250 000 FCFA à J. Fotso — 6 90 00 00 00 ?"; app biometric/PIN) | `POST /payouts` + Idempotency-Key; below threshold → `processing`; above → `pending_approval` (→ F-017) | confirm modal |
| 4 | — | — | Provider disbursement; on success payout `succeeded` | `/payouts` list |

- **Failure branches**: `insufficient_balance` → blocking banner "Solde du portefeuille insuffisant" + CTA Approvisionner (F-019); `payout_limit_exceeded` → "Limite de paiement dépassée pour votre niveau — demandez le niveau 2"; `channel_unavailable` → submit disabled with `triangle-alert`; provider failure → payout `failed` with failure_code + retry `rotate-cw`; role without permission → UI hidden entirely.
- **Postconditions** (250 000 success): during processing debit `merchant_payout_wallet` 250 000 · credit `payout_inflight` 250 000; on provider success debit `payout_inflight` · credit `provider_float:orange` 250 000 (docs/03 canonical; failure reverses the inflight entry).
- **Events**: `payout.succeeded` | `payout.failed`. **Notifications**: N-10 (large payouts also SMS N-11).

### F-015 — Paiement en masse CSV / Bulk CSV payout (upload→map→validate→review→approve)
- **Actor(s)**: maker Finance; approver Admin/Owner (F-017) · **Trigger**: `/payouts/new` (En masse) · **Preconditions**: wallet funded · **Frequency**: monthly (payroll) + ad-hoc.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Finance | Télécharge le modèle CSV ; téléverse son fichier | Parses UTF-8/;-or-,; ≤ 1 000 lignes v1 | `/payouts/new` (mass) |
| 2 | Finance | Mappe les colonnes (téléphone, nom, montant, canal?, référence?) | Auto-suggest by header names | column mapper |
| 3 | — | Rapport de validation | Per-row checks: phone E.164/prefix, amount ≥ 100 entier, duplicate refs, unknown channel; errors listed per row ("Ligne 12 : numéro invalide") | validation report |
| 4 | Finance | Corrige (re-upload) ou "Exclure les lignes en erreur" | Only valid rows carried forward | idem |
| 5 | Finance | Écran de revue : total, frais, table par item, funding check "Solde : 1 200 000 FCFA — suffisant ✅ / insuffisant ❌" | `POST /payout_batches` → batch `draft` then `pending_approval` | review |
| 6 | Admin | Approuve (F-017) | batch `processing` → items dispatched → `completed`/`partially_failed` | `/payouts/batches/:id` |

- **Failure branches**: >1 000 rows → "Fichier trop grand — 1 000 lignes maximum, divisez le fichier"; all rows invalid → cannot proceed; insufficient funding at review → submit disabled + top-up CTA; per-item failures during processing → item `failed` with failure_code, retry per item; partial → banner "47/50 réussis — 3 échecs à revoir" + report download.
- **Postconditions**: per succeeded item, ledger as F-014. **Events**: `payout.succeeded`/`payout.failed` per item, `payout_batch.completed`. **Notifications**: N-12 (approvers), N-13 (completed/partial with report).

### F-016 — Paie programmée / Scheduled payroll run
- **Actor(s)**: Finance (setup), Admin/Owner (approval), System (scheduler) · **Trigger**: saved payroll list + schedule (e.g. le 28 du mois) `[API-ADD-5]` · **Preconditions**: payroll list exists on `/payouts/beneficiaries` · **Frequency**: monthly.
- **Steps**: 1) Finance creates named list (beneficiaries + amounts) → 2) sets schedule (day of month, time) → 3) on date, system drafts a batch from the list, funding-checked → `pending_approval` → N-12 to approvers ("Paie de juillet — 4 250 000 FCFA — 50 bénéficiaires") → 4) approver reviews on `approval-detail` (anomaly hints "⚠ 2 nouveaux bénéficiaires") → F-017/F-018 → 5) processing as F-015 step 6.
- **Failure branches**: wallet insufficient on run date → batch stays `draft`, N-14 "Paie non lancée — solde insuffisant"; no approval within 48 h → reminder N-12 re-sent, batch expires to `draft` after 7 days.
- **Postconditions/Events/Notifications**: as F-015.

### F-017 — Approbation d'un lot / Batch approval (maker–checker)
- **Actor(s)**: Admin/Owner, **not** the maker · **Trigger**: N-12 deep-link → `/payouts/batches/:id` approval panel / app `approval-detail` · **Preconditions**: batch `pending_approval`; approver on enrolled device (app) · **Frequency**: per batch.
- **Steps**: 1) reviews summary (total XL, count, créé par, motif) + item list + anomaly hints → 2) [Approuver `check-check`] → confirm restates "Approuver 4 250 000 FCFA vers 50 bénéficiaires ?" → 3) web: TOTP re-prompt; app: biometric/PIN, enrolled device only → 4) `POST /payout_batches/{id}/approve` → `processing`; audit strip "créé par X, approuvé par Y".
- **Failure branches**: maker attempts self-approval → 403 `permission_denied`, UI shows "Vous ne pouvez pas approuver votre propre lot"; wallet drained between creation and approval → approval rejected server-side "Solde devenu insuffisant — approvisionnez puis réessayez"; 2FA fail → retry; device not enrolled → "Approbation impossible depuis cet appareil — utilisez un appareil enregistré".
- **Postconditions**: batch processing; per-item ledger as F-014. **Events**: item events + `payout_batch.completed`. **Notifications**: N-13; N-11 SMS to Owner for large totals.

### F-018 — Rejet d'un lot / Batch rejection
- **Actor(s)**: Admin/Owner approver · **Trigger**: same panel as F-017 · **Steps**: [Rejeter] → required reason (free text, ≥ 10 chars) → `POST /payout_batches/{id}/reject` `[API-ADD-6]` → batch back to `draft` with rejection note visible to maker → N-15 to maker "Lot rejeté : {reason}". Maker edits and resubmits or deletes draft.
- **Failure branches**: empty reason → "Indiquez la raison du rejet"; concurrent approve/reject → first write wins, second gets `idempotency_conflict`-style 409 "Ce lot a déjà été traité".
- **Postconditions**: batch `draft`; no money moved. **Events**: none. **Notifications**: N-15.

### F-019 — Approvisionnement du portefeuille / Wallet top-up
- **Actor(s)**: Finance/Admin/Owner · **Trigger**: [Approvisionner] on `/balance` / app `topup-instructions`; or blocking banner CTA · **Frequency**: weekly.
- **Steps**: 1) enters intended amount → `POST /topups` → returns unique reference code + exact USSD steps per channel ("Composez *126# → Transfert → 6 5X XX XX XX → montant → référence {code}") → 2) merchant executes transfer from their own MoMo/OM/bank → 3) system auto-matches incoming funds (F-061) → screen auto-updates to ✓ "Approvisionnement reçu : 500 000 FCFA" + push N-16. Alt path: internal transfer from `merchant_available` to wallet (instant, confirm modal restates amount) `[API-ADD-7]`.
- **Failure branches**: amount received ≠ intended → matched anyway on reference, actual amount credited, note shown; no reference used → falls to unmatched queue, ops manual match (F-049 adjacent), merchant sees "En attente de rapprochement — jusqu'à 24 h"; expired intent (7 days) → new code needed.
- **Postconditions**: journal `topup`: debit `provider_float:{channel}` 500 000 · credit `merchant_payout_wallet` 500 000 (internal-transfer path: debit `merchant_available` / credit `merchant_payout_wallet`).
- **Events**: `balance.topup.received`. **Notifications**: N-16.

### F-020 — Règlement reçu & changement de calendrier / Settlement receipt & schedule change (24 h cooldown)
- **Actor(s)**: Finance/Admin/Owner · **Trigger**: settlement paid (F-059) or settings change on `/settlements` settings · **Frequency**: daily receipts; rare changes.
- **Steps** (receipt): 1) N-17 push/email/SMS arrives → deep-link `/settlements/:id` (app `settlements-detail`) → 2) merchant views included transactions, downloads relevé PDF/CSV. **Steps (change)**: 1) settings → schedule (T+1/hebdomadaire) and/or destination account → 2) TOTP required → 3) change saved with **cooldown banner "Nouveau compte de règlement actif dans 24 h"** — settlements in the window still go to the old account → 4) N-22 security alert (push+email+SMS) "Compte de règlement modifié — si ce n'est pas vous, contactez-nous immédiatement".
- **Failure branches**: TOTP fail → change not saved; destination phone prefix mismatch with chosen channel → "Ce numéro n'est pas un numéro {channel}"; change attempted twice in 24 h → blocked "Un changement est déjà en cours d'activation".
- **Postconditions**: `settlement_schedule`/`default_settlement_channel` updated after cooldown; audit_log. **Events**: `settlement.paid` (receipt side). **Notifications**: N-17, N-22.

### F-021 — Inviter un membre / Team invite
- **Actor(s)**: Admin/Owner · **Trigger**: `/team` invite modal / app `team-invite` · **Steps**: 1) phone or email + role picker with plain-language explainers ("Finance : crée des paiements, ne peut pas les approuver") → 2) `[API-ADD-8]` sends invite (N-18) → 3) invitee opens `/accept-invite`: sees inviter, merchant, role; sets password if new user; accepts → membership created → N-19 to Owner "Nouveau membre : {name} ({role})".
- **Failure branches**: already a member → "Cette personne fait déjà partie de l'équipe"; expired invite (7 days) → "Invitation expirée — demandez une nouvelle invitation"; decline → inviter notified in bell tray.
- **Postconditions**: membership row. **Notifications**: N-18, N-19.

### F-022 — Changer un rôle / Role change
- **Actor(s)**: Admin/Owner · **Trigger**: `/team` row action · **Steps**: role sheet → confirm "Passer {name} de Finance à Admin ?" → `[API-ADD-8]` → effective immediately, target's open sessions get refreshed permissions → N-19 variant email to target.
- **Failure branches**: demoting last Owner → blocked "Il doit rester au moins un propriétaire"; self-demotion by sole Owner → same block; Viewer attempting → UI hidden.
- **Postconditions**: membership.role updated; audit_log. **Notifications**: N-19.

### F-023 — Retirer un membre / Member removal
- **Actor(s)**: Admin/Owner · **Trigger**: `/team` remove action · **Steps**: confirm "Retirer {name} de {merchant} ? Son accès sera coupé immédiatement." → `[API-ADD-8]` → sessions revoked, enrolled devices unlinked from this merchant → N-19 variant.
- **Failure branches**: removing last Owner → blocked; removing self → allowed except sole Owner.
- **Postconditions**: membership deleted, sessions revoked, audit_log. **Notifications**: N-19.

### F-024 — Créer / faire tourner une clé API / API key create & rotate
- **Actor(s)**: Developer/Admin/Owner · **Trigger**: `/developers/keys` · **Steps** (create): name it → `[API-ADD-9]` → **reveal-once modal** with copy `copy`; hash stored only. **Steps (rotate)**: create replacement → both keys valid during grace window chosen by merchant (0/24 h/72 h banner "Ancienne clé expire le {date}") → old key auto-revoked at window end.
- **Failure branches**: closing reveal modal without copying → warning "Vous ne pourrez plus voir cette clé"; Viewer/Finance → section hidden.
- **Postconditions**: `api_keys` rows; `last_used_at` tracked. **Notifications**: N-23 (email to Owner on live-key creation).

### F-025 — Révoquer une clé API / API key revoke
- **Actor(s)**: Developer/Admin/Owner · **Trigger**: `/developers/keys` revoke · **Steps**: confirm restates key prefix + last-used "Révoquer sk_live_…4f2 (utilisée il y a 2 h) ? Les requêtes échoueront immédiatement." → `[API-ADD-9]` → `revoked_at` set; subsequent API calls 401 `authentication_failed`.
- **Failure branches**: revoking the only live secret key → extra warning "C'est votre seule clé réelle — vos encaissements API s'arrêteront."
- **Postconditions**: key dead; audit_log. **Notifications**: N-23.

### F-026 — Configurer un webhook / Webhook setup
- **Actor(s)**: Developer/Admin/Owner · **Trigger**: `/developers/webhooks` create · **Steps**: 1) URL (https required) + event picker (docs/02 types) → `POST /webhook_endpoints` → secret reveal-once → 2) "Envoyer un ping" → `POST /webhook_endpoints/{id}/ping` → shows response code/latency → 3) endpoint `healthy`.
- **Failure branches**: non-https URL → "L'URL doit être en https"; ping non-2xx → inline "Réponse {code} — vérifiez votre serveur" with docs link `book-open`; unreachable → timeout message.
- **Postconditions**: endpoint row enabled. **Notifications**: none.

### F-027 — Webhook en échec & réactivation / Webhook failing & re-enable
- **Actor(s)**: Developer + System · **Trigger**: deliveries failing (F-058) · **Steps**: 1) after 24 h of failures → endpoint badge `failing` `triangle-alert` + N-24 (dev push + email) → 2) merchant fixes server, opens endpoint detail: delivery log (event, code, attempts, next retry), presses redeliver per event `[API-ADD-10]` → 3) if auto-disabled after 7 days (F-058), banner "Point de terminaison désactivé le {date}" + [Réactiver] `[API-ADD-10]` → re-enable, then manually redeliver missed events from `/developers/events` (90-day log).
- **Failure branches**: redeliver still failing → stays in ladder; events older than 90 days → not redeliverable, copy states retention.
- **Postconditions**: endpoint enabled/disabled state; deliveries logged. **Notifications**: N-24, N-25 (auto-disabled).

### F-028 — Créer un plan / Plan create
- **Actor(s)**: Admin/Owner/Finance · **Trigger**: `/subscriptions/plans` · **Steps**: name + amount (≥ 100 FCFA) + interval (semaine/mois/année) → `POST /plans` → plan listed. Edits create new plan version; existing subscriptions keep old amount (stated in edit modal).
- **Failure branches**: duplicate name → warning, allowed; amount < 100 → validation error.
- **Postconditions**: plan row. **Notifications**: none.

### F-029 — Abonner un client / Subscribe customer
- **Actor(s)**: Admin/Owner/Finance · **Trigger**: `/subscriptions` "Nouvel abonnement" · **Steps**: 1) customer (existing picker or phone+name) + plan + start date (default aujourd'hui) → `POST /subscriptions` → subscription `active`, `current_period_*` set → 2) first invoice generated at start (F-060) → customer approves the first charge on their phone (F-043).
- **Failure branches**: start date past → "La date de début doit être aujourd'hui ou future"; first charge fails → retry ladder (F-060); customer phone invalid → validation.
- **Postconditions**: subscription + first invoice `open`. **Events**: `invoice.paid`/`invoice.payment_failed` downstream. **Notifications**: N-32 (customer SMS "Votre abonnement {plan} démarre…").

### F-030 — Relance manuelle / Manual dunning
- **Actor(s)**: merchant (Finance/Admin/Owner) · **Trigger**: subscription `past_due` detail, "Envoyer un lien de paiement" `message-circle`; or bulk "relancer" on past-due list · **Steps**: 1) system generates a single-use payment link for the open invoice `[API-ADD-11]` → 2) sent to customer via SMS/WhatsApp (N-33) → 3) payer pays (F-044) → invoice `paid`, subscription back to `active`.
- **Failure branches**: invoice already paid between click and send → toast "Facture déjà payée"; rate limit → max 1 manual relance per invoice per 24 h "Relance déjà envoyée aujourd'hui".
- **Postconditions**: on payment, ledger as F-008; invoice `paid`. **Events**: `invoice.paid`. **Notifications**: N-33.

### F-031 — Enregistrement d'un appareil / Device enrollment
- **Actor(s)**: app user · **Trigger**: first login on a device (part of F-001/F-002 app path) · **Steps**: OTP-verified login → device fingerprint + FCM token stored `[API-ADD-12]` → device listed in `/settings/security` and app `security → appareils` (name, model, last active) → approvals allowed from this device after 24 h seasoning if user has approver role (anti-takeover).
- **Failure branches**: rooted device → warning on launch + payout approvals disabled on that device; max 5 devices → "Limite d'appareils atteinte — révoquez un ancien appareil".
- **Postconditions**: device row; N-20 alert. **Notifications**: N-20.

### F-032 — Révocation d'un appareil / Device revocation
- **Actor(s)**: user (own devices) or Admin/Owner (any member's) · **Trigger**: `/settings/security` devices list / app `security` · **Steps**: confirm "Révoquer « Samsung A14 » ? L'application sera déconnectée sur cet appareil." → `[API-ADD-12]` → sessions killed, push tokens invalidated → revoked device shows login screen on next open.
- **Failure branches**: revoking current device → extra confirm "Vous utilisez cet appareil actuellement".
- **Postconditions**: device revoked; audit_log. **Notifications**: N-21 variant email.

### F-033 — Récupération de compte / Account recovery
- **Actor(s)**: user who lost phone/SIM and/or 2FA · **Trigger**: "Je n'ai plus accès à mon numéro" on `/forgot-password` · **Steps**: 1) identity re-verification: email OTP (if on file) + upload of owner ID matching KYB file `[API-ADD-13]` → 2) ops manual review queue (SLA 24 h) compares to `kyb_documents` → 3) approved → phone number updated, all sessions/devices revoked, 2FA reset required at next login, payouts locked 48 h → N-22 to all Owner/Admin contacts.
- **Failure branches**: mismatch → rejected with reason "Les informations ne correspondent pas au dossier — contactez le support"; suspected takeover → risk case (F-052) opened, account frozen for money-out.
- **Postconditions**: credentials rotated; audit trail. **Notifications**: N-22.

### F-034 — Changement d'entreprise / Business switch
- **Actor(s)**: user with ≥ 2 memberships · **Trigger**: merchant switcher in topbar / app header · **Steps**: picker lists businesses (name, role, tier) → select → context switches (data, keys, role) `[API-ADD-14]`; last-used business remembered per device; "Créer une nouvelle entreprise" entry launches F-001 steps 4–5 under same user (sub-accounts, PRD §1 SHOULD).
- **Failure branches**: suspended business in list → shown greyed "Suspendu" and read-only on open.
- **Postconditions**: session scoped to selected merchant_id. **Notifications**: none.

---

## B. Payer flows (`pay.ijimpay.com` — not our account holder)

### F-035 — Payer via lien réutilisable / Pay via reusable link
- **Actor(s)**: payer · **Trigger**: opens `/l/{slug}` (WhatsApp/SMS/QR) · **Preconditions**: link active · **Frequency**: the volume driver.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Payeur | Voit logo/nom marchand, titre, **montant héros** (fixe) ou saisit le montant (libre) | funnel event `link_viewed` / `amount_entered` | `/l/{slug}` |
| 2 | Payeur | "Payer 15 000 FCFA" → saisit son numéro; ChannelChip auto-détecté ("Numéro MTN détecté"), [Changer] pour surcharger | `phone_submitted` | Phone & channel |
| 3 | Payeur | "Confirmer le paiement" | `POST /charges` (publishable-key context) → `pending`; `charge_created` | → `/pay/{charge_id}` |
| 4 | Payeur | Valide sur son téléphone : MTN "Validez la notification, ou composez **\*126\*1#** puis confirmez avec votre code MoMo." / Orange "Composez **\*150\*4\*4#** pour approuver…" | 3 s polling/SSE; countdown ring 5 min; progressive copy 45 s / 2 min | Waiting |
| 5 | — | — | `succeeded` → success moment; receipt link `/r/{id}` | Success |

- **Failure branches**: link inactive → "Ce lien n'est plus actif"; expired → `timer-off` state; amount below min → inline "Montant minimum : {min} FCFA"; invalid number → inline; `channel_unavailable` → "Orange Money est indisponible actuellement" + suggest other channel; rate limit → "Une demande est déjà en cours sur ce numéro — validez-la ou attendez 5 min"; terminal failures → F-039; abandon → link stays reusable, funnel `abandoned`.
- **Postconditions**: on success, ledger as F-008; charge linked to `payment_link_id`. **Events**: `charge.succeeded|failed|expired`. **Notifications**: N-07 (merchant), N-30 (payer receipt).

### F-036 — Payer via lien à usage unique / Single-use link
- Same as F-035 with: **Preconditions** link unpaid; after success the link is consumed — re-opening `/l/{slug}` shows "already paid" state with the receipt link. **Failure branch add**: two payers open simultaneously → first successful charge wins; second gets "Ce lien a déjà été payé" at charge-create time (server check), lands on the already-paid state.

### F-037 — Payer via session de checkout (redirection + retour) / Checkout session
- **Actor(s)**: payer from e-commerce site · **Trigger**: merchant server `POST /checkout_sessions` → redirect to `/c/{session_id}` · **Steps**: landing shows order summary accordion + "Retour à {merchant}" (→ `cancel_url`) → steps 2–5 of F-035 → success screen auto-redirects to `success_url` after 3 s carrying `charge_id` + signature params (merchant verifies server-side), manual "Retourner à {merchant}" button.
- **Failure branches**: session expired → "Retournez à la boutique pour recommencer"; payer cancels → `cancel_url`, session `abandoned` (merchant webhook per docs/09); charge succeeds after tab closed → receipt reachable, webhook unaffected; iframe widget blocked → plain redirect fallback.
- **Postconditions**: session `completed`; ledger as F-008. **Events**: charge events. **Notifications**: N-07, N-30.

### F-038 — QR statique au comptoir (montant libre) / Static counter QR
- **Actor(s)**: payer at a stall · **Trigger**: scans printed QR → `/l/{slug}` open-amount · **Steps**: enters the amount the merchant states aloud → F-035 steps 2–5 → success screen shows "Montrez cet écran au commerçant" while merchant's phone rings with N-07 push (the true confirmation).
- **Failure branches**: payer enters wrong amount and pays → merchant refunds (F-012) or requests complement charge; all F-035 failures.
- **Postconditions/Events/Notifications**: as F-035.

### F-039 — Réessayer après échec / Retry after each failure_code
- **Actor(s)**: payer · **Trigger**: failure screen on `/pay/{charge_id}` · **Steps & branches (one per code)** — each shows `circle-x`, human reason, and actions [Réessayer] (creates a NEW charge, same details) · [Changer de numéro/canal] (F-041) · back to merchant (sessions):
  - `insufficient_payer_funds` → "Solde insuffisant sur ce compte" — retry advised after payer tops up their MoMo; also offers channel change.
  - `payer_rejected` → "Le paiement a été refusé" — retry re-sends approval push.
  - `payer_timeout` / status `expired` → "La demande a expiré" — retry immediate; copy "Gardez votre téléphone à portée de main".
  - `payer_not_found` → "Ce numéro n'a pas de compte {channel}" — retry disabled for same number+channel pair; forces F-041.
  - `provider_error` → "Erreur chez l'opérateur — réessayez" — retry allowed; if `/channels` shows degraded, banner suggests the other channel.
  - `channel_unavailable` (at create) → "Ce moyen de paiement est indisponible actuellement" — only channel change offered.
- **Postconditions**: each retry = new `charges` row with its own idempotency key; failed charge remains for funnel analytics. **Events**: per charge. **Notifications**: N-07 on eventual success.

### F-040 — Renvoyer la demande / Resend request
- **Actor(s)**: payer · **Trigger**: at 2 min on waiting screen, [Renvoyer la demande] appears (allowed once) · **Steps**: click → system cancels the pending charge and creates a fresh one (new USSD push), same `/pay/{charge_id}`-style page for the new charge; button then disabled "Demande renvoyée".
- **Failure branches**: original succeeds in the race window → server rejects resend, page flips to Success; resend also times out → normal expiry (F-056) then F-039.
- **Postconditions**: old charge `expired` (superseded note in metadata), new charge `pending`. **Events**: `charge.expired` (old) then terminal event (new).

### F-041 — Changer de numéro / canal en cours / Change number or channel mid-flow
- **Actor(s)**: payer · **Trigger**: [Changer de numéro] on waiting (from 2 min) or failure screen · **Steps**: returns to Phone & channel with amount preserved → new number → auto-detect chip, override chips MTN/Orange (radio) → confirm → new charge. Pending old charge is cancelled server-side.
- **Failure branches**: unknown prefix → manual channel pick required; same F-035 create errors.
- **Postconditions**: new charge on new phone/channel. **Events**: as usual.

### F-042 — Vérifier un reçu / Verify receipt
- **Actor(s)**: payer, or anyone shown a receipt (anti-fake-receipt) · **Trigger**: opens `/r/{receipt_id}` (QR on thermal receipt, link on success screen) · **Steps**: public no-auth page: banner "✔ Paiement vérifié par IjimPay", merchant, montant, statut, réf, date, canal, PDF download; anti-fraud note "Vérifiez toujours vos reçus sur pay.ijimpay.com". No JS required.
- **Failure branches**: unknown id → 404 "Reçu introuvable — méfiez-vous des reçus falsifiés"; refunded charge → receipt shows StatusBadge `refunded`.
- **Postconditions**: none. **Notifications**: none.

### F-043 — Approbation du cycle d'abonnement / Subscription cycle approval
- **Actor(s)**: subscribed customer · **Trigger**: billing date — system fires charge (F-060), USSD push arrives on their phone · **Steps**: 1) optional pre-notice N-32 the day before ("Votre abonnement {plan} de 5 000 FCFA sera prélevé demain") → 2) approval push arrives → customer validates with MoMo/OM code → 3) invoice `paid`, N-30 receipt.
- **Failure branches**: ignores push → retry ladder T+0, +6 h, +24 h, +72 h (F-060); each retry sends a fresh push; funds insufficient → same ladder; ladder exhausted → dunning F-044.
- **Postconditions**: on success, ledger as F-008, invoice `paid`. **Events**: `invoice.paid`. **Notifications**: N-30, N-32.

### F-044 — Paiement via lien de relance / Dunning link payment
- **Actor(s)**: past-due customer · **Trigger**: dunning SMS/WhatsApp N-33/N-34 with single-use link · **Steps**: opens `/l/{slug}` (fixed amount = invoice) → F-035 steps 2–5 → invoice `paid`, subscription `past_due → active`, retry ladder cancelled.
- **Failure branches**: pays after subscription auto-canceled → payment still accepted for the open invoice if link valid; if link expired with cancellation → "Ce lien n'est plus actif — contactez {merchant}"; standard payment failures → F-039.
- **Postconditions**: ledger as F-008; subscription restored. **Events**: `invoice.paid`. **Notifications**: N-07, N-30; merchant bell "Abonnement de {customer} régularisé".

---

## C. Ops flows (`ops.ijimpay.com` — SSO + hardware keys; roles ops, ops-admin, compliance, finance-admin)

### F-045 — Approbation KYB / KYB approve
- **Actor(s)**: ops (compliance for edge cases) · **Trigger**: item in `/queue/kyb` (SLA timer visible) · **Steps**: 1) side-by-side docs viewer vs form data → 2) checks: RCCM legible & matches legal name; ID matches owner; settlement-account proof matches declared number → 3) [Approuver Tier 1] `[API-ADD-15]` → merchant `kyb_status=approved`, tier 1, limits applied → 4) merchant notified N-04, checklist item done, go-live CTA appears (F-007).
- **Failure branches**: doc unreadable → per-doc reject (F-046) instead; name mismatch tolerable (accents/abbreviations) → approve with note; SLA breach → item escalates visually (danger tint) + pages ops-admin.
- **Postconditions**: tier 1; audit_log. **Notifications**: N-04.

### F-046 — Rejet KYB / KYB reject
- **Actor(s)**: ops · **Trigger**: same queue · **Steps**: per-document reject with FR reason template picker ("Photo illisible — reprenez la pièce à plat, sans reflet", "Document expiré", "Le nom ne correspond pas au RCCM", "Preuve de compte au mauvais nom") + optional free text → `[API-ADD-15]` → merchant `kyb_status=rejected` (per-doc statuses preserved) → N-05 → merchant resubmits (F-006).
- **Failure branches**: 3rd rejection → auto-task "call-back merchant" assigned; suspected forged docs → open risk case (F-052) instead of plain reject, merchant sees only "en cours d'examen approfondi".
- **Postconditions**: doc rows `rejected` with reasons. **Notifications**: N-05.

### F-047 — Montée en niveau / Tier upgrade (Tier 2)
- **Actor(s)**: ops maker + ops-admin approver (dual-control above Tier 1 per docs/08 §B) · **Trigger**: merchant request from F-007 or proactive · **Steps**: 1) ops reviews volumes, risk flags, extra docs on `/merchants/:id` tier & limits editor → 2) proposes Tier 2 limits → 3) second approver (ops-admin) confirms → tier applied `[API-ADD-15]` → N-06.
- **Failure branches**: approver rejects → note back to maker; open risk case on merchant → upgrade blocked "Cas de risque ouvert".
- **Postconditions**: tier 2 + limits; audit dual entries. **Notifications**: N-06.

### F-048 — Suspension d'un marchand / Merchant suspension
- **Actor(s)**: ops maker + ops-admin approver (dual-control) · **Trigger**: risk case, legal order, fraud confirmation · **Steps**: 1) `/merchants/:id` → [Suspendre] with mandatory reason + scope (collections only / all incl. payouts / full freeze incl. settlements) → 2) second approver confirms → merchant `status=suspended` → API returns `permission_denied` on blocked operations; dashboard/app show suspension banner "Compte suspendu — contactez le support" → N-26 email to Owner.
- **Failure branches**: partial suspension (payouts only) → collections continue, payouts hidden with banner; lift suspension → same dual-control in reverse.
- **Postconditions**: status flag; no ledger change (balances frozen, not seized). **Notifications**: N-26.

### F-049 — Rapprochement : import → correspondance → résolution / Reconciliation
- **Actor(s)**: finance-admin/ops · **Trigger**: provider report arrives (SFTP nightly or manual upload on `/reconciliation`) · **Frequency**: daily per provider.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | Ops | Importe le rapport | `provider_reports` row; matcher runs: exact `provider_ref`, fallback fuzzy (phone+amount+time window) | `/reconciliation` |
| 2 | — | — | `reconciliation_items` created per unmatched line, tabbed by type | discrepancy queue |
| 3 | Ops | Traite chaque type — see below | resolve with note; adjustments dual-control (F-050) | investigate panel |

- **Per-type resolution**: **`missing_ours`** (telco has it, we don't): investigate raw provider payload; if a real payment we never recorded → create charge record + adjustment journal crediting the merchant (dual-control), notify merchant bell; if telco error → resolve "erreur opérateur" with note, dispute filed with telco. **`missing_theirs`** (we show success, telco doesn't): re-poll provider (manual re-poll button); if genuinely absent → reverse our charge via adjustment (debit `merchant_pending`/`merchant_available`, credit `provider_float`), merchant notified, dispute opened. **`amount_mismatch`**: adjustment for the delta in the correct direction, note references both amounts.
- **Failure branches**: report file malformed → import rejected with line errors; duplicate report period → warned, re-import replaces unresolved items only; unresolved > 48 h → pages finance-admin.
- **Postconditions**: all items `matched` or resolved with notes; every money-moving resolution = balanced `adjustment` journal entry. **Notifications**: internal paging; merchant bell entries where balances changed.

### F-050 — Ajustement manuel (double contrôle) / Manual adjustment dual-control
- **Actor(s)**: finance-admin maker + second approver · **Trigger**: `/adjustments` create, or from F-049 resolution · **Steps**: 1) debit/credit account pickers (chart of accounts), amount, reason, reference → 2) **journal entry preview shown before confirm** (mandatory per docs/08 §B) → 3) submitted `pending_approval` → 4) second approver reviews and approves → journal `adjustment` posted (append-only; corrections are new reversing entries).
- **Failure branches**: self-approval → blocked; unbalanced entry → impossible by construction (single amount, two accounts); approver rejects → returns to maker with note.
- **Postconditions**: journal `adjustment`: debit {account A} X · credit {account B} X; full audit trail (maker, approver, reason).
- **Notifications**: merchant bell + N-27 email if a merchant balance changed.

### F-051 — Coupe-circuit & rétablissement / Circuit-break & restore
- **Actor(s)**: ops maker + ops-admin (dual-control) · **Trigger**: provider incident (from F-063 or telco notice) · **Steps**: 1) `/providers` → toggle "pause Orange collections" (scope: collections/disbursements/both) → second approver confirms → 2) `GET /channels` reports `orange_money: down`; checkout hides/disables the channel with "Orange Money est indisponible actuellement"; dashboard/app show warning Banner "Les paiements Orange peuvent être lents actuellement" (degraded) or down copy; payout submit disabled for channel → 3) on telco recovery, restore with same dual-control → banners clear, synthetic monitor (F-063) confirms green before full restore (staged: 10 % traffic canary optional).
- **Failure branches**: charges in flight at pause → allowed to reach terminal state normally; restore while telco still broken → synthetic failures re-page immediately.
- **Postconditions**: channel status flag; audit. **Notifications**: status banners (in-product); N-28 email to affected merchants on prolonged (> 2 h) outage.

### F-052 — Alerte risque → dossier → résolution / Risk flag → case → resolution
- **Actor(s)**: risk rules (system) + compliance · **Trigger**: rule fires (velocity, payer=merchant patterns, structuring, PIN-reset+payout pattern) · **Steps**: 1) flagged item appears in `/risk` review queue with rule, score, linked objects → 2) analyst opens case: notes, requests info from merchant if needed (email template), can apply holds (payouts locked) → 3) resolution: **cleared** (flag dismissed, holds lifted) / **restricted** (limits lowered, dual-control) / **escalated** (suspension F-048 and/or STR F-053).
- **Failure branches**: merchant unresponsive 7 days → auto-escalate; false-positive-heavy rule → threshold edit in rules list (audited).
- **Postconditions**: case record with outcome; audit. **Notifications**: N-29 (merchant info request) when applicable.

### F-053 — Déclaration de soupçon (ANIF) / STR filing
- **Actor(s)**: compliance only · **Trigger**: case outcome "escalated — STR" (F-052) · **Steps**: 1) `/risk` case → [Générer le dossier ANIF] → STR export pack compiled (merchant KYB, transactions, ledger extracts, case notes) → 2) compliance officer reviews pack, files with ANIF per regulatory channel (offline), records filing reference + date in case → 3) **no tipping-off**: merchant is never notified; UI shows nothing merchant-side.
- **Failure branches**: pack missing docs → checklist blocks generation "Pièce d'identité manquante au dossier"; filing deadline timer (regulatory) shown on case.
- **Postconditions**: case marked `str_filed` with reference; append-only audit. **Notifications**: none (deliberately).

### F-054 — Intégration du personnel ops / Staff onboarding
- **Actor(s)**: ops-admin · **Trigger**: new hire · **Steps**: 1) `/staff` → create account via company SSO directory → assign role (ops, ops-admin, compliance, finance-admin) → 2) hardware security key enrollment mandatory before first login is usable (status chip on `/staff`) → 3) role-scoped access; every action lands in `/audit` append-only viewer.
- **Failure branches**: no hardware key within 7 days → account auto-disabled; role change/offboarding → immediate session revocation, dual-control for ops-admin grants.
- **Postconditions**: staff record with key enrollment; audit. **Notifications**: internal email only.

---

## D. System flows (jobs, workers, automatic behavior)

### F-055 — Cycle de vie d'un encaissement / Charge lifecycle (create→provider→poller→terminal→ledger→webhook→push)
- **Actor(s)**: API gateway, provider adapter, poller, ledger writer, outbox relay, webhook worker, FCM · **Trigger**: `POST /charges` · **Frequency**: every charge.
- **Steps**:

| # | Acteur | Action | Comportement système | Écran |
|---|---|---|---|---|
| 1 | API | Valide + idempotence | `charges` row `pending`, `expires_at=+5 min`; `provider_transactions` row (direction `collection`) with raw_request | — |
| 2 | Adapter | Appelle le telco (request-to-pay) | stores raw_response, `provider_ref` | — |
| 3 | Poller | Interroge le statut (backoff 3 s→10 s), partial index `status='pending'` | `last_polled_at`, `poll_count`++ | payer sees `/pay/{id}` |
| 4 | — | Terminal reached (callback or poll) | **Same DB transaction**: charge status + ledger postings + outbox row (docs/03 rule 1) | — |
| 5 | Relay | Publishes outbox → queue | event row created (immutable, 90 d) | `/developers/events` |
| 6 | Webhook worker | POST to endpoints with `X-Ijimpay-Signature` | delivery logged; retries F-058 | endpoint detail |
| 7 | Push | FCM high-priority to merchant devices | N-07 with sound | app `charge-success` |

- **Failure branches**: provider call fails synchronously → charge `failed` `provider_error` immediately; callback and poll race → first terminal write wins (row lock), duplicates no-op; ledger write failure → whole transaction rolls back, charge stays `pending` for retry (never a status without postings).
- **Postconditions**: journal `charge_succeeded` (debit `provider_float:{ch}` gross · credit `merchant_pending` net · credit `platform_fees` fee) — only on success. **Events**: `charge.succeeded|failed|expired`. **Notifications**: N-07 (+N-30 payer).

### F-056 — Expiration des encaissements / Charge expiry job
- **Actor(s)**: cron (every 30 s) · **Trigger**: `pending` charges past `expires_at` · **Steps**: 1) final provider status query (avoid expiring a just-succeeded charge) → 2) still non-terminal → charge `expired` in same transaction as outbox row → 3) payer page flips to expired state; merchant waiting screen shows failure with resend.
- **Failure branches**: provider says succeeded at final check → mark `succeeded` + ledger instead (the check exists for this); provider unreachable → expire anyway, reconciliation (F-049) is the backstop.
- **Postconditions**: charge `expired`, no ledger entries. **Events**: `charge.expired`. **Notifications**: bell only.

### F-057 — Règle de re-vérification sur callback / Callback re-query rule
- **Actor(s)**: provider adapter · **Trigger**: any provider callback received · **Steps**: **never trust the callback payload for money state** — on callback, re-query the provider's status API for `provider_ref`; only the re-query result drives the state transition; callback merely accelerates polling. Signature/IP checks on callback endpoint; unknown `provider_ref` → logged, alert if volume.
- **Failure branches**: re-query fails → keep polling on normal schedule; callback claims success but re-query says pending → remain `pending` (poller resolves).
- **Postconditions**: state changes only from authoritative re-query. **Events**: downstream as F-055.

### F-058 — Échelle de reprise webhook → désactivation → réactivation / Webhook retry ladder
- **Actor(s)**: webhook worker · **Trigger**: non-2xx or timeout on delivery · **Steps**: retries at **1 m, 5 m, 30 m, 2 h, 6 h, puis toutes les 12 h jusqu'à 72 h** per event; endpoint-level health tracked; at 24 h continuous failure → endpoint badge `failing` + N-24; at **7 days** continuous failure → endpoint auto-disabled (deliveries stop) + N-25; merchant re-enables via F-027 and redelivers from the 90-day event log.
- **Failure branches**: endpoint returns 2xx intermittently → health resets, no disable; signature clock skew payer-side (|now−t| > 5 min reject rule) is the merchant's verification concern (F-069).
- **Postconditions**: `webhook_deliveries` rows per attempt. **Notifications**: N-24, N-25.

### F-059 — Job de règlement T+1 / Settlement T+1 job
- **Actor(s)**: settlement scheduler + disbursement worker · **Trigger**: daily at schedule (or weekly per merchant setting) · **Steps**: 1) per merchant: sum matured `merchant_pending` (past hold window) → journal **maturation**: debit `merchant_pending` / credit `merchant_available` → 2) if auto-settlement destination configured: create settlement for available amount → journal: debit `merchant_available` X · credit `settlement_clearing` X → 3) disburse to merchant's own MoMo/OM/bank → on provider success: debit `settlement_clearing` X · credit `provider_float:{ch}` X; settlement `paid` with included transactions list → 4) N-17 + relevé.
- **Failure branches**: disbursement fails → settlement `in-transit` retried; after 3 fails → reverse clearing entry back to `merchant_available`, ops paged, merchant banner "Règlement retardé — nos équipes sont dessus"; destination in 24 h cooldown (F-020) → old account used.
- **Postconditions**: the three journal entries above. **Events**: `settlement.paid`. **Notifications**: N-17 (push+email+SMS).

### F-060 — Facturation d'abonnement → échelle de reprise → past_due → annulation / Subscription invoice lifecycle
- **Actor(s)**: billing scheduler · **Trigger**: `current_period_end` reached · **Steps**: 1) invoice created `open` → charge attempt T+0 (at historically high-approval hour) → 2) failure → retries **+6 h, +24 h, +72 h** (`retry_state` jsonb tracks attempt #, next_attempt_at), each a fresh USSD push (F-043) → 3) ladder exhausted → invoice stays `open`, subscription `past_due`, automatic dunning link sent to customer (N-34) + merchant notified (bell + digest) → 4) after N failed cycles (default 2) → subscription `canceled`, open invoices `uncollectible`.
- **Failure branches**: customer pays via dunning link mid-ladder → ladder cancelled, invoice `paid`, subscription `active`; merchant pauses subscription → no further attempts; channel down on attempt day → attempt deferred, not counted.
- **Postconditions**: paid invoices post ledger as F-008. **Events**: `invoice.paid`, `invoice.payment_failed`, `subscription.past_due`, `subscription.canceled`. **Notifications**: N-32, N-34; merchant N-35 digest line.

### F-061 — Rapprochement automatique des approvisionnements / Top-up auto-match
- **Actor(s)**: incoming-funds watcher · **Trigger**: credit detected on our provider float matching a top-up reference (F-019) · **Steps**: 1) match on reference code (exact) → 2) journal `topup`: debit `provider_float:{ch}` / credit `merchant_payout_wallet` → 3) event + push N-16; app `topup-instructions` screen live-flips to ✓.
- **Failure branches**: no reference match → unmatched queue for ops manual match (48 h SLA); amount differs from intent → credit actual amount, note attached; duplicate transfer same reference → second credited too, flagged for review.
- **Postconditions**: wallet credited. **Events**: `balance.topup.received`. **Notifications**: N-16.

### F-062 — Assertions nocturnes du grand livre / Nightly reconciliation assertions
- **Actor(s)**: nightly job (03:00 WAT) · **Trigger**: cron · **Steps** (docs/03 rule 4): 1) every journal entry sums to zero (trigger-enforced, re-asserted) → 2) every `account_balances` materialization equals `SUM(postings)` → 3) `provider_float` per telco equals latest telco statement → 4) any mismatch → `reconciliation_items` created + ops paged; dashboard `/providers` shows float-vs-ledger deltas.
- **Failure branches**: assertion failure = P1 page, never auto-corrected; statement not yet available → float check skipped with warning, retried at 06:00.
- **Postconditions**: green report or paged incident. **Notifications**: internal paging only.

### F-063 — Moniteur synthétique → statut → bannières / Synthetic monitor
- **Actor(s)**: monitor worker · **Trigger**: every 5 min per provider · **Steps**: 1) fires a real small test charge on a company-owned SIM per channel (auto-approved rig) → 2) measures success + latency + pending age → 3) rolling window classifies `operational | degraded | down` → published on `GET /channels` → 4) UI reacts automatically: dashboard/app provider strip dots, warning Banner "Les paiements Orange peuvent être lents actuellement" (degraded), checkout channel messaging/disable (down), payout channel disable → 5) sustained `down` → suggests circuit-break to ops (F-051 remains a human dual-control decision).
- **Failure branches**: rig SIM broken (both channels red simultaneously with telcos fine) → self-check alert, status held at last-known; flapping → hysteresis (2 consecutive windows to change state).
- **Postconditions**: channel status; monitor charges excluded from merchant analytics. **Notifications**: banners in-product; ops paging on `down`.

### F-064 — Alerte de dérive de solde / Balance-drift alert
- **Actor(s)**: continuous checker (hourly) · **Trigger**: cron · **Steps**: 1) recompute `SUM(postings)` per account vs `account_balances` materialization and vs cached API `GET /balance` values → 2) any drift ≠ 0 → freeze is NOT automatic, but: P1 page, `/providers` red tile, adjustments queue item with computed delta → 3) resolution only via F-050 dual-control adjustment or bug-fix + replay.
- **Failure branches**: drift explained by in-flight transaction timing → checker re-runs on a snapshot; repeated same-account drift → auto-opens engineering incident.
- **Postconditions**: alert trail; ledger untouched until human adjustment. **Notifications**: internal only.

### F-065 — Rejeu d'idempotence / Idempotency replay
- **Actor(s)**: API gateway · **Trigger**: POST money operation with a previously seen `Idempotency-Key` (24 h store) · **Steps**: 1) same key + same request body hash → return the **original stored response** (same status code, same object ids) — no new charge/payout/refund created → 2) same key + different body → `409 idempotency_conflict` "Cette clé d'idempotence a déjà été utilisée avec une requête différente".
- **Failure branches**: key reused after 24 h expiry → treated as new (documented, docs/02 §1); concurrent duplicate in-flight → second request blocks until first commits, then replays.
- **Postconditions**: exactly one business object per key. **Events**: none extra.

---

## E. Developer flows

### F-066 — Parcours doré sandbox : première charge < 30 min / Sandbox golden path
- **Actor(s)**: developer at a merchant (any tier, test mode) · **Trigger**: signup (F-001) → docs quickstart · **Frequency**: every integration.
- **Steps**: 1) `/developers/keys` → copy `sk_test_` → 2) quickstart (docs.ijimpay.com, `book-open` banner) → `POST /charges` with magic number `237670000001`, `Idempotency-Key`, reference → 202 with `pending` → 3) after 5 s simulated success → `GET /charges/{id}` shows `succeeded` → 4) sets a test webhook endpoint (F-026) + "send test event" button → verifies signature (F-069) → 5) dashboard `/transactions` (test mode) shows the charge — checklist item "Testez l'API" turns done. Target wall-clock < 30 min from signup.
- **Failure branches**: 401 → key mode/typo hints in error `doc_url`; missing Idempotency-Key on POST → `invalid_request` with explicit message; webhook not received → ping tool + delivery log for diagnosis.
- **Postconditions**: test-mode charge + event + delivery. **Notifications**: none (test mode fires test webhooks identically).

### F-067 — Numéros magiques / Magic numbers matrix
- **Actor(s)**: developer · **Trigger**: testing failure handling · **Steps / behavior contract** (docs/02 §3): `237670000001` → succeeds after 5 s · `237670000002` → fails `insufficient_payer_funds` · `237670000003` → fails `payer_rejected` · `237670000004` → stays pending until expiry → `expired`. Each produces the corresponding webhook event; checkout test pages behave identically. Developer must exercise all four before go-live (checklist F-068).
- **Failure branches**: magic number used in live mode → normal live behavior (they are real-shaped numbers only in the simulator); non-magic number in test mode → simulator defaults to success after 5 s.
- **Postconditions**: test objects only.

### F-068 — Liste de contrôle avant mise en production / Go-live checklist
- **Actor(s)**: developer + Owner · **Trigger**: F-007 go-live modal and docs page · **Steps** (each checkable, stored per merchant `[API-ADD-16]`): 1) all four magic-number outcomes handled in code · 2) webhook signature verification implemented (F-069) and ping green on the **live** endpoint · 3) idempotency keys on every POST money call · 4) terminal-state handling relies on webhooks/polling, never on redirect alone (session flows verify signature params server-side) · 5) live keys stored in secret manager, never client-side (`pk_` only in browser) · 6) settlement account confirmed · 7) rate-limit/429 handling with `Retry-After`. Completing unlocks a "Prêt pour le réel ✓" badge on the checklist (advisory, not blocking).
- **Failure branches**: live endpoint ping failing → item stays red with delivery log link.
- **Postconditions**: checklist state saved. **Notifications**: none.

### F-069 — Vérification de signature webhook / Webhook signature verification
- **Actor(s)**: developer's server · **Trigger**: every received webhook · **Steps** (contract per docs/02 §2.7): 1) read `X-Ijimpay-Signature: t=...,v1=...` → 2) compute `hex(hmac_sha256(secret, t + "." + raw_body))` on the **raw** body → 3) constant-time compare with `v1` → 4) reject if |now − t| > 5 min (replay) → 5) respond 2xx fast (< 5 s), process async; dedupe on event `id` (at-least-once delivery).
- **Failure branches**: body re-serialized before HMAC → mismatch (docs warn: raw bytes); multiple `v1` values during secret rotation → accept any match; non-2xx/slow → retry ladder F-058 (duplicates expected → dedupe).
- **Postconditions**: merchant system state driven only from verified events.

---

## Appendix A — Catalogue complet des messages / Complete message catalog

Every push/email/SMS/WhatsApp message in the flows above. Channels per docs/08 §C matrix. All FR bodies are the actual v1 copy; EN equivalent one-liner given. Sender for customer-facing SMS/WhatsApp: merchant-branded "via IjimPay".

| # | Trigger (flow) | Audience | Canaux | Titre/Sujet FR | Corps FR (template) | EN |
|---|---|---|---|---|---|---|
| N-01 | F-001 signup done | New owner | Email | Bienvenue sur Ijim Pay | Bonjour {name}, votre compte {merchant} est créé. Explorez le mode test, puis vérifiez votre entreprise pour encaisser en réel : {link}. Encaissez, simplement. | Welcome — your Ijim Pay account is ready. |
| N-02 | F-001/2/3/4 OTP | User | SMS | — | Ijim Pay : votre code est {code}. Valable 10 minutes. Ne le partagez jamais. | Your Ijim Pay code is {code}. |
| N-03 | F-005/F-006 submitted | Owner/Admin | Email + push | Documents reçus | Nous avons bien reçu vos documents pour {merchant}. Examen sous ~24 h — nous vous préviendrons dès la décision. | We received your documents; review within ~24 h. |
| N-04 | F-045 approved | Owner/Admin | Push + email + SMS | Entreprise vérifiée ✓ | Félicitations ! {merchant} est vérifiée (Niveau 1). Vous pouvez passer en mode réel : {link}. | Your business is verified — you can go live. |
| N-05 | F-046 rejected | Owner/Admin | Push + email + SMS | Vérification : action requise | Certains documents de {merchant} n'ont pas pu être validés : {reason}. Renvoyez-les ici : {link}. | Some documents were rejected: {reason}. Please resubmit. |
| N-06 | F-047 tier 2 | Owner | Push + email | Niveau 2 activé | {merchant} est passée au Niveau 2 : limites complètes activées. | Tier 2 approved — full limits active. |
| N-07 | F-008/9/35–38/44 charge.succeeded | Merchant devices | Push (son) + bell | Paiement reçu | +{amount} FCFA de {payer_phone} — {reference}. | Payment received: +{amount} FCFA. |
| N-08 | F-012 refund.succeeded | Merchant | Push + bell | Remboursement effectué | −{amount} FCFA remboursés à {payer_phone} ({reference}). | Refund of {amount} FCFA completed. |
| N-09 | F-013 export ready | Requesting user | Email | Votre export est prêt | Votre export {period} ({rows} lignes) est prêt. Téléchargez-le (lien valable 72 h) : {link}. | Your export is ready — download link (72 h). |
| N-10 | F-014 payout.succeeded | Maker + Owner | Push + bell | Paiement envoyé | {amount} FCFA envoyés à {beneficiary_name} ({beneficiary_phone}). | Payout of {amount} FCFA sent. |
| N-11 | F-014/17 large payout | Owner | SMS | — | Ijim Pay : paiement de {amount} FCFA approuvé vers {count} bénéficiaire(s) depuis {merchant}. Si ce n'est pas vous : {support_link}. | Large payout approved — contact us if this wasn't you. |
| N-12 | F-015/16 pending_approval | Approvers | Push + email + bell | Validation requise | Un lot de {count} paiements ({total} FCFA), créé par {maker}, attend votre validation : {link}. | A payout batch awaits your approval. |
| N-13 | F-015 batch completed / partially_failed | Maker + approver | Push + email (rapport joint) + bell | Lot terminé | Lot {batch_ref} : {ok}/{count} réussis pour {total} FCFA.{if_failed : « {failed} échec(s) à revoir : {link} »} Rapport en pièce jointe. | Batch finished: {ok}/{count} succeeded — report attached. |
| N-14 | F-016 payroll not launched | Finance + Owner | Push + email | Paie non lancée | La paie « {list_name} » n'a pas pu démarrer : solde du portefeuille insuffisant ({balance} FCFA pour {total} FCFA requis). Approvisionnez : {link}. | Payroll not started: insufficient wallet balance. |
| N-15 | F-018 batch rejected | Maker | Push + email + bell | Lot rejeté | Votre lot {batch_ref} a été rejeté par {approver} : « {reason} ». Modifiez-le ici : {link}. | Your batch was rejected: {reason}. |
| N-16 | F-019/61 topup.received | Finance + Owner | Push + email + bell | Approvisionnement reçu | +{amount} FCFA crédités sur votre portefeuille de paiement (réf {code}). | Top-up of {amount} FCFA received. |
| N-17 | F-059 settlement.paid | Finance + Owner | Push + email (relevé) + SMS + bell | Règlement envoyé | {amount} FCFA réglés vers votre compte {channel} •••{last4} pour la période du {date}. Relevé : {link}. | Settlement of {amount} FCFA paid — statement attached. |
| N-18 | F-021 invite | Invitee | Email + SMS | {inviter} vous invite sur Ijim Pay | {inviter} vous invite à rejoindre {merchant} en tant que {role}. Acceptez ici (valable 7 jours) : {link}. | You've been invited to join {merchant} as {role}. |
| N-19 | F-021/22/23 team change | Owner + target | Email + bell | Équipe mise à jour | {actor} a {action : « ajouté {name} ({role}) » / « changé le rôle de {name} en {role} » / « retiré {name} »} dans {merchant}. | Team change: {action}. |
| N-20 | F-002/31 new device | User | Push + email + SMS | Nouvel appareil connecté | Connexion à votre compte depuis {device_model} le {date}. Si ce n'est pas vous, sécurisez votre compte : {link}. | New device signed in — secure your account if this wasn't you. |
| N-21 | F-003/4/32 credential change | User | Push + email + SMS | Sécurité : modification effectuée | Votre {credential : mot de passe / PIN / appareil} a été {action : modifié / révoqué} le {date}. Si ce n'est pas vous : {support_link}. | Your {credential} was changed. |
| N-22 | F-020/33 settlement acct / recovery | All Owner+Admin | Push + email + SMS | Alerte sécurité | {action : « Compte de règlement modifié — actif dans 24 h » / « Récupération de compte effectuée »} sur {merchant}. Si ce n'est pas vous, contactez-nous immédiatement : {support_link}. | Security alert: {action}. Contact us immediately if this wasn't you. |
| N-23 | F-024/25 live key change | Owner | Email | Clé API réelle {action} | Une clé API réelle a été {action : créée / révoquée} sur {merchant} par {actor} le {date}. | A live API key was {action}. |
| N-24 | F-058 failing 24 h | Developer role | Push + email + bell | Webhook en échec | Votre point de terminaison {url} échoue depuis 24 h (dernier code : {status_code}). Il sera désactivé après 7 jours d'échecs : {link}. | Your webhook endpoint has been failing for 24 h. |
| N-25 | F-058 auto-disabled | Developer + Owner | Email + bell | Webhook désactivé | {url} a été désactivé après 7 jours d'échecs. Réactivez-le puis relivrez les événements manqués : {link}. | Endpoint auto-disabled after 7 days of failures. |
| N-26 | F-048 suspension | Owner | Email | Compte suspendu | L'accès de {merchant} est suspendu : {scope}. Motif : {reason_public}. Contactez-nous : {support_link}. | Your account has been suspended. |
| N-27 | F-050 adjustment on merchant balance | Finance + Owner | Email + bell | Ajustement sur votre solde | Un ajustement de {sign}{amount} FCFA a été appliqué à votre solde (réf {reference}) : {reason_public}. Détails : {link}. | A balance adjustment of {amount} FCFA was applied. |
| N-28 | F-051 prolonged outage | Affected merchants | Email + bell | Incident opérateur en cours | Les paiements {channel} sont indisponibles depuis {start_time}. Vos encaissements {other_channel} fonctionnent normalement. Suivi : {status_link}. | {channel} payments are currently unavailable. |
| N-29 | F-052 info request | Owner | Email | Informations requises | Pour continuer à utiliser Ijim Pay, merci de nous fournir : {items} avant le {deadline}. Répondez ici : {link}. | We need additional information by {deadline}. |
| N-30 | F-035–44 payer receipt | Payer | SMS + WhatsApp (optional email) | — | Paiement de {amount} FCFA à {merchant} confirmé ✓. Reçu : {receipt_url} — via Ijim Pay. | Payment of {amount} FCFA to {merchant} confirmed — receipt: {url}. |
| N-31 | F-012 refund to payer | Payer | SMS | — | {merchant} vous a remboursé {amount} FCFA sur votre compte {channel}. Réf {reference} — via Ijim Pay. | {merchant} refunded you {amount} FCFA. |
| N-32 | F-029/43/60 cycle pre-notice | Customer | SMS + WhatsApp | — | Rappel : votre abonnement {plan} chez {merchant} ({amount} FCFA) sera prélevé demain. Gardez votre compte {channel} approvisionné. — via Ijim Pay | Reminder: your {plan} subscription charge is due tomorrow. |
| N-33 | F-030 manual dunning | Customer | SMS + WhatsApp | — | {merchant} : votre paiement de {amount} FCFA pour {plan} est en attente. Payez en toute sécurité ici : {link} — via Ijim Pay | Your {plan} payment is pending — pay securely here: {link}. |
| N-34 | F-060 auto dunning (past_due) | Customer | SMS + WhatsApp | — | Votre abonnement {plan} chez {merchant} est impayé ({amount} FCFA). Régularisez avant le {deadline} pour éviter l'annulation : {link} — via Ijim Pay | Your subscription is past due — pay before {deadline} to avoid cancellation. |
| N-35 | Daily (digest opt-out per user) | Merchant users | Email | Votre journée Ijim Pay — {date} | Encaissé : {total} FCFA ({count} paiements, {success_rate} % de réussite). Échecs : {failed_count}. Abonnements impayés : {past_due_count}. Solde disponible : {available} FCFA. Détails : {link}. | Daily digest: {total} FCFA collected, {count} payments. |

---

## API additions needed (referenced as `[API-ADD-n]` above; none invented inline)

| Ref | Method + path | Purpose |
|---|---|---|
| API-ADD-1 | `POST /auth/otp` · `POST /auth/otp/verify` · `POST /auth/login` · `POST /auth/password_reset` | Dashboard/app auth (OTP send/verify, login incl. TOTP step, password reset) |
| API-ADD-2 | `PUT /merchant/profile` · `POST /merchant/kyb_documents` · `POST /merchant/kyb/submit` | Onboarding drafts, document upload, KYB submission |
| API-ADD-3 | `PATCH /payment_links/{id}` | Edit / re-activate a link |
| API-ADD-4 | `POST /exports` · `GET /exports/{id}` | Async statement/CSV export jobs |
| API-ADD-5 | `POST /payroll_lists` · `GET/PATCH/DELETE /payroll_lists/{id}` · `POST /payroll_lists/{id}/schedule` | Saved payroll lists + scheduled runs |
| API-ADD-6 | `POST /payout_batches/{id}/reject` | Checker rejection with reason |
| API-ADD-7 | `POST /balance_transfers` | Internal transfer available → payout wallet |
| API-ADD-8 | `POST /invites` · `POST /invites/{id}/accept` · `PATCH/DELETE /members/{user_id}` | Team invite, role change, removal |
| API-ADD-9 | `POST /api_keys` · `POST /api_keys/{id}/revoke` | Key create/rotate/revoke (dashboard-auth) |
| API-ADD-10 | `POST /webhook_endpoints/{id}/enable` · `PATCH /webhook_endpoints/{id}` · `POST /events/{id}/redeliver` | Re-enable, edit, manual redelivery |
| API-ADD-11 | `POST /invoices/{id}/dunning_link` | Generate single-use dunning payment link |
| API-ADD-12 | `GET /devices` · `POST /devices/{id}/revoke` | Enrolled device list & revocation |
| API-ADD-13 | `POST /auth/recovery` | Account recovery with identity re-verification |
| API-ADD-14 | `GET /me/memberships` · `POST /me/switch` | Business list & context switch |
| API-ADD-15 | Ops (internal API): `POST /ops/kyb/{merchant_id}/approve|reject` · `PATCH /ops/merchants/{id}/tier` · `POST /ops/merchants/{id}/suspend` | KYB decisions, tier, suspension |
| API-ADD-16 | `GET/PUT /merchant/golive_checklist` | Persisted go-live checklist state |

## Icon additions needed (not in docs/07 §4 map — to be added there before use)

`rotate-cw` (retry/redeliver — already used in docs/08 prose) · `scale` (reconciliation — flagged in docs/08 §B) · `smartphone` (enrolled devices — used in docs/08 A10) · `plus` (app FAB — used in docs/10) · `upload` (CSV/report import) · `flag` (risk flag/case) · `circle-pause` / `circle-play` (subscription pause/resume) · `building-2` (business switcher) · `file-text` (statements/relevés) · `phone-call` (voice OTP fallback).

```
INVENTORY: flows_merchant=34 flows_payer=10 flows_ops=10 flows_system=11 flows_developer=4 flows_total=69 notifications=35
```
