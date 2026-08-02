# Ijim Pay — Mobile App Screens Spec (Flutter, Android-first)

Status: v1 · Supersedes `docs/10-mobile-screens.md` (its route-style slugs remain canonical ids). Numbering: screens `M-01…`, sheets/dialogs `MS-01…`. Design tokens, icons, components: `docs/07-brand-design-system.md`. API: `docs/02-api-spec.md` — anything not there is listed in **API additions needed** at the end, never invented inline. All copy FR-first; amounts always `12 500 FCFA` (thin-space thousands, currency after).

## Global conventions (apply to every screen unless overridden)

- **Bottom nav** (5): Accueil `layout-dashboard` · Liens `link` · **Encaisser `hand-coins`** (raised 64px brand-600 circle, white icon, elevation on container only) · Activité `arrow-left-right` · Menu `settings`.
- **Offline classes** (docs/10 §8): `cached-read` (renders Drift cache < 0.5 s + `cloud-off` chip "Hors ligne — données de {HH:mm}"), `queued-write` (non-money creates queue locally, auto-flush toast "1 lien créé hors ligne a été publié"), `blocked-write` (money actions refused offline: error-sheet MS-01 "Impossible hors connexion — les encaissements et paiements nécessitent le réseau.").
- **Hardware back**: pops the local stack; on a bottom-nav root, first press returns to Accueil, second press (on Accueil, within 2 s) exits with toast "Appuyez encore pour quitter". Money-confirm screens intercept back with their cancel confirm. Sheets: back = dismiss sheet.
- **FLAG_SECURE**: `yes` on balance, payout/approval, security, PIN and receipt screens (stated per screen); default `no`.
- **TalkBack**: every icon-only control has a `Semantics` label (given per screen); min target 48dp; font-scale 1.3 without truncation.
- **StatusBadge**: only the 8 canonical statuses — succeeded, pending, failed, expired, refunded, processing, pending_approval (warning-600 *outlined* per docs/07 §2 collision rule 2), draft shown as processing? No — draft renders `processing` label "Brouillon" is web-only; the app shows batches only from `pending_approval` onward. `accent-500` never appears in transactional UI.
- **Root/jailbreak check** at launch: warning banner "Appareil non sécurisé détecté — les paiements sortants sont désactivés." + payout actions hidden.
- **App-lock**: after 2 min in background (configurable in M-43: 30 s / 2 min / 5 min), any foreground resume routes through M-50.
- **Roles**: Owner, Admin, Developer, Finance, Viewer. Viewer = read-only (buttons hidden, not disabled). Payout approval = Owner/Admin (never the maker). Refund = Finance/Admin/Owner.

### Drift cache tables (SQLite via Drift)

| Table | Contents | Written by | Read by (screens) |
|---|---|---|---|
| `charges_cache` | last 200 charges (`GET /charges` fields) | ACT, HOME sync | M-12, M-26, M-27, M-28 |
| `links_cache` | payment links (`GET /payment_links`) | LINKS sync | M-21, M-24, M-25 |
| `balance_cache` | `GET /balance` snapshot + ts | HOME/MENU sync | M-12, M-37 |
| `ledger_cache` | last 100 `GET /balance_transactions` | M-37 sync | M-37 |
| `settlements_cache` | `GET /settlements` list + details | MENU sync | M-38, M-39 |
| `batches_cache` | `GET /payout_batches` summaries | PAYOUT sync | M-29, M-30 |
| `beneficiaries_cache` | beneficiary directory | PAYOUT sync | M-32, M-34, MS-12 |
| `channels_cache` | `GET /channels` health + history | HOME sync | M-12, M-14, M-16 |
| `notifications_cache` | notification tray items | push + sync | M-13 |
| `merchant_cache` | merchant profile, tier, kyb_status | session sync | M-11, M-12, M-41, M-42 |
| `team_cache` | members list | MENU sync | M-40 |
| `recent_payers` | last 10 payer phones (local-only, opt-out M-44) | M-16 | M-16 |
| `outbox_queue` | queued non-money writes (link creates) | M-22 offline | flush worker, M-21 badge |
| `prefs` | locale, sound, biometric flag, lock timeout, printer MAC | settings screens | all |

### Push-notification routing table (FCM; all routes survive cold start; unknown type → M-13)

| Push type (data.type) | Event source | Route (cold or warm) | Notes |
|---|---|---|---|
| `charge.succeeded` | webhook event | M-18 if that charge is awaited on M-17, else M-27 (tx-detail) | high-priority channel, cash-register sound + haptic |
| `charge.failed` / `charge.expired` | event | M-19 if awaited, else M-27 | default channel |
| `refund.succeeded` | event | M-27 of the refunded charge | |
| `payout_batch.pending_approval` | event | M-30 (approval-detail) | approver roles only |
| `payout_batch.completed` / `partially_failed` | event | M-29 → Historique tab, batch row expanded | |
| `payout.succeeded` / `payout.failed` | event | M-29 → Historique tab | |
| `settlement.paid` | event | M-39 (settlement-detail) | |
| `balance.topup.received` | event | M-35 success state (if open) else M-37 | |
| `kyb.decision` | KYB review (API addition) | M-11 (verification-status) | also SMS+email per docs/08 §C |
| `security.new_device` | API addition | M-43 → Appareils | |
| `team.member_changed` | API addition | M-40 | bell only, silent |
| `invoice.payment_failed` / `subscription.past_due` | event | M-13 (tray; subscriptions are web-managed v1) | |

Deep links: `ijimpay://tx/{charge_id}` → M-27 · `ijimpay://batch/{id}` → M-30 · `ijimpay://settlement/{id}` → M-39 · `ijimpay://verification` → M-11 · `ijimpay://link/{id}` → M-24. All pass through auth gate (M-50/M-07) first, target preserved.

---

## AUTH

### M-01 — Écran de démarrage / Splash
- **Route**: `splash` · **Icon**: lucide — (logo mark) · **Access**: all · **Purpose**: boot, decide next route (session? min-version? maintenance?).
- **Layout zones**: full-bleed brand-900 surface; centered logo; bottom version caption.
- **Data displayed**: app version (local); minimum version + maintenance flag (`GET /app/config` — API addition); session validity (local token).
- **Inputs**: aucun.
- **Actions**: aucune action utilisateur (auto-route ≤ 1,5 s : session valide → M-50/M-12 ; sinon → M-02 ; version < minimum → M-48 ; maintenance → M-49).
- **Modals/sheets**: —
- **States**: loading (logo pulse); error (config fetch fails → proceed with cached config, offline-tolerated); no empty/permission states.
- **System**: Gestes: aucun · Offline: cached-read (boots on cache) · FLAG_SECURE: no · Back: exits app · TalkBack: "Ijim Pay, chargement" · Deep-link: intercepts and stores pending route.
- **Events**: `app_open` analytics.

### M-02 — Bienvenue / Welcome
- **Route**: `welcome` · **Icon**: lucide `globe` (top-right language) · **Access**: unauthenticated · **Purpose**: entry choice between signup and login.
- **Layout zones**: header (language toggle) · hero (logo + tagline "Encaissez, simplement." + illustration) · footer (two buttons).
- **Data displayed**: none (static).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer un compte | — | — | → M-03 (mode signup) |
| Se connecter | — | — | → M-03 (mode login) ; si compte connu sur l'appareil → M-07 |
| Changer de langue | `globe` | — | bascule FR/EN, persiste dans `prefs` |

- **Modals/sheets**: —
- **States**: static only.
- **System**: Gestes: aucun · Offline: fully static · FLAG_SECURE: no · Back: exits app · TalkBack: `globe` = "Changer de langue" · Deep-link: n/a.
- **Events**: —

### M-03 — Numéro de téléphone / Phone entry
- **Route**: `phone-entry` · **Icon**: lucide `arrow-left` (back) · **Access**: unauthenticated · **Purpose**: capture the phone number and send OTP.
- **Layout zones**: header (back) · content (title "Votre numéro de téléphone", field, legal links ToS/Confidentialité) · footer (CTA).
- **Data displayed**: none.

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Numéro de téléphone | tel, préfixe `+237` fixe, format national `6 70 00 00 00` | 9 chiffres, commence par 6 ; préfixes MTN 650–654/67x/680–684 ou Orange 655–659/69x/685–689 | vide | "Entrez un numéro camerounais valide (commençant par 6)." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Continuer | — | — | `POST /auth/otp/request` (API addition) → M-04 ; désactivé tant que numéro invalide |

- **Modals/sheets**: MS-01 on network/API error.
- **States**: loading (button spinner, same width); error inline (invalid prefix), rate-limited → "Trop de tentatives. Réessayez dans {n} min."; offline → blocked-write MS-01.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: no · Back: → M-02 · TalkBack: field "Numéro de téléphone, préfixe +237" · Deep-link: n/a.
- **Events**: `otp_requested`.

### M-04 — Code de vérification / OTP
- **Route**: `otp` · **Icon**: lucide `message-square` · **Access**: unauthenticated · **Purpose**: verify the 6-digit SMS code (auto-read via SMS Retriever, no SMS permission needed).
- **Layout zones**: header (back) · content (title "Code envoyé au +237 6 70 00 00 00", 6 digit boxes, resend row) · numeric keyboard.
- **Data displayed**: masked phone (local), resend countdown (local timer 30 s).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Code à 6 chiffres | otp (6 cases, auto-avance, auto-lecture SMS) | 6 chiffres exactement | vide | "Code incorrect. Il vous reste {n} essais." / "Code expiré — demandez-en un nouveau." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Renvoyer le code | — | — | `POST /auth/otp/request` ; actif après 30 s ("Renvoyer dans 0:{ss}") |
| Modifier le numéro | `pencil` | — | → M-03 (numéro pré-rempli) |

- Auto-submit at 6th digit: `POST /auth/otp/verify` (API addition) → nouveau compte → M-05 ; compte existant → M-05 si pas de PIN sinon M-07.
- **Modals/sheets**: MS-01.
- **States**: loading (boxes shimmer); wrong (boxes shake + haptic, error line); 5 wrong → cooldown state "Trop d'essais. Réessayez dans 15 min." with disabled input; expired.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: no · Back: → M-03 · TalkBack: "Case {i} sur 6 du code de vérification" · Deep-link: n/a.
- **Events**: `otp_verified` / `otp_failed`.

### M-05 — Créer votre PIN / Create PIN
- **Route**: `create-pin` · **Icon**: lucide `lock` · **Access**: authenticated, no PIN set · **Purpose**: set the 4-digit app PIN (entered twice).
- **Layout zones**: content (title, explainer "Votre PIN protège l'application", 4 dots) · custom PIN pad (0–9, `delete`).
- **Data displayed**: step indicator (1/2 "Choisissez un PIN" → 2/2 "Confirmez votre PIN").

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| PIN (2 saisies) | pavé 4 chiffres | 4 chiffres ; refuse 0000, 1234, 1111…9999 (suites/répétitions) ; 2ᵉ saisie identique | vide | "Ce PIN est trop simple — choisissez-en un autre." / "Les deux PIN ne correspondent pas. Recommencez." |

- Auto-advance; on success `POST /auth/pin/set` (API addition, PIN dérivé/haché côté client) → M-06.
- **Modals/sheets**: MS-01.
- **States**: mismatch (dots shake, restart at step 1); loading.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: **yes** · Back: step 2 → step 1 ; step 1 → confirm exit of signup (system dialog) · TalkBack: pad keys "Chiffre {n}", `delete` = "Effacer" · Deep-link: n/a.
- **Events**: `pin_created`.

### M-06 — Activer la biométrie / Biometric opt-in
- **Route**: `biometric-optin` · **Icon**: lucide `fingerprint` · **Access**: authenticated · **Purpose**: offer biometric unlock instead of PIN.
- **Layout zones**: hero (`fingerprint` illustration) · copy "Déverrouillez plus vite avec votre empreinte" · footer (2 buttons).
- **Data displayed**: device biometric capability (local BiometricManager); screen skipped if unsupported.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Activer | `fingerprint` | — | prompt système ; succès → `prefs.biometric=true` → M-08 (nouveau marchand) ou M-12 |
| Plus tard | — | — | → M-08 / M-12 ; PIN seul |

- **Modals/sheets**: system biometric prompt only.
- **States**: biometric enrollment missing → "Aucune empreinte enregistrée sur cet appareil. Ajoutez-en une dans les réglages Android." + [Plus tard].
- **System**: Gestes: aucun · Offline: local only · FLAG_SECURE: no · Back: = Plus tard · TalkBack: hero "Illustration empreinte digitale" · Deep-link: n/a.
- **Events**: `biometric_enabled` / `biometric_skipped`.

### M-07 — Connexion / Login (returning)
- **Route**: `login-pin` · **Icon**: lucide `lock` / `fingerprint` · **Access**: known device, session refresh needed · **Purpose**: re-authenticate with PIN or biometric.
- **Layout zones**: header (avatar + merchant/user name) · 4 dots + PIN pad · footer ("Changer de compte").
- **Data displayed**: user name + merchant name (`merchant_cache`).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| PIN | pavé 4 chiffres | 4 chiffres | vide | "PIN incorrect. Il vous reste {n} essais." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Biométrie | `fingerprint` | — | prompt système ; succès → refresh JWT silencieux → destination (pending deep-link ou M-12) |
| Changer de compte | — | — | ouvre MS-16 |
| PIN oublié ? | — | — | → M-03 (ré-auth OTP complète) ; visible après 1 échec |

- **Modals/sheets**: MS-16, MS-01.
- **States**: 5 wrong PINs → forced OTP re-auth (M-03) with copy "Trop d'essais — vérifiez votre identité par SMS."; token refresh fails → same; offline + valid cached session → app opens read-only (money writes blocked anyway).
- **System**: Gestes: aucun · Offline: PIN verified locally, cached session honored for reads · FLAG_SECURE: **yes** · Back: exits app · TalkBack: `fingerprint` = "Se connecter avec la biométrie" · Deep-link: gate — holds and forwards pending route.
- **Events**: `login_success` / `login_pin_failed`.

---

## ONBRD

### M-08 — Votre entreprise / Business info
- **Route**: `business-info` · **Icon**: lucide `badge-check` (wizard header) · **Access**: Owner (creator) · **Purpose**: step 1/3 of KYB — business profile.
- **Layout zones**: wizard header (progress 1/3, "Plus tard" skip to app in test mode) · scrollable form · footer CTA.
- **Data displayed**: prefilled draft (`merchant_cache` if resuming; server draft via `GET /merchant` — API addition).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom commercial | texte | 2–80 caractères | vide | "Entrez le nom de votre entreprise." |
| Raison sociale | texte | 2–120 caractères ; requis si forme ≠ "Particulier" | vide | "Entrez la raison sociale figurant sur votre RCCM." |
| Forme juridique | sélecteur (Particulier / ETS / SARL / SA / GIC / Association) | requis | Particulier | "Choisissez une forme juridique." |
| Secteur d'activité | sélecteur (liste fermée : Commerce, Restauration, Services, Transport, Éducation, Autre…) | requis | vide | "Choisissez votre secteur." |
| Ville | texte | 2–60 caractères | vide | "Entrez votre ville." |
| Adresse / quartier | texte | 2–120 caractères | vide | "Entrez votre adresse ou quartier." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Continuer | — | Owner | `PATCH /merchant` (API addition) → M-09 |
| Plus tard | — | Owner | sauvegarde brouillon local → M-12 (mode test, bannière KYB) |

- **Modals/sheets**: MS-01.
- **States**: loading (skeleton form); per-field errors; resume state prefilled.
- **System**: Gestes: aucun · Offline: draft saved locally (queued-write of the draft only; submit is blocked-write) · FLAG_SECURE: no · Back: confirm "Quitter la vérification ? Vos réponses sont enregistrées." · TalkBack: standard field labels · Deep-link: `ijimpay://verification` resumes wizard at first incomplete step.
- **Events**: `onboarding_step_completed(1)`.

### M-09 — Capture des documents / Doc capture (per-doc loop)
- **Route**: `doc-capture` (repeats per doc) · **Icon**: lucide `camera` · **Access**: Owner/Admin · **Purpose**: step 2/3 — guided capture of RCCM, pièce d'identité, preuve de compte, with retake loop.
- **Layout zones**: doc checklist header (chips: RCCM · CNI recto · CNI verso · Preuve de compte, each with StatusBadge-style check) · camera viewport with frame overlay + torch `flashlight` · shutter bar · review pane after shot.
- **Data displayed**: required doc list by legal form (`GET /kyb_documents` — API addition); per-doc status (à faire / capturé / envoyé / rejeté + motif).
- **Inputs**: aucun champ texte — capture photo uniquement (ou import `image` depuis la galerie pour le RCCM PDF/photo).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Prendre la photo | `camera` | Owner/Admin | capture ; analyse locale flou/reflets ; si mauvaise → MS-18 (reprise) ; si bonne → aperçu |
| Importer un fichier | `image` | Owner/Admin | sélecteur système (jpg/png/pdf ≤ 10 Mo) |
| Utiliser cette photo | `circle-check` | Owner/Admin | `POST /kyb_documents` (API addition, multipart) ; passe au doc suivant |
| Reprendre | `rotate-cw` | Owner/Admin | retour au viseur |
| Lampe | `flashlight` | — | toggle torche |
| Continuer | — | Owner/Admin | actif quand tous les docs requis sont envoyés → M-10 |

- **Modals/sheets**: MS-03 (camera permission, shown before first viewport), MS-18, MS-01.
- **States**: permission-denied (full-pane state: "L'appareil photo est nécessaire pour photographier vos documents." + [Ouvrir les réglages]); upload progress per doc; upload failed (retry chip on the doc); rejected doc (red chip + reason from ops, tap → recapture that doc only); empty n/a.
- **System**: Gestes: pinch-to-zoom in viewport; tap-to-focus · Offline: blocked-write ("La vérification nécessite une connexion.") but captured photos persist locally and upload resumes · FLAG_SECURE: **yes** (identity docs) · Back: from review → viewfinder; from viewfinder → M-08 with confirm · TalkBack: shutter = "Prendre la photo", torch = "Activer la lampe" · Deep-link: `ijimpay://verification` if docs incomplete.
- **Events**: `kyb_doc_uploaded(type)`.

### M-10 — Compte de règlement / Settlement account
- **Route**: `settlement-account` · **Icon**: lucide `landmark` · **Access**: Owner · **Purpose**: step 3/3 — where T+1 settlements are sent.
- **Layout zones**: wizard header (3/3) · form · info Banner "Ce compte doit être au nom de votre entreprise ou de son propriétaire." · footer CTA.
- **Data displayed**: channel auto-detect from prefix (docs/07 §8 mapping).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Type de compte | segmenté (MTN MoMo / Orange Money / Banque) | requis | auto selon numéro utilisateur | — |
| Numéro (MoMo/OM) | tel `+237` fixe | 9 chiffres, préfixe cohérent avec le type choisi | numéro du compte utilisateur | "Ce numéro ne correspond pas à un compte {canal}." |
| Nom du titulaire | texte | 2–80 caractères | raison sociale | "Entrez le nom du titulaire du compte." |
| IBAN / n° de compte (si Banque) | texte | 10–34 caractères alphanum | vide | "Entrez un numéro de compte valide." |
| Banque (si Banque) | sélecteur (liste banques CM) | requis si type Banque | vide | "Choisissez votre banque." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Terminer | — | Owner | `PATCH /merchant` (settlement fields) → M-11 |

- **Modals/sheets**: MS-01.
- **States**: loading; field errors; offline blocked-write.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: **yes** · Back: → M-09 · TalkBack: standard · Deep-link: wizard resume.
- **Events**: `onboarding_step_completed(3)`.

### M-11 — Statut de vérification / Verification status
- **Route**: `verification-status` · **Icon**: lucide `badge-check` · **Access**: all roles · **Purpose**: show KYB tier, per-doc status, and the switch-to-live path.
- **Layout zones**: tier badge hero (Niveau 0/1/2 + limites en clair) · per-doc status chips list · timeline copy "En attente d'examen — environ 24 h" · footer CTA.
- **Data displayed**: `merchant_cache`: tier, kyb_status (`GET /merchant`); per-doc status + rejection reason (`GET /kyb_documents`).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Explorer en mode test | `flask-conical` | — | → M-12 (mode test) |
| Reprendre un document rejeté | `camera` | Owner/Admin | → M-09 ciblé sur ce document |
| Passer en mode réel | — | Owner/Admin | visible si Tier ≥ 1 : bascule le mode live (`prefs`) → M-12 |

- **Modals/sheets**: MS-01.
- **States**: pending review (default); approved (success surface brand-100, confetti-free ✓, push arrives here); rejected (danger Banner + reasons per doc); loading skeleton; offline cached-read.
- **System**: Gestes: pull-to-refresh · Offline: cached-read · FLAG_SECURE: no · Back: → M-12 · TalkBack: chips "Document {nom}, statut {statut}" · Deep-link: `ijimpay://verification`; push `kyb.decision` lands here.
- **Events**: mark KYB notification read.

---

## HOME

### M-12 — Accueil / Home
- **Route**: `home` · **Icon**: lucide `layout-dashboard` · **Access**: all roles · **Purpose**: today's money at a glance + quick entry into the core loop.
- **Layout zones**: header (merchant name + switcher chevron if multi, test/live pill, `bell` with unread badge) · KYB/test Banner (amber, `flask-conical`, "Mode test — aucun argent réel" + CTA "Passer en réel" once Tier 1) · hero card · balance row · provider strip · quick actions · pending-approval card (approver roles) · latest transactions list.
- **Data displayed**: Encaissé aujourd'hui = sum of today's succeeded charges (`GET /charges?status=succeeded&created[gte]=today` aggregated client-side; server stat endpoint in API additions); tx count + success-rate ring (same source); Disponible / En attente (`GET /balance` → `available[]`, `pending[]`); provider dots (`GET /channels` → mtn_momo/orange_money status); pending approvals count (`GET /payout_batches?status=pending_approval`); latest 5 TxRows (`GET /charges?limit=5`: status, customer.phone/name, description, channel, amount, created_at).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Notifications | `bell` | — | → M-13 |
| Encaisser | `hand-coins` | Owner/Admin/Finance | → M-15 |
| Créer un lien | `link` | Owner/Admin/Finance | ouvre M-22 (sheet) |
| Voir le QR du comptoir | `qr-code` | — | QR statique du marchand plein écran (variante de M-20, montant libre) |
| Valider les paiements | `check-check` | Owner/Admin | carte "2 lots attendent votre validation" → M-29 onglet Validations |
| Voir le solde | `wallet` | — | tap ligne solde → M-37 |
| Détail opérateur | `activity` | — | tap pastille MTN/Orange → M-14 |
| Changer de marchand | `chevron-right` | multi-merchant users | sheet switcher (liste des marchands, re-sync) |

- **Modals/sheets**: M-22 (as sheet), MS-01, MS-02 (notification permission — asked on first Home after login, Android 13+).
- **States**: loading (skeleton tiles + 5 skeleton rows); empty (new merchant: checklist card "Vérifiez votre entreprise → Créez votre premier lien → Encaissez votre premier paiement", each row with done-state check); error (error strip + retry); offline (cached-read, `cloud-off` chip "données de 14:02"); provider degraded (warning Banner "Les paiements Orange peuvent être lents actuellement.").
- **System**: Gestes: pull-to-refresh; hero tap → M-26 filtré Aujourd'hui · Offline: cached-read < 0.5 s · FLAG_SECURE: no (amounts visible but screen is the app's face; balance masked behind `eye` toggle, persisted) · Back: double-press exit · TalkBack: `bell` = "Notifications, {n} non lues"; balance eye = "Masquer les montants" · Deep-link: nav root.
- **Events**: fires MS-02 permission request once; `home_viewed`.

### M-13 — Notifications
- **Route**: `notifications` · **Icon**: lucide `bell` · **Access**: all roles · **Purpose**: notification tray grouped by day, deep-link per item.
- **Layout zones**: header (back, title "Notifications", action "Tout marquer lu") · grouped list (Aujourd'hui / Hier / date headers).
- **Data displayed**: items from `notifications_cache` fed by push + `GET /notifications` (API addition): icon per type (same mapping as push table), title, body, time, read state.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Tout marquer lu | `circle-check` | — | `POST /notifications/mark_all_read` (API addition), optimistic |
| Ouvrir une notification | — | — | route per push table (tx → M-27, lot → M-30, règlement → M-39, KYB → M-11…) ; marque lue |

- **Modals/sheets**: MS-01.
- **States**: loading skeleton; empty "Aucune notification pour l'instant." (illustration `bell`); offline cached-read; error retry.
- **System**: Gestes: pull-to-refresh · Offline: cached-read · FLAG_SECURE: no · Back: → M-12 · TalkBack: rows "Notification : {titre}, {heure}, {lue|non lue}" · Deep-link: push tap with unknown type lands here.
- **Events**: read-state sync.

### M-14 — État des opérateurs / Provider status detail
- **Route**: `provider-status-detail` · **Icon**: lucide `activity` · **Access**: all roles · **Purpose**: plain-language provider health + recent history.
- **Layout zones**: per-provider cards (MTN, Orange) · history section (7 derniers jours) · help footer.
- **Data displayed**: `GET /channels`: status operational|degraded|down per provider, rendered as "● Opérationnel" (success-600) / "● Ralenti" (warning-600) / "● Indisponible" (danger-600); history entries (`GET /channels/{channel}/history` — API addition): date, incident copy FR.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Actualiser | `rotate-cw` | — | refetch `GET /channels` |

- **Modals/sheets**: MS-01.
- **States**: loading; degraded copy "Les paiements {opérateur} peuvent être lents ou échouer. Vous pouvez proposer l'autre canal."; down copy "{Opérateur} est indisponible. Les encaissements sur ce canal sont suspendus."; offline cached-read with timestamp; empty history "Aucun incident ces 7 derniers jours."
- **System**: Gestes: pull-to-refresh · Offline: cached-read · FLAG_SECURE: no · Back: → M-12 · TalkBack: dots carry text labels already · Deep-link: from provider Banner anywhere.
- **Events**: —

---

## CHARGE (core loop, target < 15 s)

### M-15 — Montant / Amount pad
- **Route**: `amount-pad` · **Icon**: lucide `hand-coins` · **Access**: Owner/Admin/Finance (Viewer/Developer: tab hidden) · **Purpose**: enter the amount to collect, full-screen, one hand.
- **Layout zones**: header (close `x`, test/live pill) · amount display (live-formatted 40px tabular "12 500 FCFA") · recent-amounts chips row (last 4 local) · numeric pad (0–9, `delete`, "00") · footer CTA.
- **Data displayed**: recent amounts (`prefs`/local).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Montant | pavé numérique | entier XAF ; ≥ 100 FCFA ; ≤ plafond du palier (Tier 1/2, depuis `merchant_cache`) | 0 | "Montant minimum : 100 FCFA." / "Montant au-dessus de votre limite ({limite} FCFA). Passez au niveau supérieur dans Menu → Vérification." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Continuer | — | idem écran | → M-16 avec le montant ; désactivé si montant invalide |
| Montant récent | — | — | chip → remplit le montant |
| Effacer | `delete` | — | retire le dernier chiffre ; appui long = tout effacer |

- **Modals/sheets**: —
- **States**: offline: pad usable but Continuer shows MS-01 blocked-write copy at M-16 submit; no loading/empty.
- **System**: Gestes: long-press `delete` clears · Offline: entry allowed, creation blocked later · FLAG_SECURE: no · Back: → previous tab · TalkBack: display "Montant : {montant} francs CFA", `delete` = "Effacer le dernier chiffre" · Deep-link: nav center tab.
- **Events**: —

### M-16 — Canal & payeur / Channel & payer
- **Route**: `channel-payer` · **Icon**: lucide `hand-coins` · **Access**: same as M-15 · **Purpose**: capture payer phone, auto-detect channel, send the USSD push (or switch to QR).
- **Layout zones**: header (back, amount recap chip) · phone field + recent payers chips · detected ChannelChip row with "Changer" · reference optionnelle (repliée) · footer (primary CTA + secondary "Afficher le QR").
- **Data displayed**: recent payers (`recent_payers`, opt-out in M-44); channel health (`channels_cache`) — down channel disabled with `triangle-alert` tooltip "Orange Money est indisponible actuellement".

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Numéro du client | tel `+237` fixe | 9 chiffres commençant par 6 ; préfixe reconnu MTN/Orange sinon choix manuel du canal requis | vide | "Entrez le numéro Mobile Money du client." / "Préfixe inconnu — choisissez le canal manuellement." |
| Nom du client (optionnel) | texte | ≤ 60 caractères | vide | — |
| Référence (optionnel) | texte | ≤ 40 caractères, unique par marchand (409 → erreur) | auto `POS-{yyyyMMdd}-{n}` | "Cette référence existe déjà." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer la demande | — | idem | `POST /charges` (Idempotency-Key UUID, `{amount, currency:"XAF", channel, customer:{phone,name}, reference}`) → 202 → M-17 ; double-tap verrouillé (bouton lock 3 s) |
| Afficher le QR | `qr-code` | — | `POST /payment_links` (single-use, fixed amount) → M-20 |
| Changer de canal | — | — | révèle les 2 ChannelChips (radio) |

- **Modals/sheets**: MS-01; MS-04 (contacts permission) via champ "choisir dans les contacts" `book-user`.
- **States**: loading (CTA spinner); channel down (chip disabled + tooltip); rate-limited (`rate_limited` / 3 pending per payer): "Une demande est déjà en cours sur ce numéro — demandez au client de la valider ou attendez 5 min."; offline blocked-write: "Impossible hors connexion — l'encaissement nécessite le réseau."
- **System**: Gestes: aucun · Offline: blocked-write (never queued) · FLAG_SECURE: no · Back: → M-15 (montant conservé) · TalkBack: `qr-code` = "Afficher le code QR à la place", `book-user` = "Choisir dans les contacts" · Deep-link: n/a.
- **Events**: `charge_created`.

### M-17 — En attente du client / Charge waiting
- **Route**: `charge-waiting` · **Icon**: lucide `clock` (pulsing) · **Access**: creator flow · **Purpose**: live wait for payer approval with USSD instructions.
- **Layout zones**: hero (pulsing `clock` in brand-100 circle, amount XL, payer number) · instruction card per channel (MTN : "Demandez au client de valider la notification, ou de composer **\*126\*1#** puis son code MoMo." · Orange : "Demandez au client de composer **\*150\*4\*4#** puis son code Orange Money." — codes served by config, jamais codés en dur) · expiry countdown ring (5 min, `charge.expires_at`) · footer actions.
- **Data displayed**: `GET /charges/{id}` polled 3 s: status, expires_at, failure_code; payer phone (local).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Renvoyer la demande | `rotate-cw` | idem | visible dès 2 min, 1 fois max : `POST /charges/{id}/cancel` (API addition) + nouveau `POST /charges` |
| Annuler | `x` | idem | confirm inline "Annuler cette demande ?" → `POST /charges/{id}/cancel` → M-15 |

- Progressive copy: 15 s "Demandez au client de valider…" · 45 s "Toujours en attente — vérifiez que le client a bien reçu la demande." · 2 min ajoute [Renvoyer la demande].
- Terminal transitions: `succeeded` → M-18 · `failed` → M-19 · `expired` → M-19 (raison "La demande a expiré").
- **Modals/sheets**: MS-01 (poll failures after 3 consecutive misses: "Connexion instable — le statut sera mis à jour dès le retour du réseau. La notification sonnera au paiement.").
- **States**: pending (default, live); network-lost (banner but screen stays; FCM will still deliver terminal state); expired.
- **System**: Gestes: aucun · Offline: read degrades to push-driven; no writes · FLAG_SECURE: no · Back: = Annuler (with confirm) · TalkBack: ring "Temps restant : {m} minutes {s} secondes" (announce each 30 s) · Deep-link: push `charge.succeeded/failed` targets this charge → routes to M-18/M-19 even if app was killed mid-wait.
- **Events**: screen-kept-awake (wakelock) while pending.

### M-18 — Paiement reçu / Charge success
- **Route**: `charge-success` · **Icon**: lucide `circle-check` · **Access**: creator flow / push · **Purpose**: unambiguous market-stall confirmation — flash + sound + haptic.
- **Layout zones**: full-screen brand-600 flash → brand-100 surface, check scale-in, amount 40px, payer phone, ref, time · footer actions · auto-return countdown "Retour dans 5 s".
- **Data displayed**: charge: amount, customer.phone, reference, succeeded_at, fee/net (from `GET /charges/{id}`).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Partager le reçu | `share-2` | — | → M-28 (receipt preview/share) |
| Nouvel encaissement | `hand-coins` | idem M-15 | → M-15 (annule l'auto-retour) |

- **Modals/sheets**: MS-05 via M-28.
- **States**: single state; sound plays only if `prefs.sound=true` (toggle in M-44); auto-return to M-15 after 5 s unless touched.
- **System**: Gestes: any tap cancels auto-return · Offline: fully renderable from push payload + cache · FLAG_SECURE: no · Back: → M-15 · TalkBack: announces immediately "Paiement reçu : {montant} francs CFA de {numéro}" (live region, assertive) · Deep-link: push `charge.succeeded` while M-17 open lands here; reachable cold-start.
- **Events**: cash-register sound + haptic (signature moment, docs/07 §7); writes charge to `charges_cache`.

### M-19 — Paiement échoué / Charge failed
- **Route**: `charge-failed` · **Icon**: lucide `circle-x` · **Access**: creator flow / push · **Purpose**: explain the failure in plain FR and offer the fastest retry.
- **Layout zones**: hero (`circle-x` danger tint, amount, payer) · reason card · footer (3 actions).
- **Data displayed**: failure_code mapped: `insufficient_payer_funds` → "Solde insuffisant sur le compte du client." · `payer_rejected` → "Le client a refusé le paiement." · `payer_timeout`/expired → "La demande a expiré sans réponse." · `payer_not_found` → "Ce numéro n'a pas de compte {canal}." · `channel_unavailable` → "{Opérateur} est indisponible actuellement." · `provider_error` → "Erreur chez l'opérateur — réessayez."
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer | `rotate-cw` | idem | nouveau `POST /charges` mêmes détails → M-17 |
| Changer de canal | — | idem | → M-16 avec l'autre canal présélectionné |
| QR à la place | `qr-code` | — | → M-20 (lien single-use au même montant) |

- **Modals/sheets**: MS-01.
- **States**: single state per failure_code; offline: actions blocked-write.
- **System**: Gestes: aucun · Offline: renders from cache/push · FLAG_SECURE: no · Back: → M-15 · TalkBack: live region "Paiement échoué : {raison}" · Deep-link: push `charge.failed` cold-start capable.
- **Events**: error haptic (single buzz), no sound.

### M-20 — QR de paiement / QR display
- **Route**: `qr-display` · **Icon**: lucide `qr-code` · **Access**: same as M-15 (static counter QR: all roles) · **Purpose**: fullscreen scannable QR for this amount (or static open-amount counter QR).
- **Layout zones**: amount huge on top (dynamic mode) or "Le client choisit le montant" (static) · QR centered max-size · caption "Le client scanne avec son appareil photo" · status line (dynamic mode: waits like M-17) · footer.
- **Data displayed**: dynamic: `payment_link.url` + `qr_png_url` from `POST /payment_links` response (rendered locally from url, not the PNG — sharper); static: merchant's permanent open link (`links_cache`); live payment status via the underlying link's charge events (push).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Annuler | `x` | — | dynamic: `POST /payment_links/{id}/deactivate` → M-16 ; static: ferme |
| Partager | `share-2` | — | MS-05 avec l'URL du lien |

- **Modals/sheets**: MS-05, MS-01.
- **States**: brightness auto-max while visible (restored on exit); dynamic success → routes to M-18; static: stays until closed; offline: static QR renders from cache; dynamic creation blocked-write.
- **System**: Gestes: aucun · Offline: static cached-read; dynamic blocked-write · FLAG_SECURE: no (QR is meant to be scanned/shown) · Back: = Annuler · TalkBack: "Code QR de paiement de {montant} francs CFA" · Deep-link: n/a.
- **Events**: `qr_shown`.

---

## LINKS

### M-21 — Liens de paiement / Links list
- **Route**: `links-list` · **Icon**: lucide `link` · **Access**: all (create: Owner/Admin/Finance) · **Purpose**: manage payment links.
- **Layout zones**: header (title, `search`) · cards list · FAB `plus` · queued-badge strip if `outbox_queue` non vide ("1 lien en attente de publication `cloud-off`").
- **Data displayed**: `GET /payment_links` → per card: title, amount or "montant libre", collected total, paid count ("Payé 12 fois"), active dot (success-600 / ink-500), created_at.
- **Inputs**: recherche (texte libre, filtre titre local).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer un lien | `plus` | Owner/Admin/Finance | ouvre M-22 (sheet) |
| Ouvrir un lien | — | — | → M-24 |
| Partager | `share-2` | — | swipe/long-press → MS-05 |
| Voir le QR | `qr-code` | — | swipe/long-press → M-25 |
| Désactiver | `trash-2` | Owner/Admin/Finance | swipe/long-press → MS-06 |

- **Modals/sheets**: M-22, MS-05, MS-06, MS-01.
- **States**: loading (3 skeleton cards); empty (illustration + "Créez votre premier lien en 30 secondes" + [Créer un lien]); error retry; offline cached-read + queued items shown greyed with `cloud-off`.
- **System**: Gestes: pull-to-refresh; swipe row reveals partager/QR/désactiver; long-press = same menu · Offline: cached-read; create is queued-write · FLAG_SECURE: no · Back: → M-12 · TalkBack: swipe actions duplicated in long-press menu with labels · Deep-link: `ijimpay://link/{id}` → M-24.
- **Events**: outbox flush toast surfaces here.

### M-22 — Créer un lien / Link create (bottom sheet, single screen)
- **Route**: `link-create` · **Icon**: lucide `link` · **Access**: Owner/Admin/Finance · **Purpose**: create a link in < 30 s.
- **Layout zones**: sheet grabber + title "Nouveau lien" · form · footer CTA.
- **Data displayed**: —

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Titre | texte | 2–60 caractères | vide | "Donnez un titre à votre lien (ex. : Gâteau d'anniversaire)." |
| Montant | segmenté Fixe / Libre + champ montant si Fixe | Fixe : entier ≥ 100 FCFA ; Libre : montant minimum optionnel ≥ 100 | Fixe | "Montant minimum : 100 FCFA." |
| Description (optionnel) | texte multiligne | ≤ 200 caractères | vide | — |
| Réutilisable | interrupteur | — | activé | — |
| Expire le (optionnel) | date (`calendar`) | > aujourd'hui | jamais | "La date d'expiration doit être future." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer | — | idem | `POST /payment_links` `{title, amount, amount_type, reusable, expires_at}` → M-23 ; hors ligne → mise en file (`outbox_queue`) + toast "Lien enregistré — il sera publié dès le retour du réseau." puis fermeture |

- **Modals/sheets**: MS-01 (server validation errors).
- **States**: loading CTA; field errors; offline queued-write (only non-money write in the app).
- **System**: Gestes: drag-down dismiss (confirm if dirty: "Abandonner ce lien ?") · Offline: queued-write · FLAG_SECURE: no · Back: dismiss (same confirm) · TalkBack: switch "Réutilisable, {activé|désactivé}" · Deep-link: n/a.
- **Events**: `link_created` (+ queued variant).

### M-23 — Lien créé / Link created (share)
- **Route**: `link-created` · **Icon**: lucide `circle-check` · **Access**: creator · **Purpose**: immediate share of the fresh link.
- **Layout zones**: success check + title · URL big monospace card · share buttons row · footer "Terminé".
- **Data displayed**: `payment_link.url`, `qr_png_url`, title, amount.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| WhatsApp | `message-circle` | — | intent WhatsApp, texte prérempli "Payez {titre} ici en toute sécurité 👉 {url}" |
| SMS | `message-square` | — | intent SMS, même texte |
| Copier | `copy` | — | presse-papiers + toast "Lien copié" |
| QR | `qr-code` | — | → M-25 |
| Terminé | — | — | → M-21 (liste rafraîchie) |

- **Modals/sheets**: MS-05 (generic share via "Plus" `share-2`).
- **States**: single; WhatsApp absent → bouton masqué.
- **System**: Gestes: aucun · Offline: si le lien vient de la file, écran affiché à la publication (toast → tap → ici) · FLAG_SECURE: no · Back: → M-21 · TalkBack: chaque bouton "Partager par {canal}" · Deep-link: n/a.
- **Events**: `link_shared(channel)`.

### M-24 — Détail du lien / Link detail
- **Route**: `link-detail` · **Icon**: lucide `link` · **Access**: all (edit/deactivate: Owner/Admin/Finance) · **Purpose**: stats and management of one link.
- **Layout zones**: header (title, active dot, `ellipsis-vertical` menu) · stats row (Collecté · Payé n fois · Conversion vues→payés %) · tx list for this link (TxRows) · footer actions.
- **Data displayed**: `GET /payment_links/{id}`: title, url, amount/amount_type, reusable, active, expires_at; stats from link analytics (conversion — API addition `GET /payment_links/{id}/stats`); charges filtered (`GET /charges?payment_link={id}` — filter is an API addition).
- **Inputs**: aucun (édition via M-22 pré-rempli).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Partager | `share-2` | — | MS-05 |
| Voir le QR | `qr-code` | — | → M-25 |
| Modifier | `pencil` | Owner/Admin/Finance | ouvre M-22 pré-rempli (PATCH — API addition `PATCH /payment_links/{id}`) |
| Désactiver | `trash-2` | Owner/Admin/Finance | MS-06 → `POST /payment_links/{id}/deactivate` |
| Copier l'URL | `copy` | — | presse-papiers |
| Ouvrir une transaction | — | — | → M-27 |

- **Modals/sheets**: M-22, MS-05, MS-06, MS-01.
- **States**: loading skeleton; deactivated (grey banner "Ce lien est désactivé" + [Réactiver] Owner/Admin — API addition reactivate); empty tx list "Aucun paiement sur ce lien pour l'instant. Partagez-le !"; offline cached-read.
- **System**: Gestes: pull-to-refresh · Offline: cached-read; edit/deactivate blocked-write · FLAG_SECURE: no · Back: → M-21 · TalkBack: `ellipsis-vertical` = "Plus d'actions sur le lien" · Deep-link: `ijimpay://link/{id}`.
- **Events**: —

### M-25 — QR du lien / Link QR
- **Route**: `link-qr` · **Icon**: lucide `qr-code` · **Access**: all · **Purpose**: fullscreen QR of a link + print/PDF export.
- **Layout zones**: title + amount (or "montant libre") · QR max-size · caption · footer.
- **Data displayed**: link url (rendered as QR locally), `qr_png_url` for export.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Imprimer / PDF A6 | `printer` | — | génère la carte comptoir A6 (logo, titre, QR, url courte) → share sheet système (impression/PDF) ; impression Bluetooth directe via M-51 (flag v1.2) |
| Partager | `share-2` | — | MS-05 |

- **Modals/sheets**: MS-05.
- **States**: brightness auto-max; offline: renders from cached url.
- **System**: Gestes: aucun · Offline: cached-read · FLAG_SECURE: no · Back: → M-24/M-23 · TalkBack: "Code QR du lien {titre}" · Deep-link: n/a.
- **Events**: `link_qr_exported`.

---

## ACT

### M-26 — Activité / Activity list
- **Route**: `activity-list` · **Icon**: lucide `arrow-left-right` · **Access**: all roles · **Purpose**: full transaction history with filters and day totals.
- **Layout zones**: header (title, `search`, `list-filter` badge-dotted when active) · segmented filter (Tout · Réussis · En attente · Échoués) · date chips (Aujourd'hui / 7 j / Mois / Personnalisé) · sticky day headers with day totals · TxRow list · overflow menu (export).
- **Data displayed**: `GET /charges` with filters `status`, `channel`, `created[gte|lte]`, `customer_phone` (search by ref/phone): each TxRow = StatusBadge, customer phone/name, description, ChannelChip, AmountText (+green in), time; day header total = sum succeeded of the day.
- **Inputs**: recherche (téléphone ou référence ; ≥ 3 caractères avant requête).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Filtres | `list-filter` | — | ouvre MS-19 |
| Ouvrir une transaction | — | — | → M-27 (sheet) |
| Exporter le mois (CSV) | `download` | Owner/Admin/Finance | menu `ellipsis-vertical` → `GET /exports/charges?month=` (API addition, envoi par e-mail) → toast "Export en cours — vous le recevrez par e-mail." |

- **Modals/sheets**: M-27, MS-19, MS-01.
- **States**: loading (8 skeleton rows); empty "Aucune transaction — créez un lien ou encaissez au comptoir." + [Encaisser]; filtered-empty "Aucun résultat pour ces filtres." + [Réinitialiser]; error retry; offline cached-read (200 dernières) with chip; pending rows live-update (poll 15 s + push).
- **System**: Gestes: pull-to-refresh; infinite scroll (cursor `starting_after`) · Offline: cached-read · FLAG_SECURE: no · Back: → M-12 · TalkBack: `list-filter` = "Filtres, {n} actifs" · Deep-link: hero tap from M-12 arrives pre-filtered Aujourd'hui.
- **Events**: —

### M-27 — Détail de la transaction / Tx detail (bottom sheet, full-height)
- **Route**: `tx-detail` · **Icon**: lucide `arrow-left-right` · **Access**: all (refund: Finance/Admin/Owner) · **Purpose**: everything about one charge + receipt/refund actions.
- **Layout zones**: header (amount XL + StatusBadge + ChannelChip) · timeline (Créée {heure} → Demande envoyée → {Réussie|Échouée|Expirée} {heure}, provider_ref, poll count secondary) · customer card (phone, name) · breakdown card (Brut / Frais / Net from `amount`, `fee`, `net`) · origin card (lien ou facture si `payment_link_id`/`invoice_id`) · refund card (si remboursée : montant, date) · actions list.
- **Data displayed**: `GET /charges/{id}`: all fields incl. `failure_code` (human FR mapping as M-19), `reference`, `created_at`, `succeeded_at`, `expires_at`.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Partager le reçu | `share-2` | — | → M-28 (visible si succeeded/refunded) |
| Rembourser | `undo-2` | Finance/Admin/Owner | ouvre MS-07 (succeeded uniquement, ≤ net non déjà remboursé) |
| Copier la référence | `copy` | — | presse-papiers + toast "Référence copiée" |
| Signaler un problème | `life-buoy` | — | ouvre MS-17 (contexte tx prérempli) |

- **Modals/sheets**: M-28, MS-07, MS-08 (refund confirm), MS-17, MS-01.
- **States**: loading skeleton; pending live-updates in place; failed shows reason card; refunded shows `refunded` badge + linked refund; permission (Viewer): refund hidden; offline cached-read (actions money-moving blocked).
- **System**: Gestes: drag-down dismiss; swipe between adjacent list items (left/right within M-26 context) · Offline: cached-read · FLAG_SECURE: no · Back: dismiss sheet · TalkBack: sheet opens announcing "Transaction de {montant}, statut {statut}" · Deep-link: `ijimpay://tx/{id}` + push `charge.*` land here (cold-start: fetch then render).
- **Events**: —

### M-28 — Reçu / Receipt preview & share
- **Route**: `receipt-preview` · **Icon**: lucide `share-2` · **Access**: all · **Purpose**: render the 80mm-style receipt and share/print it.
- **Layout zones**: receipt render (monochrome: logo, nom marchand, montant XXL, statut, réf, date, QR vers `pay.ijimpay.com/r/{id}`) · format toggle (Image / PDF) · footer actions.
- **Data displayed**: charge fields + receipt verify URL; PDF via `GET /receipts/{charge_id}.pdf` (API addition) with local image fallback rendered offline from cache.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| WhatsApp | `message-circle` | — | partage l'image + texte "Reçu Ijim Pay — {montant} le {date}. Vérifiez : {url}" |
| Partager | `share-2` | — | MS-05 (image ou PDF selon toggle) |
| Imprimer | `printer` | — | v1 : share sheet système ; v1.2 (flag `pos_printer`) : impression directe → M-51 si aucune imprimante appairée |

- **Modals/sheets**: MS-05, MS-01.
- **States**: rendering (skeleton receipt); offline: image générée localement depuis le cache ("PDF indisponible hors ligne").
- **System**: Gestes: pinch-zoom preview · Offline: cached-read (image local) · FLAG_SECURE: **yes** (receipt carries customer phone) · Back: → M-27/M-18 · TalkBack: preview "Reçu de {montant} francs CFA, {date}" · Deep-link: n/a.
- **Events**: `receipt_shared(format,channel)`.

---

## PAYOUT (v1.1)

### M-29 — Paiements sortants / Payouts home
- **Route**: `payouts-home` · **Icon**: lucide `send` · **Access**: Finance/Admin/Owner (hidden for Viewer/Developer; hidden entirely if role lacks permission, and on rooted devices) · **Purpose**: wallet + approvals + history + beneficiaries hub.
- **Layout zones**: wallet balance card ("Portefeuille de paiement" AmountText XL + [Approvisionner `plus-circle`] + [Payer `send`]) · tabs · list per tab.
- **Tabs**:
  - **Validations** (badge = count pending_approval): batch cards — total, count items, créé par, date, StatusBadge `pending_approval` (outlined warning) → M-30. Empty: "Aucun lot à valider. Tout est à jour."
  - **Historique**: payouts + batches merged timeline (`GET /payouts`, `GET /payout_batches`): amount, beneficiary/batch label, StatusBadge (processing/succeeded/failed/partially_failed as processing+warning banner) → M-30 (batch) or item detail row expand. Empty: "Aucun paiement sortant pour l'instant."
  - **Bénéficiaires**: → embedded M-32 list. Empty: "Ajoutez vos premiers bénéficiaires pour payer en un clic."
- **Data displayed**: wallet balance (`GET /balance` → payout wallet line); batches (`GET /payout_batches?status=pending_approval`): totals, created_by; history lists.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approvisionner | `plus-circle` | Finance/Admin/Owner | → M-35 |
| Payer | `send` | Finance/Admin/Owner | → M-34 |
| Ouvrir un lot | `layers` | approbateurs pour Validations | → M-30 |

- **Modals/sheets**: MS-01.
- **States**: loading skeletons per tab; empties above; insufficient wallet Banner "Solde du portefeuille insuffisant pour vos paiements en attente." + CTA; provider down: `triangle-alert` note sur le canal; offline cached-read, all writes blocked.
- **System**: Gestes: pull-to-refresh; tab swipe · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-12 · TalkBack: tab badges "Validations, {n} en attente" · Deep-link: push `payout_batch.*` → tabs as per routing table.
- **Events**: —

### M-30 — Validation du lot / Approval detail
- **Route**: `approval-detail` · **Icon**: lucide `check-check` · **Access**: Owner/Admin, **not the maker** (maker sees read-only with banner "Vous avez créé ce lot — un autre administrateur doit le valider.") · **Purpose**: review a batch before approving.
- **Layout zones**: summary header (total XL, nombre d'items, créé par, date, motif) · anomaly hints card (`triangle-alert` warning-600 : "2 nouveaux bénéficiaires jamais payés", "Montant 3× supérieur à votre lot habituel") · scrollable item list (beneficiary name+phone, ChannelChip, AmountText) · audit strip (créé par X le {date}) · footer (2 buttons, approve never default-focused).
- **Data displayed**: `GET /payout_batches/{id}`: status, totals, created_by, items[] (amount, channel, beneficiary.phone/name, reference, status); anomaly hints computed client-side vs `beneficiaries_cache` + batch history.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approuver | `check-check` | Owner/Admin (≠ maker) | → M-31 |
| Rejeter | `x` | Owner/Admin (≠ maker) | ouvre MS-09 (raison obligatoire) → `POST /payout_batches/{id}/reject` (API addition) |

- **Modals/sheets**: MS-09, MS-01.
- **States**: loading; already-decided ("Ce lot a déjà été {approuvé par {nom}|rejeté}." read-only); maker read-only; provider down for batch channel: approve disabled + "Orange Money est indisponible — la validation reprendra dès le rétablissement."; offline: read from cache, both actions blocked-write.
- **System**: Gestes: pull-to-refresh · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-29 · TalkBack: hints "Avertissement : {texte}" · Deep-link: push `payout_batch.pending_approval` + `ijimpay://batch/{id}`.
- **Events**: —

### M-31 — Confirmer l'approbation / Approval confirm
- **Route**: `approval-confirm` · **Icon**: lucide `check-check` · **Access**: Owner/Admin (≠ maker), **enrolled device only** (device registered in M-43; unenrolled → state below) · **Purpose**: final restated confirmation with biometric/PIN.
- **Layout zones**: restatement card ("Approuver **4 250 000 FCFA** vers **50 bénéficiaires** ?") · fee line · biometric/PIN zone · footer.
- **Data displayed**: batch totals, fees, wallet after ("Portefeuille après paiement : 950 000 FCFA").
- **Inputs**: PIN (pavé 4 chiffres, si biométrie indisponible) — validation locale puis jeton d'approbation.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer avec biométrie | `fingerprint` | idem écran | prompt → `POST /payout_batches/{id}/approve` (Idempotency-Key) → écran succès inline ("Lot approuvé — traitement en cours" + lien suivi → M-29 Historique) |
| Annuler | `x` | — | → M-30 |

- **Modals/sheets**: MS-01.
- **States**: success (check, no confetti, sober); `insufficient_balance` → "Portefeuille insuffisant ({manque} FCFA manquants)." + [Approvisionner] → M-35; unenrolled device: "Cet appareil n'est pas autorisé pour les validations. Ajoutez-le dans Menu → Sécurité → Appareils."; biometric fail ×3 → fallback PIN; offline blocked-write.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: **yes** · Back: → M-30 (aucune action partielle possible : l'approbation est atomique) · TalkBack: restatement read first, buttons after · Deep-link: n/a.
- **Events**: `batch_approved`; push `payout_batch.completed` follows asynchronously.

### M-32 — Bénéficiaires / Beneficiary list
- **Route**: `beneficiary-list` · **Icon**: lucide `users-round` · **Access**: Finance/Admin/Owner · **Purpose**: directory of payout beneficiaries.
- **Layout zones**: search bar · list (name + verified badge `badge-check` si nom vérifié opérateur, phone, ChannelChip) · payroll groups section (lecture seule v1.1 : "Groupes gérés sur le web") · FAB `plus`.
- **Data displayed**: `GET /beneficiaries` (API addition): name, phone, channel, verified_name flag; groups read-only.
- **Inputs**: recherche (nom/téléphone, filtre local).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ajouter | `plus` | Finance/Admin/Owner | ouvre MS-12 |
| Modifier | `pencil` | Finance/Admin/Owner | long-press → MS-12 pré-rempli (`PATCH /beneficiaries/{id}` — API addition) |
| Supprimer | `trash-2` | Admin/Owner | long-press → MS-13 |
| Payer ce bénéficiaire | `send` | Finance/Admin/Owner | → M-34 pré-rempli |

- **Modals/sheets**: MS-12, MS-13, MS-04 (contact import), MS-01.
- **States**: loading; empty "Aucun bénéficiaire — ajoutez le premier."; offline cached-read (CRUD blocked-write); error retry.
- **System**: Gestes: pull-to-refresh; long-press menu · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-29 · TalkBack: `badge-check` = "Nom vérifié par l'opérateur" · Deep-link: n/a.
- **Events**: —

*(M-33 intentionally unassigned: docs/10's `beneficiary-add` is a sheet, specified as MS-12.)*

### M-34 — Paiement individuel / Single payout
- **Route**: `payout-single` · **Icon**: lucide `send` · **Access**: Finance/Admin/Owner · **Purpose**: pay one beneficiary (maker step; approval per threshold).
- **Layout zones**: beneficiary picker (recherche + récents + [Nouveau]) · amount + motif form · funding line ("Portefeuille : 1 200 000 FCFA — suffisant" success / "insuffisant" danger + CTA) · footer CTA.
- **Data displayed**: `beneficiaries_cache`; wallet balance (`GET /balance`); channel auto from beneficiary phone prefix.

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Bénéficiaire | sélecteur (liste + recherche) ou MS-12 pour nouveau | requis | vide | "Choisissez un bénéficiaire." |
| Montant | numérique | entier ≥ 100 FCFA ; ≤ solde portefeuille ; ≤ limite palier | vide | "Montant supérieur au solde du portefeuille." / "Montant au-dessus de votre limite quotidienne." |
| Motif | texte | 2–80 caractères | vide | "Indiquez le motif du paiement (ex. : Salaire juillet)." |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Continuer | — | idem | écran de confirmation inline (restate montant + bénéficiaire + frais) → MS-08 (biométrie/PIN) → `POST /payouts` (Idempotency-Key) → succès ou `pending_approval` ("Envoyé pour validation à un administrateur.") |
| Approvisionner | `plus-circle` | idem | visible si insuffisant → M-35 |

- **Modals/sheets**: MS-12, MS-08, MS-01.
- **States**: insufficient balance (CTA swap); provider down (channel row disabled + `triangle-alert`); above-threshold → pending_approval info state; success state (sober check + réf); offline blocked-write; Viewer/Developer: screen unreachable.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: **yes** · Back: confirm si formulaire rempli ("Abandonner ce paiement ?") · TalkBack: funding line announced on change · Deep-link: n/a.
- **Events**: `payout_created`.

### M-35 — Approvisionner / Top-up instructions
- **Route**: `topup-instructions` · **Icon**: lucide `plus-circle` · **Access**: Finance/Admin/Owner · **Purpose**: fund the payout wallet with auto-matched deposit.
- **Layout zones**: amount intent form (step 1) · instruction card (step 2 : code référence XL + étapes USSD exactes par canal servies par la config) · live status footer ("En attente de votre dépôt…" pulsing `clock`).
- **Data displayed**: `POST /topups` → reference code, instructions per channel; status auto-updates via push `balance.topup.received`.

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Montant à déposer | numérique | entier ≥ 1 000 FCFA | vide | "Montant minimum d'approvisionnement : 1 000 FCFA." |
| Canal | segmenté MTN / Orange | requis ; canal down désactivé | canal de règlement par défaut | — |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Obtenir les instructions | — | idem | `POST /topups` → step 2 |
| Copier la référence | `copy` | — | presse-papiers + toast "Référence copiée" |
| Terminé | — | — | → M-29 (le matching continue en arrière-plan) |

- **Modals/sheets**: MS-01.
- **States**: step 1 form; step 2 waiting (pulsing); matched → success in place (check scale-in, "Approvisionnement reçu : 500 000 FCFA", pas de confetti) → wallet card updated; timeout copy à 30 min "Dépôt non détecté — vérifiez la référence saisie ou contactez le support."; offline blocked-write.
- **System**: Gestes: aucun · Offline: blocked-write · FLAG_SECURE: **yes** · Back: step 2 → M-29 (instructions restent visibles dans Historique) · TalkBack: reference "Référence de dépôt : {code}, bouton copier" · Deep-link: push `balance.topup.received` lands here if open, else M-37.
- **Events**: `topup_initiated` / matched push.

---

## MENU

### M-36 — Menu
- **Route**: `menu` · **Icon**: lucide `settings` · **Access**: all (rows filtered by role) · **Purpose**: hub for balance, settlements, team, settings.
- **Layout zones**: profile header (logo, nom marchand, palier badge, mode pill) · sections list : Finances (Solde `wallet`, Règlements `landmark`) · Entreprise (Profil, Vérification `badge-check`, Équipe `users-round`) · Application (Sécurité `lock`, Notifications `bell`, Langue `globe`, Imprimante `printer` [flag v1.2]) · Aide (Aide & support `life-buoy`, À propos) · footer (Se déconnecter `log-out`, version).
- **Data displayed**: merchant name/logo/tier (`merchant_cache`); role-filtered rows: Équipe (Owner/Admin), Vérification (Owner/Admin), Solde/Règlements (tous sauf montants masqués si `prefs.mask`).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir une section | `chevron-right` | selon ligne | → M-37…M-47, M-51 |
| Se déconnecter | `log-out` | — | ouvre MS-14 |

- **Modals/sheets**: MS-14.
- **States**: static from cache; offline fully functional (reads).
- **System**: Gestes: aucun · Offline: cached-read · FLAG_SECURE: no · Back: → M-12 · TalkBack: rows self-labeled · Deep-link: n/a.
- **Events**: —

### M-37 — Solde / Balance detail
- **Route**: `balance-detail` · **Icon**: lucide `wallet` · **Access**: all roles (read) · **Purpose**: the three balances + ledger.
- **Layout zones**: three AmountText tiles (Disponible · En attente de règlement · Portefeuille de paiement) · masquer toggle `eye`/`eye-off` · ledger list (icône par type : encaissement `hand-coins`, frais, paiement `send`, règlement `landmark`, ajustement, approvisionnement `plus-circle`; libellé, date, AmountText signé).
- **Data displayed**: `GET /balance` (available, pending, payout wallet); `GET /balance_transactions` (type, amount, created_at, reference object).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approvisionner | `plus-circle` | Finance/Admin/Owner | → M-35 |
| Masquer les montants | `eye-off` | — | toggle `prefs.mask` (partout dans l'app) |
| Ouvrir une ligne | — | — | ligne charge → M-27 ; règlement → M-39 |

- **Modals/sheets**: MS-01.
- **States**: loading (3 tile skeletons + rows); empty ledger "Aucun mouvement pour l'instant."; offline cached-read + timestamp chip; error retry.
- **System**: Gestes: pull-to-refresh; infinite scroll · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-36 · TalkBack: `eye-off` = "Masquer les montants" · Deep-link: push `balance.topup.received` fallback target.
- **Events**: —

### M-38 — Règlements / Settlements list
- **Route**: `settlements-list` · **Icon**: lucide `landmark` · **Access**: all roles · **Purpose**: T+1 transfers to the merchant's own account.
- **Layout zones**: header · list (période, AmountText, destination masquée "MoMo •• 00 00", StatusBadge paid / processing "en transit").
- **Data displayed**: `GET /settlements`: period, amount, destination, status, paid_at.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir un règlement | — | — | → M-39 |

- **Modals/sheets**: MS-01.
- **States**: loading; empty "Votre premier règlement arrivera après votre premier encaissement en mode réel."; offline cached-read.
- **System**: Gestes: pull-to-refresh; infinite scroll · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-36 · TalkBack: rows "Règlement du {période}, {montant}, {statut}" · Deep-link: n/a.
- **Events**: —

### M-39 — Détail du règlement / Settlement detail
- **Route**: `settlement-detail` · **Icon**: lucide `landmark` · **Access**: all roles · **Purpose**: one settlement + its included transactions + relevé.
- **Layout zones**: header (amount XL, StatusBadge, période, destination) · included transactions list (TxRows) · footer.
- **Data displayed**: `GET /settlements/{id}` with included transactions.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Partager le relevé (PDF) | `share-2` | Owner/Admin/Finance | télécharge le relevé (`GET /settlements/{id}` export — API addition `GET /settlements/{id}/statement.pdf`) → MS-05 |
| Ouvrir une transaction | — | — | → M-27 |

- **Modals/sheets**: MS-05, MS-01.
- **States**: loading; in-transit ("En transit vers votre compte — généralement sous 24 h."); offline cached-read, PDF requires network ("Relevé indisponible hors ligne").
- **System**: Gestes: pull-to-refresh · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-38 · TalkBack: standard · Deep-link: push `settlement.paid` + `ijimpay://settlement/{id}`.
- **Events**: —

### M-40 — Équipe / Team list
- **Route**: `team-list` · **Icon**: lucide `users-round` · **Access**: Owner/Admin (others: row hidden in Menu) · **Purpose**: manage members and roles.
- **Layout zones**: header + [Inviter `plus`] · members list (avatar initiales, nom, téléphone, rôle `shield` chip, 2FA dot, dernière activité).
- **Data displayed**: `GET /members` (API addition): name, phone, role, totp_enrolled, last_active_at; pending invites section.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Inviter un membre | `plus` | Owner/Admin | ouvre MS-10 |
| Changer le rôle | `shield` | Owner/Admin | long-press → sheet rôle (mêmes descriptions que MS-10) → `PATCH /members/{id}` (API addition) |
| Retirer | `trash-2` | Owner/Admin | long-press → MS-11 ; impossible sur le dernier Owner ("Impossible de retirer le dernier propriétaire.") |
| Renvoyer l'invitation | `rotate-cw` | Owner/Admin | sur invite en attente → `POST /invitations/{id}/resend` (API addition) |

- **Modals/sheets**: MS-10, MS-11, MS-04, MS-01.
- **States**: loading; empty (solo Owner): "Vous travaillez seul pour l'instant. Invitez votre équipe."; permission-denied n/a (hidden); offline cached-read, writes blocked.
- **System**: Gestes: pull-to-refresh; long-press menu · Offline: cached-read · FLAG_SECURE: no · Back: → M-36 · TalkBack: rows "{nom}, rôle {rôle}, double authentification {activée|non}" · Deep-link: push `team.member_changed`.
- **Events**: —

### M-41 — Profil de l'entreprise / Business profile
- **Route**: `business-profile` · **Icon**: lucide `badge-check` · **Access**: view all; edit Owner/Admin · **Purpose**: name + logo (shown on checkout & receipts).
- **Layout zones**: logo uploader (rond, hint "Votre logo apparaît sur la page de paiement") · form · footer CTA.
- **Data displayed**: `GET /merchant`: name, legal_name, sector, city, address, logo_url.

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Logo | image (galerie/caméra, crop carré) | jpg/png ≤ 2 Mo, min 256×256 | logo actuel | "Image trop lourde (max 2 Mo)." |
| Nom commercial | texte | 2–80 caractères | valeur actuelle | "Entrez le nom de votre entreprise." |
| Ville / Adresse | texte | comme M-08 | valeurs actuelles | idem M-08 |

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer | — | Owner/Admin | `PATCH /merchant` + `POST /merchant/logo` (API additions) → toast "Profil mis à jour" |

- **Modals/sheets**: MS-03 (si prise photo logo), MS-01.
- **States**: loading; saving; permission-denied (Viewer/Finance/Developer: lecture seule, bouton masqué); offline: read cached, save blocked.
- **System**: Gestes: aucun · Offline: cached-read · FLAG_SECURE: no · Back: confirm si modifié · TalkBack: logo = "Logo de l'entreprise, toucher pour changer" · Deep-link: n/a.
- **Events**: —

### M-42 — Documents de vérification / Verification docs
- **Route**: `verification-docs` · **Icon**: lucide `badge-check` · **Access**: Owner/Admin · **Purpose**: post-onboarding doc status + re-upload rejected + tier upgrade.
- **Layout zones**: tier card (palier actuel + limites, CTA "Passer au palier 2") · docs list (nom, statut chip : En examen `clock` / Accepté `circle-check` / Rejeté `circle-x` + motif FR) · footer.
- **Data displayed**: `GET /kyb_documents` (statuses + rejection reasons); tier from `GET /merchant`.
- **Inputs**: aucun (recapture via M-09).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Reprendre ce document | `camera` | Owner/Admin | → M-09 ciblé (boucle capture/reprise) |
| Demander le palier 2 | — | Owner | ajoute les docs requis manquants à M-09 |

- **Modals/sheets**: MS-01.
- **States**: loading; all-approved (success surface); rejected (danger chips + reasons); offline cached-read.
- **System**: Gestes: pull-to-refresh · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-36 · TalkBack: chips read status + reason · Deep-link: `ijimpay://verification` (post-onboarding target).
- **Events**: —

### M-43 — Sécurité / Security
- **Route**: `security` · **Icon**: lucide `lock` · **Access**: all (own account) · **Purpose**: PIN, biometric, app-lock, devices, sessions.
- **Layout zones**: sections list — PIN (Changer le PIN) · Biométrie (toggle) · Verrouillage auto (30 s / 2 min / 5 min) · Appareils (list : modèle `smartphone`, "Cet appareil" tag, autorisé validations oui/non, dernier accès) · Sessions actives (appareil, ville approx., date) .
- **Data displayed**: `GET /devices`, `GET /sessions` (API additions); local prefs (biometric, lock timeout).
- **Inputs**: changement de PIN = flux M-05 réutilisé précédé de la saisie de l'ancien PIN ("Ancien PIN incorrect." après erreur).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Changer le PIN | `lock` | — | ancien PIN → M-05 (mode changement) → `POST /auth/pin/set` |
| Biométrie | `fingerprint` | — | toggle ; activation exige prompt système réussi |
| Verrouillage auto | `clock` | — | sélecteur → `prefs.lock_timeout` |
| Autoriser cet appareil pour les validations | `check-check` | Owner/Admin | `POST /devices/{id}/enroll` (API addition) avec OTP de confirmation |
| Révoquer un appareil | `trash-2` | — | MS-15 → `DELETE /devices/{id}` |
| Déconnecter une session | `log-out` | — | confirm inline → `DELETE /sessions/{id}` |

- **Modals/sheets**: MS-15, MS-01.
- **States**: loading; single-device (list of one); revoked-current-device → force logout to M-02; offline: reads cached, all writes blocked; push `security.new_device` deep-links here.
- **System**: Gestes: aucun · Offline: cached-read · FLAG_SECURE: **yes** · Back: → M-36 · TalkBack: device rows "{modèle}, dernier accès {date}, {cet appareil}" · Deep-link: push `security.new_device`.
- **Events**: security email/SMS mirrors per docs/08 §C.

### M-44 — Préférences de notification / Notification prefs
- **Route**: `notifications-prefs` · **Icon**: lucide `bell` · **Access**: all (per-user) · **Purpose**: per-event toggles + sound, mirroring docs/08 §C matrix (push column).
- **Layout zones**: master rows (Son d'encaissement toggle · Mémoriser les numéros des clients toggle) · event matrix list (Paiement reçu, Paiement échoué, Lot à valider [approver roles], Lot terminé, Règlement envoyé, Approvisionnement reçu, Vérification (KYB), Sécurité — each: push toggle; e-mail/SMS mentionnés "gérés sur le web").
- **Data displayed**: `GET /notification_preferences` (API addition); `prefs.sound`, `prefs.remember_payers`.
- **Inputs**: toggles uniquement (aucune validation).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Basculer une préférence | — | — | `PUT /notification_preferences` optimiste ; rollback + toast "Impossible d'enregistrer — réessayez." en cas d'échec |
| Son d'encaissement | — | — | local `prefs.sound` ; aperçu du son au passage à activé |

- **Modals/sheets**: MS-02 (si permission système révoquée : banner "Les notifications sont désactivées pour Ijim Pay sur cet appareil." + [Ouvrir les réglages]), MS-01.
- **States**: loading; system-permission-off banner; offline: local toggles OK, server sync queued (non-money queued-write autorisé), badge "synchronisation en attente".
- **System**: Gestes: aucun · Offline: queued-write (prefs) · FLAG_SECURE: no · Back: → M-36 · TalkBack: toggles "{événement}, notification {activée|désactivée}" · Deep-link: n/a.
- **Events**: —

### M-45 — Langue / Language
- **Route**: `language` · **Icon**: lucide `globe` · **Access**: all · **Purpose**: FR/EN switch.
- **Layout zones**: radio list (Français — par défaut · English).
- **Data displayed**: `prefs.locale`; synced to `users.locale` (`PATCH /me` — API addition).
- **Inputs**: radio (aucune validation).

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Choisir la langue | `globe` | — | applique immédiatement (ICU), persiste local + serveur |

- **Modals/sheets**: —
- **States**: instant; offline: local apply, sync later.
- **System**: Gestes: aucun · Offline: queued-write (pref) · FLAG_SECURE: no · Back: → M-36 · TalkBack: radios "Français, sélectionné" · Deep-link: n/a.
- **Events**: —

### M-46 — Aide & support / Help
- **Route**: `help` · **Icon**: lucide `life-buoy` · **Access**: all · **Purpose**: WhatsApp support, FAQ, tutorials.
- **Layout zones**: contact card (WhatsApp CTA) · FAQ accordion (10 entrées FR embarquées + à jour en ligne) · tutoriels vidéo (vignettes `play-circle`).
- **Data displayed**: FAQ/tutorial manifest (`GET /app/config`), embedded fallback.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Écrire sur WhatsApp | `message-circle` | — | deep-link wa.me du support, message prérempli "Bonjour, je suis {marchand} (id {merchant_id})…" |
| Signaler un problème | `life-buoy` | — | ouvre MS-17 (sans contexte tx) |
| Voir un tutoriel | `play-circle` | — | ouvre la vidéo (navigateur/YouTube) |

- **Modals/sheets**: MS-17.
- **States**: offline: FAQ embarquée disponible, vidéos non ("Vidéos indisponibles hors ligne").
- **System**: Gestes: aucun · Offline: cached-read (FAQ) · FLAG_SECURE: no · Back: → M-36 · TalkBack: standard · Deep-link: n/a.
- **Events**: `support_contacted`.

### M-47 — À propos / About
- **Route**: `about` · **Icon**: lucide `book-open` · **Access**: all · **Purpose**: version, legal.
- **Layout zones**: logo + version + build · liens (Conditions d'utilisation · Confidentialité · Licences open source).
- **Data displayed**: version locale; liens web.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir un lien légal | `book-open` | — | navigateur externe |
| Licences | — | — | écran système Flutter LicensePage |

- **Modals/sheets**: —
- **States**: static.
- **System**: Gestes: 7 taps sur la version → écran diagnostic (device id, dernier sync, file d'attente) · Offline: static · FLAG_SECURE: no · Back: → M-36 · TalkBack: standard · Deep-link: n/a.
- **Events**: —

---

## GLOBAL & implied screens

### M-48 — Mise à jour requise / Force update
- **Route**: `force-update` · **Icon**: lucide `triangle-alert` · **Access**: all (blocking) · **Purpose**: block app below minimum version.
- **Layout zones**: illustration · copy "Une mise à jour est nécessaire pour continuer à encaisser en toute sécurité." · CTA.
- **Data displayed**: `GET /app/config` → min_version, store_url (cached: last known config also enforces).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Mettre à jour | `download` | — | ouvre Play Store (store_url) |

- **Modals/sheets**: —
- **States**: single, non-dismissible.
- **System**: Gestes: aucun · Offline: enforced from cached config · FLAG_SECURE: no · Back: exits app · TalkBack: full copy read · Deep-link: swallows all pending routes.
- **Events**: —

### M-49 — Maintenance
- **Route**: `maintenance` · **Icon**: lucide `triangle-alert` · **Access**: all (blocking) · **Purpose**: planned downtime notice.
- **Layout zones**: illustration · copy "Ijim Pay est en maintenance. Retour prévu à {heure}. Vos fonds sont en sécurité." · retry.
- **Data displayed**: `GET /app/config` → maintenance window.
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer | `rotate-cw` | — | re-fetch config ; auto-retry toutes les 60 s |

- **Modals/sheets**: —
- **States**: single; auto-dismiss when window ends.
- **System**: Gestes: aucun · Offline: distinct from offline banner (network up, API down) · FLAG_SECURE: no · Back: exits app · TalkBack: copy read · Deep-link: held, replayed after.
- **Events**: —

### M-50 — Verrouillage / App lock
- **Route**: `app-lock` · **Icon**: lucide `lock` · **Access**: authenticated, on resume after timeout · **Purpose**: PIN/biometric gate on foreground resume (timeout per M-43).
- **Layout zones**: blurred scrim over last screen · lock card (logo, 4 dots, pad, `fingerprint`).
- **Data displayed**: none (verification local; token refresh silencieux derrière).
- **Inputs**: PIN (pavé 4 chiffres — "PIN incorrect. Il vous reste {n} essais.").

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Biométrie | `fingerprint` | — | prompt auto à l'ouverture si activée ; succès → dismiss, retour exactement où on était |
| PIN oublié ? | — | — | → M-03 (ré-auth OTP) après 1 échec |

- **Modals/sheets**: —
- **States**: 5 échecs → OTP re-auth (M-03); recents-screen thumbnail always masked (FLAG_SECURE screens + lock scrim).
- **System**: Gestes: aucun · Offline: unlock local, reads continue cached · FLAG_SECURE: **yes** · Back: exits app (never bypass) · TalkBack: "Application verrouillée. Entrez votre PIN." · Deep-link: gate — target preserved.
- **Events**: `app_unlocked`.

### M-51 — Imprimante / Printer pairing (feature flag `pos_printer`, v1.2)
- **Route**: `printer-pairing` · **Icon**: lucide `printer` · **Access**: Owner/Admin/Finance · **Purpose**: pair a Bluetooth thermal printer (58/80 mm) for direct receipt printing.
- **Layout zones**: status card (imprimante appairée : nom, `bluetooth`, [Imprimer un test]) · scan list (appareils détectés) · help footer ("Allumez l'imprimante et activez le Bluetooth.").
- **Data displayed**: paired printer (`prefs.printer_mac`, name); BLE scan results (local).
- **Inputs**: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Rechercher | `rotate-cw` | idem | scan BLE (permission Bluetooth système Android 12+) |
| Appairer | `bluetooth` | idem | pairing → sauvegarde `prefs` → test print proposé |
| Imprimer un test | `printer` | idem | ticket test 80 mm |
| Oublier l'imprimante | `trash-2` | idem | confirm inline → efface `prefs.printer_mac` |

- **Modals/sheets**: system Bluetooth permission dialog; MS-01.
- **States**: no-bluetooth ("Activez le Bluetooth pour rechercher une imprimante." + [Activer]); scanning (spinner list); permission-denied (state + [Ouvrir les réglages]); paired; print-failed toast "Impression échouée — vérifiez le papier et la batterie."
- **System**: Gestes: pull-to-refresh = rescan · Offline: fully local · FLAG_SECURE: no · Back: → M-36 · TalkBack: rows "Imprimante {nom}, appuyer pour appairer" · Deep-link: n/a; hidden entirely when flag off.
- **Events**: `printer_paired`.

---

## Modals & Drawers/Sheets

### MS-01 — Feuille d'erreur / Error sheet (global)
- Single component for every API/network error. Zones: icône (par code : `cloud-off` réseau, `triangle-alert` serveur, `lock` permission) · phrase FR simple (mapping des codes stables docs/02 §1 ; défaut "Une erreur est survenue — réessayez.") · `request_id` en petit (copiable au tap) · actions.
- Inputs: aucun.

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer | `rotate-cw` | — | rejoue l'appel d'origine (même Idempotency-Key pour les POST argent) |
| Contacter le support | `life-buoy` | — | → M-46 (request_id prérempli) |

- Règles : jamais de code brut à l'écran ; `authentication_failed` → force M-07 ; `permission_denied` → copy "Votre rôle ne permet pas cette action." sans bouton Réessayer.

### MS-02 — Autorisation notifications / Notification permission (Android 13+)
- Pre-prompt sheet before the system dialog, shown once on first M-12: illustration `bell` + "Soyez averti à la seconde où un client paie — c'est votre caisse qui sonne." · [Activer les notifications] → system dialog · [Plus tard]. Denied twice → M-44 shows settings banner instead. No inputs. Confirm rule: never re-prompt in the charge flow itself.

### MS-03 — Autorisation caméra / Camera permission
- Pre-prompt before first camera use (M-09, M-41 logo): "L'appareil photo sert uniquement à photographier vos documents." · [Continuer] → system dialog · [Annuler]. Permanently denied → pane state with [Ouvrir les réglages] (App-Settings intent). No inputs.

### MS-04 — Autorisation contacts / Contacts permission + picker
- Pre-prompt before contact picker (M-16 payer, MS-10 invite, MS-12 beneficiary): "Choisissez un contact au lieu de taper le numéro. Vos contacts ne sont jamais envoyés à nos serveurs." · [Continuer] / [Saisir manuellement]. Granted → system contact picker; picked number normalized E.164, invalid → "Ce contact n'a pas de numéro camerounais valide."

### MS-05 — Partager / Share sheet
- Brand row (WhatsApp `message-circle` · SMS `message-square` · Copier `copy`) + bouton "Plus" `share-2` → Android share sheet. Payload variants: lien (texte prérempli FR), reçu (image/PDF + texte), relevé (PDF). No inputs; WhatsApp hidden if not installed.

### MS-06 — Désactiver le lien / Deactivate link confirm
- ConfirmSheet: "Désactiver « {titre} » ? Les clients ne pourront plus payer via ce lien." · [Désactiver] destructive (danger-600, non focus par défaut) → `POST /payment_links/{id}/deactivate` · [Annuler]. Erreur → MS-01.

### MS-07 — Rembourser / Refund sheet (Finance/Admin/Owner)
- Zones: restate charge (montant net remboursable) · form · footer.

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Montant à rembourser | numérique | entier ≥ 100 ; ≤ net non remboursé | montant total | "Le remboursement dépasse le montant disponible ({max} FCFA)." |
| Motif | sélecteur (Erreur de montant / Client insatisfait / Doublon / Autre) + texte si Autre | requis | vide | "Choisissez un motif." |

- [Continuer] → MS-08 (restate "Rembourser 12 500 FCFA au 6 70 00 00 00 ?") → `POST /charges/{id}/refund` (Idempotency-Key). Succès: toast "Remboursement envoyé" + badge `refunded` sur M-27. Offline: blocked-write.

### MS-08 — Confirmation PIN/biométrie / Money confirm
- Used by refunds, single payouts, batch approval fallback. Restates action + amount + counterparty in one sentence (bold amount); biometric prompt auto; fallback PIN pad after 3 fails; [Annuler]. Never default-focused confirm; 30 s validity then re-prompt. FLAG_SECURE yes.

### MS-09 — Rejeter le lot / Reject batch (Owner/Admin ≠ maker)
| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Raison du rejet | texte multiligne | 5–200 caractères | vide | "Expliquez la raison du rejet (visible par le créateur du lot)." |

- [Rejeter] destructive → `POST /payout_batches/{id}/reject` (API addition) → retour M-29 avec toast "Lot rejeté" ; notification au maker.

### MS-10 — Inviter un membre / Invite member (Owner/Admin)
- Zones: contact source (picker `book-user` via MS-04 ou saisie) · rôle radio avec descriptions en clair : Admin — "Tout gérer, sauf supprimer l'entreprise" · Développeur — "Clés API et webhooks" · Finance — "Encaisser, payer, rembourser" · Lecture — "Consulter uniquement" (Owner non attribuable).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Téléphone ou e-mail | tel/email | numéro CM valide ou e-mail RFC | vide | "Entrez un numéro ou un e-mail valide." |
| Rôle | radio (4) | requis | Lecture | "Choisissez un rôle." |

- [Envoyer l'invitation] → `POST /invitations` (API addition) → toast "Invitation envoyée à {destinataire}". Déjà membre → "Cette personne fait déjà partie de l'équipe."

### MS-11 — Retirer le membre / Remove member confirm
- "Retirer {nom} de {marchand} ? Son accès sera coupé immédiatement." [Retirer] destructive → `DELETE /members/{id}` (API addition) · [Annuler]. Dernier Owner → action indisponible (règle affichée).

### MS-12 — Ajouter un bénéficiaire / Add-edit beneficiary (Finance/Admin/Owner)
| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Téléphone | tel `+237` | 9 chiffres, préfixe MTN/Orange (canal auto affiché en ChannelChip) | vide | "Entrez un numéro Mobile Money valide." |
| Nom complet | texte | 2–80 caractères | vide | "Entrez le nom du bénéficiaire." |

- Contact picker via MS-04. [Enregistrer] → `POST /beneficiaries` (API addition) ; si vérification de nom opérateur disponible et divergente → hint "Nom chez l'opérateur : {nom} — vérifiez avant de payer." Duplicate phone → "Ce numéro est déjà dans vos bénéficiaires."

### MS-13 — Supprimer le bénéficiaire / Delete beneficiary confirm (Admin/Owner)
- "Supprimer {nom} ({numéro}) ? Les paiements passés restent dans l'historique." [Supprimer] destructive → `DELETE /beneficiaries/{id}` · [Annuler].

### MS-14 — Se déconnecter / Sign-out confirm
- "Se déconnecter de {marchand} ? Vous devrez saisir votre PIN pour revenir." [Se déconnecter] `log-out` → `POST /auth/logout` (API addition, best-effort), purge tokens (cache Drift conservé chiffré) → M-02 · [Annuler].

### MS-15 — Révoquer l'appareil / Revoke device confirm
- "Révoquer {modèle} ? Il sera déconnecté et ne pourra plus valider de paiements." [Révoquer] destructive → `DELETE /devices/{id}` · [Annuler]. Si appareil courant : copy ajoute "Vous serez déconnecté immédiatement."

### MS-16 — Changer de compte / Switch account sheet
- Liste des comptes connus sur l'appareil (avatar, marchand, numéro masqué) + "Ajouter un compte" → M-03. Tap → M-07 pour ce compte. Long-press → "Oublier ce compte sur cet appareil" (confirm inline, purge locale).

### MS-17 — Signaler un problème / Report a problem
| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Description | texte multiligne | 10–500 caractères | vide | "Décrivez le problème en quelques mots (10 caractères min)." |
| Joindre les journaux | interrupteur | — | activé | — |

- Contexte auto : charge_id/request_id si ouvert depuis M-27/MS-01. [Envoyer] → `POST /support/tickets` (API addition) → toast "Merci — notre équipe vous répond sous 24 h (souvent bien plus vite)." Offline: queued-write.

### MS-18 — Reprise de capture / Retake dialog (doc capture)
- Shown when local blur/glare check fails: aperçu de la photo + raison ("Photo floue — rapprochez-vous et tenez le téléphone stable." / "Reflets détectés — évitez la lumière directe.") · [Reprendre `rotate-cw`] (défaut) · [Utiliser quand même] (secondaire, autorisé max 1 fois par doc — le serveur peut re-rejeter).

### MS-19 — Filtres / Activity filters sheet
| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Statut | chips multi (8 StatusBadge) | — | tous | — |
| Canal | chips (MTN / Orange) | — | tous | — |
| Période | chips (Aujourd'hui / 7 j / Mois) + plage personnalisée (2 dates) | fin ≥ début | Aujourd'hui | "La date de fin doit suivre la date de début." |
| Montant min / max | numérique ×2 | max ≥ min | vides | "Le maximum doit dépasser le minimum." |

- [Appliquer] (compte les résultats en live "Voir 42 résultats") · [Réinitialiser]. Filtres persistés par session.

---

## API additions needed (method + path — referenced above, absent from docs/02)

- `POST /auth/otp/request` · `POST /auth/otp/verify` · `POST /auth/pin/set` · `POST /auth/token/refresh` · `POST /auth/logout` · `PATCH /me`
- `GET /merchant` · `PATCH /merchant` · `POST /merchant/logo`
- `GET /kyb_documents` · `POST /kyb_documents`
- `GET /members` · `PATCH /members/{id}` · `DELETE /members/{id}` · `POST /invitations` · `POST /invitations/{id}/resend`
- `GET /devices` · `POST /devices` (FCM token) · `POST /devices/{id}/enroll` · `DELETE /devices/{id}` · `GET /sessions` · `DELETE /sessions/{id}`
- `GET /notifications` · `POST /notifications/mark_all_read` · `GET /notification_preferences` · `PUT /notification_preferences`
- `GET /app/config` (min version, maintenance, USSD codes, FAQ manifest, feature flags)
- `POST /charges/{id}/cancel` · `GET /charges?payment_link={id}` (filter) · `GET /exports/charges?month=`
- `PATCH /payment_links/{id}` (edit + reactivate) · `GET /payment_links/{id}/stats`
- `POST /payout_batches/{id}/reject`
- `GET /beneficiaries` · `POST /beneficiaries` · `PATCH /beneficiaries/{id}` · `DELETE /beneficiaries/{id}`
- `GET /receipts/{charge_id}.pdf` · `GET /settlements/{id}/statement.pdf`
- `GET /channels/{channel}/history`
- `POST /support/tickets`
- Push event types (server-side, on top of docs/02 events): `kyb.decision`, `security.new_device`, `team.member_changed`.

## Icon additions needed (not in docs/07 §4 map)

`camera` · `image` · `flashlight` · `delete` (numpad backspace) · `x` (close) · `chevron-right` · `eye` / `eye-off` (mask amounts) · `book-user` (contact picker) · `smartphone` (devices — used in docs/08 but absent from the §4 map) · `rotate-cw` (retry/redeliver — same) · `bluetooth` (printer pairing) · `calendar` (date fields) · `plus` (FAB) · `play-circle` (video tutorials).

---

INVENTORY: pages=50 tabs=3 modals=19 forms=18 tables=69 actions=133
