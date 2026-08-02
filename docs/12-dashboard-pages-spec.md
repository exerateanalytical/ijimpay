# Ijim Pay — Merchant Dashboard Pages Spec (app.ijimpay.com)

Status: Draft v1 · Supersedes `docs/08-web-screens.md` §A (strictly more detailed; routes/roles/behaviors consistent with it). Design tokens, icons, components per `docs/07-brand-design-system.md`. API references per `docs/02-api-spec.md`; anything not in 02 is listed in **API additions needed** at the end — nothing is invented silently inline.

Conventions used in this document:
- All UI copy is French-first; EN parity exists via the `globe` switcher but only FR strings are specified here (EN is a translation task, not a design decision).
- Amounts always rendered `12 500 FCFA` (thin non-breaking space thousands, currency after).
- Roles: **Owner, Admin, Developer, Finance, Viewer**. "Tous" = all five. Viewer sees read-only pages with action buttons **hidden** (not disabled). Exception (per F-013 docs/16): l'export (lecture seule) est permis au Viewer.
- StatusBadge statuses (the only 8): `succeeded`, `pending`, `failed`, `expired`, `refunded`, `processing`, `pending_approval` (warning-600 **outlined** per 07 collision rule 2), plus `draft` rendered as ink-500 outline pill (non-status pill, not a StatusBadge — used only for payout batches).
- `accent-500` never appears in any screen below.
- Every list uses cursor pagination (`?limit=20&starting_after=<id>`, `has_more`).
- "2FA re-prompt" = DM-28.
- The bold markers "Tableau" (UI data table) and "Onglet" (tab) are used consistently for inventory counting.

---

## 0. Global chrome (present on all authenticated pages D-11 → D-42)

### 0.1 Sidebar (240 px, collapsible to icon rail; state persisted per user)

Exact item list, top to bottom (Lucide icon · FR label · route · visible to):

| # | Icône | Libellé FR | Route | Visible pour |
|---|---|---|---|---|
| — | wordmark | Ijim Pay (logo, click → `/`) | `/` | Tous |
| 1 | `layout-dashboard` | Aperçu | `/` | Tous |
| — | group label | **Encaissements** | — | Tous |
| 2 | `arrow-left-right` | Transactions | `/transactions` | Tous |
| 3 | `link` | Liens de paiement | `/links` | Tous |
| 4 | `user-round` | Clients | `/customers` | Tous |
| — | group label | **Décaissements** | — | Tous |
| 5 | `send` | Paiements sortants | `/payouts` | Tous |
| 6 | `users` | Bénéficiaires | `/payouts/beneficiaries` | Tous |
| 7 | `repeat` | Abonnements | `/subscriptions` | Tous |
| — | group label | **Finances** | — | Tous |
| 8 | `wallet` | Solde | `/balance` | Tous |
| 9 | `landmark` | Règlements | `/settlements` | Tous |
| — | group label | **Développeurs** | — | Owner, Admin, Developer |
| 10 | `key-round` | Clés API | `/developers/keys` | Owner, Admin, Developer |
| 11 | `webhook` | Webhooks | `/developers/webhooks` | Owner, Admin, Developer |
| 12 | `scroll-text` | Événements | `/developers/events` | Owner, Admin, Developer |
| 13 | `scroll-text` + suffix "API" | Journal API | `/developers/logs` | Owner, Admin, Developer |
| 14 | `users-round` | Équipe | `/team` | Tous (actions Owner/Admin) |
| 15 | `settings` | Paramètres | `/settings/business` | Tous (edits Owner/Admin) |
| — | footer | `book-open` Documentation → docs.ijimpay.com (nouvel onglet) · `life-buoy` Aide | — | Tous |

Active item: `brand-100` background, `brand-600` icon + text, 3 px `brand-600` left bar. Collapsed rail shows icons only with tooltip labels.

### 0.2 Topbar (64 px)

Left → right:
1. **Sélecteur de marchand** — current merchant name + tier chip (`badge-check` vert "Vérifié Tier 2" / warning "Tier 1" / ink "Tier 0 — sandbox"); click opens DM-29. Hidden if user belongs to a single merchant.
2. **Bascule Test/Réel** — segmented control `flask-conical` "Test" / "Réel". Switching reloads current route in the other mode's dataset. In test mode a persistent amber banner spans the full width under the topbar: `flask-conical` **"Mode test — aucun argent réel ne circule."** + link "Passer en mode réel" (disabled with tooltip "Vérification requise" while tier 0).
3. **Recherche ⌘K** — input placeholder "Rechercher (Ctrl/⌘ K)" → opens D-44.
4. `bell` **Notifications** — unread count badge (danger-600 dot, max "9+") → opens D-43.
5. **Avatar menu** — initials avatar: Mon profil (→ D-42) · `globe` Langue : Français / English · `book-open` Documentation · `log-out` Se déconnecter (POST session end, → `/login`).

### 0.3 Global banners (stacking order, top first)

1. Test mode banner (see above) — amber, permanent in test.
2. KYB banner (info-600) when tier < requested: "Vérification en cours — votre volume est limité." + lien "Voir le statut" → `/settings/verification`.
3. Provider degraded (warning-600, from `GET /channels`): "Les paiements {Orange|MTN} peuvent être lents actuellement." Auto-dismisses when status returns `operational`.
4. Offline detection (danger-600): "Connexion perdue — reconnexion en cours…" (navigator offline event; retries every 5 s).
5. Webhook endpoint failing > 24 h (warning, Developer/Admin/Owner only): "Un point de terminaison webhook échoue depuis 24 h." → `/developers/webhooks`.
6. **Compte suspendu** (danger-600, permanent, non fermable — déclenché par l'ops via docs/13 OM-05 / F-048): « **Compte suspendu — contactez le support.** » Le dashboard passe intégralement en **lecture seule** : toutes les actions de mouvement d'argent (encaisser, payer, rembourser, approvisionner, approuver) et toutes les mutations sont **masquées** (mêmes règles d'affichage que Viewer, pour tous les rôles) ; toute mutation tentée par API renvoie `permission_denied`. Suspension partielle (décaissements uniquement) : les encaissements restent actifs, la section Décaissements est masquée avec bandeau warning.
7. **Session ops — impersonation lecture seule** (danger-600, permanent, non fermable — ouverte par l'ops via docs/13 OM-06): « **Session ops — lecture seule.** » Toutes les mutations sont désactivées (boutons masqués, mêmes règles que Viewer sur toutes les pages, sections Développeurs incluses en lecture) ; la session expire automatiquement après **30 min** (retour à un écran « Session ops terminée. ») ; chaque page vue est journalisée côté ops.

Live updates: WebSocket channel per merchant+mode; fallback 15 s polling on D-11 and D-12 only.

---

## 1. Pages

### D-01 — Connexion / Login
- **Route**: `/login` · **Icon**: lucide `lock` · **Access**: public (redirect `/` if authenticated) · **Purpose**: authenticate a user by phone-or-email + password, with TOTP second step when enrolled.
- **Layout zones**: centered card 400 px on `paper` background; logo top; footer links (Langue `globe`, Aide `life-buoy`, CGU).
- **Data displayed**: none (no API reads pre-auth).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Téléphone ou e-mail | text | E.164 CM (`+237` prefix, 9 chiffres) OU e-mail RFC 5322 | vide | « Entrez un numéro camerounais ou une adresse e-mail valide. » |
| Mot de passe | password (`eye`/`eye-off` toggle) | requis, ≥ 8 caractères | vide | « Mot de passe requis. » |
| Code de vérification (étape 2, si TOTP) | 6 chiffres, auto-submit | 6 chiffres | vide | « Code incorrect. Réessayez. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Se connecter | — (primary) | public | `POST /auth/login` (API addition) → si TOTP requis, affiche étape 2 → session cookie, redirection `/` (ou deep-link mémorisé) |
| Mot de passe oublié ? | — (link) | public | → D-04 |
| Créer un compte | — (link) | public | → D-02 |
| Valider le code | — (primary) | public | `POST /auth/login/totp` (API addition) |

- **Modals/drawers/sheets opened**: none.
- **States**: loading (bouton spinner); erreur identifiants « Identifiants incorrects. Vérifiez et réessayez. »; rate-limited (429) « Trop de tentatives. Réessayez dans {n} minutes. » avec bouton désactivé + compte à rebours; compte suspendu « Ce compte est suspendu. Contactez le support. »; session expirée (arrivée depuis D-48) info banner « Votre session a expiré. Reconnectez-vous. ».
- **Events/notifications triggered**: security email + SMS "Nouvelle connexion depuis un nouvel appareil" on unknown device fingerprint.

### D-02 — Inscription / Signup
- **Route**: `/signup` · **Icon**: lucide `user-round` · **Access**: public · **Purpose**: create a user + merchant shell with phone, OTP, password and business name.
- **Layout zones**: centered card; 3 inline sub-steps with dots progress (Téléphone → Code → Compte).
- **Data displayed**: none.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Numéro de téléphone | tel, préfixe `+237` fixe, affichage `6 70 00 00 00` | 9 chiffres, préfixe mobile CM valide | vide | « Ce numéro n'est pas un mobile camerounais valide. » |
| Code reçu par SMS | 6 chiffres | 6 chiffres, expire 10 min | vide | « Code invalide ou expiré. Demandez un nouveau code. » |
| Adresse e-mail (facultatif) | email | RFC 5322 si rempli | vide | « Adresse e-mail invalide. » |
| Mot de passe | password + jauge de force | ≥ 8 car., 1 chiffre, 1 lettre | vide | « 8 caractères minimum, avec au moins un chiffre. » |
| Nom de l'entreprise | text | 2–80 caractères | vide | « Entrez le nom de votre entreprise. » |
| J'accepte les Conditions d'utilisation | checkbox | requis | non coché | « Vous devez accepter les conditions. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Recevoir le code | `message-square` | public | `POST /auth/signup/otp` (API addition) → étape Code |
| Renvoyer le code | `rotate-cw` | public | même appel; cooldown 30 s affiché « Renvoyer (28 s) » |
| Créer mon compte | — (primary) | public | `POST /auth/signup` (API addition) → session, redirection `/onboarding/business` |

- **Modals/drawers/sheets opened**: none.
- **States**: numéro déjà utilisé « Un compte existe déjà avec ce numéro. Se connecter ? » (lien D-01); OTP rate-limited « Trop de codes demandés. Réessayez dans 15 minutes. »; loading per step.
- **Events/notifications triggered**: welcome email (si e-mail fourni); OTP SMS.

### D-03 — Vérification OTP / OTP verification
- **Route**: `/verify-otp` · **Icon**: lucide `message-square` · **Access**: public (avec jeton de contexte) · **Purpose**: standalone OTP entry used when phone verification is required outside signup (new device, phone change).
- **Layout zones**: centered card; masked phone « Code envoyé au +237 6 •• •• •• 43 ».
- **Data displayed**: masked phone from verification context (`POST /auth/otp` response — API addition).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Code | 6 chiffres, auto-focus, auto-submit | 6 chiffres, 5 essais max | vide | « Code invalide ou expiré. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Valider | — (primary) | public | `POST /auth/otp/verify` (API addition) → poursuit le flux appelant |
| Renvoyer le code | `rotate-cw` | public | cooldown 30 s |
| Changer de numéro | — (link) | public | retour au flux appelant |

- **Modals/drawers/sheets opened**: none.
- **States**: 5 échecs → verrou 15 min « Trop d'essais. Réessayez dans 15 minutes. »; loading.
- **Events/notifications triggered**: OTP SMS.

### D-04 — Mot de passe oublié / Forgot password
- **Route**: `/forgot-password` · **Icon**: lucide `lock` · **Access**: public · **Purpose**: request a password reset link/code by phone or email.
- **Layout zones**: centered card; retour `arrow-left` vers `/login`.
- **Data displayed**: none.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Téléphone ou e-mail | text | mêmes règles que D-01 | vide | « Entrez un numéro ou un e-mail valide. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer le lien | — (primary) | public | `POST /auth/password/forgot` (API addition). Réponse identique que le compte existe ou non (anti-énumération) |
| Je n'ai plus accès à mon numéro | — (link, sous le formulaire) | public | → D-49 (récupération de compte, F-033) |

- **Modals/drawers/sheets opened**: none.
- **States**: succès (toujours) « Si un compte existe, vous recevrez un lien de réinitialisation par SMS ou e-mail d'ici quelques minutes. »; rate-limited.
- **Events/notifications triggered**: reset SMS (code + lien court) ou e-mail (lien, expire 30 min).

### D-05 — Réinitialiser le mot de passe / Reset password
- **Route**: `/reset-password?token=…` · **Icon**: lucide `lock` · **Access**: public avec jeton valide · **Purpose**: set a new password from a reset link.
- **Layout zones**: centered card.
- **Data displayed**: none (token validated server-side).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nouveau mot de passe | password + jauge | ≥ 8 car., 1 chiffre, 1 lettre, ≠ ancien | vide | « 8 caractères minimum, avec au moins un chiffre. » |
| Confirmer le mot de passe | password | identique au précédent | vide | « Les mots de passe ne correspondent pas. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réinitialiser | — (primary) | public | `POST /auth/password/reset` (API addition) → révoque toutes les sessions → D-01 avec bandeau succès « Mot de passe modifié. Connectez-vous. » |

- **Modals/drawers/sheets opened**: none.
- **States**: jeton expiré/invalide — page dédiée « Ce lien a expiré. » + bouton « Demander un nouveau lien » → D-04.
- **Events/notifications triggered**: security email + SMS « Votre mot de passe a été modifié ».

### D-06 — Accepter l'invitation / Accept invite
- **Route**: `/accept-invite?token=…` · **Icon**: lucide `users-round` · **Access**: public avec jeton · **Purpose**: let an invited person join a merchant with an assigned role.
- **Layout zones**: centered card: avatar + « {Inviteur} vous invite à rejoindre {Marchand} en tant que {Rôle} » + explication du rôle (1 phrase par rôle) ; formulaire mot de passe si nouvel utilisateur.
- **Data displayed**: inviter name, merchant name, role — from invite token resolution `GET /invites/{token}` (API addition).
- **Inputs** (nouveaux utilisateurs uniquement):

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Votre nom | text | 2–60 caractères | vide | « Entrez votre nom. » |
| Mot de passe | password | mêmes règles que D-05 | vide | « 8 caractères minimum, avec au moins un chiffre. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Accepter et rejoindre | — (primary) | public | `POST /invites/{token}/accept` (API addition) → session → `/` du marchand |
| Refuser | — (secondary) | public | `POST /invites/{token}/decline` (API addition) → page « Invitation refusée. » |

- **Modals/drawers/sheets opened**: none.
- **States**: invitation expirée « Cette invitation a expiré. Demandez à {Inviteur} de la renvoyer. »; déjà membre → redirection `/` + toast « Vous êtes déjà membre de {Marchand}. ».
- **Events/notifications triggered**: e-mail + cloche à l'inviteur « {Nom} a rejoint votre équipe » (cf. matrice docs/08 §C).

### D-07 — Onboarding · Infos entreprise / Business info
- **Route**: `/onboarding/business` · **Icon**: lucide `badge-check` · **Access**: Owner (créateur du compte) · **Purpose**: capture legal identity of the business (step 1/4, progress bar « Étape 1 sur 4 »).
- **Layout zones**: narrow form column 560 px; progress bar top; aside right: encart « Pourquoi ces informations ? » (KYB, réglementation).
- **Data displayed**: pre-filled business name from signup (`GET /merchant` — API addition).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom commercial | text | 2–80 car. | nom du signup | « Entrez le nom commercial. » |
| Raison sociale | text | 2–120 car. | vide | « Entrez la raison sociale (telle qu'au RCCM). » |
| Forme juridique | select: Entreprise individuelle / SARL / SA / GIE / Association / Autre | requis | — | « Choisissez une forme juridique. » |
| Secteur d'activité | select (liste fermée: Commerce de détail, Restauration, Services, Éducation, Transport, E-commerce, Immobilier, Santé, Autre) | requis | — | « Choisissez un secteur. » |
| Ville | text | 2–60 car. | vide | « Entrez la ville. » |
| Adresse | text | 5–160 car. | vide | « Entrez l'adresse. » |
| Numéro RCCM (si immatriculé) | text | format `RC/XXX/année/lettre/numéro`, facultatif Tier 1 | vide | « Format RCCM invalide (ex. RC/DLA/2024/B/1234). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer et continuer | — (primary) | Owner | `PATCH /merchant` (API addition) → D-08 |
| Enregistrer et quitter | — (link) | Owner | même appel → `/` (bandeau KYB rappellera de finir) |

- **Modals/drawers/sheets opened**: none.
- **States**: loading; resumable (champs re-remplis au retour); erreur serveur inline « Impossible d'enregistrer. Réessayez. ».
- **Events/notifications triggered**: none.

### D-08 — Onboarding · Documents / Documents
- **Route**: `/onboarding/documents` · **Icon**: lucide `badge-check` · **Access**: Owner · **Purpose**: upload KYB documents (step 2/4).
- **Layout zones**: progress bar; document card list; aside « Documents acceptés » (formats, exemples).
- **Data displayed**: per-document status from `GET /merchant/kyb_documents` (API addition): `manquant | envoyé | approuvé | rejeté (motif)`.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| RCCM / attestation d'immatriculation | fichier drag-drop (`upload`) ou caméra mobile (`camera`) | PDF/JPG/PNG ≤ 10 Mo | — | « Fichier trop lourd (10 Mo max) ou format non accepté (PDF, JPG, PNG). » |
| Pièce d'identité du dirigeant (CNI/passeport) | fichier | idem | — | idem |
| Preuve de compte de règlement (capture MoMo/OM ou RIB) | fichier | idem | — | idem |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer les documents et continuer | — (primary) | Owner | `POST /merchant/kyb_documents` (API addition, multipart) → D-09 |
| Compléter plus tard | — (link) | Owner | → D-09 sans upload; tier reste 0 |
| Remplacer un document rejeté | `rotate-cw` | Owner | re-upload; affiche le motif de rejet FR sous la carte |

- **Modals/drawers/sheets opened**: none.
- **States**: upload en cours (barre de progression par fichier); rejeté — carte danger avec motif exact des ops (ex. « Photo illisible — reprenez la pièce à plat, sans reflet. »); tout approuvé — cartes vertes `circle-check`.
- **Events/notifications triggered**: à l'issue de la revue ops: push + e-mail + SMS « Vérification approuvée / refusée » (matrice 08 §C).

### D-09 — Onboarding · Compte de règlement / Settlement account
- **Route**: `/onboarding/account` · **Icon**: lucide `landmark` · **Access**: Owner · **Purpose**: register where settlements are paid (step 3/4).
- **Layout zones**: progress bar; form; aside « Quand suis-je payé ? » (T+1 par défaut).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Type de compte | radio: MTN MoMo / Orange Money / Compte bancaire | requis | MTN MoMo | « Choisissez un type de compte. » |
| Numéro MoMo/OM | tel `+237` | préfixe cohérent avec l'opérateur choisi (MTN 650–654/67x/680–684, Orange 655–659/69x/685–689) | vide | « Ce numéro ne correspond pas à l'opérateur choisi. » |
| Nom du titulaire | text | 2–80 car., doit correspondre au KYB | vide | « Entrez le nom du titulaire. » |
| IBAN / RIB (si banque) | text | 27 car. CM `CMxx…` | vide | « RIB invalide. » |

- **Data displayed**: current settlement destination (`GET /merchant` — API addition).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer et continuer | — (primary) | Owner | `PATCH /merchant/settlement_account` (API addition) → D-10 |
| Retour | `arrow-left` | Owner | → D-08 (état conservé) |

- **Modals/drawers/sheets opened**: none.
- **States**: loading; vérification du nom (si dispo opérateur) — hint « Nom vérifié : NGOUM Jude ✓ » ou warning « Le nom renvoyé par l'opérateur diffère : vérifiez le numéro. ».
- **Events/notifications triggered**: none (changements ultérieurs passent par DM-17 avec 2FA + délai 24 h).

### D-10 — Onboarding · Terminé / Done
- **Route**: `/onboarding/done` · **Icon**: lucide `circle-check` · **Access**: Owner · **Purpose**: confirm completion, show tier status and route the merchant into test mode (step 4/4).
- **Layout zones**: centered success moment (check scale-in, brand-600 flash — signature interaction); tier status card; next-steps checklist.
- **Data displayed**: `kyb_status` + tier from `GET /merchant`; checklist state (lien créé ? clé API utilisée ?).
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Explorer en mode test | `flask-conical` (primary) | Owner | bascule mode test → `/` |
| Créer mon premier lien | `link` | Owner | → `/links/new` (mode test) |
| Voir le statut de vérification | `badge-check` | Owner | → `/settings/verification` |

- **Modals/drawers/sheets opened**: none.
- **States**: docs complets — « Vos documents sont en cours de revue (sous 24 h ouvrées). » ; docs incomplets — warning « Il manque des documents — votre compte reste en mode test. » + CTA retour D-08.
- **Events/notifications triggered**: none.

### D-11 — Accueil / Home
- **Route**: `/` · **Icon**: lucide `layout-dashboard` · **Access**: Tous · **Purpose**: today's money picture and fastest paths to collect, link and pay.
- **Layout zones**: header row (salutation + date) → stat tiles (4) → provider health strip → two-column: volume chart (2/3) + quick actions card (1/3) → latest transactions list.
- **Data displayed**:
  - « Bonjour {prénom} » + date `fr-CM` (session user).
  - Tuile 1 **Encaissé aujourd'hui**: somme des `charges.status=succeeded` du jour (`GET /charges?status=succeeded&created[gte]=…`) en AmountText XL + « vs hier {±n %} ».
  - Tuile 2 **Transactions**: count du jour + anneau taux de réussite (succeeded / total terminal).
  - Tuile 3 **Solde disponible** `wallet`: `GET /balance` → `available` (clic → `/balance`).
  - Tuile 4 **En attente de règlement**: `GET /balance` → `pending`.
  - Bandeau santé `activity`: `GET /channels` — « MTN ● opérationnel · Orange ● dégradé ».
  - Graphique volume 14 jours empilé par canal (agrégat de `GET /charges`; endpoint stats dédié en **API additions**).
  - **8 dernières transactions** (TxRow) — `GET /charges?limit=8` → clic ouvre DM-01.
- **Inputs**: aucun (les saisies passent par DM-04).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer un lien | `link` | Owner, Admin, Finance, Developer | → `/links/new` |
| Encaisser | `hand-coins` | Owner, Admin, Finance | ouvre DM-04 |
| Payer | `send` | Owner, Admin, Finance | → `/payouts/new` |
| Voir tout (transactions) | — (link) | Tous | → `/transactions` |

- **Modals/drawers/sheets opened**: DM-01, DM-04.
- **States**: loading (skeleton tiles + rows); empty (nouveau marchand) — carte checklist « Bien démarrer » : ① Vérifiez votre entreprise ② Créez votre premier lien ③ Testez l'API ④ Passez en mode réel — chaque item avec icône et état fait/à faire; error — tuiles remplacées par « — » + toast « Impossible de charger les données. Réessayer »; permission: Viewer voit tout sauf les 3 boutons d'action rapide.
- **Events/notifications triggered**: none (page consomme le temps réel: nouvelles transactions apparaissent en tête de liste avec surlignage brand-100 2 s).

### D-12 — Transactions / Transactions
- **Route**: `/transactions` (détail: drawer DM-01 sur `/transactions/:id`) · **Icon**: lucide `arrow-left-right` · **Access**: Tous · **Purpose**: search, inspect and export every charge.
- **Layout zones**: header (titre + Exporter) → barre de filtres → **Tableau** → pagination.
- **Data displayed**: **Tableau** TxRow (`GET /charges` + filtres): StatusBadge (`status`), téléphone/nom client (`customer`), description, ChannelChip (`channel`), AmountText (`amount`), heure (`created_at`); au survol d'une ligne `failed`: libellé humain du `failure_code` (ex. `insufficient_payer_funds` → « Solde du payeur insuffisant »).
- **Inputs** (barre de filtres):

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Période | date-range picker (`calendar`) | début ≤ fin, max 366 jours | 30 derniers jours | « Période invalide. » |
| Statut | multi-chips StatusBadge (8) | — | tous | — |
| Canal | multi-select MTN / Orange | — | tous | — |
| Montant min / max | 2 × number FCFA | min ≤ max, entiers ≥ 0 | vides | « Le minimum dépasse le maximum. » |
| Recherche | text (`search`) | ≥ 2 caractères ; réf, téléphone ou nom | vide | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Exporter | `download` | Owner, Admin, Finance, Viewer | ouvre DM-35 (CSV; > 10 000 lignes → export asynchrone par e-mail) |
| Vues enregistrées : Aujourd'hui / Échecs récents | `list-filter` | Tous | applique un jeu de filtres prédéfini (stocké côté client) |
| Ouvrir le détail | — (row click) | Tous | ouvre DM-01 (URL `/transactions/:id`, liste conservée derrière) |

- **Modals/drawers/sheets opened**: DM-01, DM-35.
- **States**: loading (skeleton 10 lignes); empty « Aucune transaction — créez un lien de paiement pour commencer. » + CTA `/links/new`; empty (filtres actifs) « Aucun résultat pour ces filtres. » + « Réinitialiser les filtres »; error « Impossible de charger les transactions. Réessayer »; pending rows: mise à jour en direct (websocket/poll 15 s) avec `clock` pulsant.
- **Events/notifications triggered**: export prêt → e-mail avec lien de téléchargement signé (expire 72 h, N-09) + cloche.

### D-13 — Liens de paiement / Payment links
- **Route**: `/links` · **Icon**: lucide `link` · **Access**: Tous · **Purpose**: manage all payment links, their performance and sharing.
- **Layout zones**: header (titre + « Nouveau lien ») → toggle vue cartes/**Tableau** → grille ou tableau → pagination.
- **Data displayed**: `GET /payment_links` — par lien: titre (`title`), mini-QR, montant (`amount` / « Montant libre » si `amount_type=open`), total encaissé, nombre de paiements, interrupteur actif (`active`), date de création; URL `https://pay.ijimpay.com/l/{slug}`.
- **Inputs**: aucun (recherche via ⌘K).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Nouveau lien | `link` (primary) | Owner, Admin, Finance, Developer | → D-14 |
| Copier l'URL | `copy` | Tous | presse-papiers + toast « Lien copié » |
| Afficher le QR | `qr-code` | Tous | ouvre DM-05 |
| Partager | `share-2` | Tous | ouvre DM-06 |
| Modifier | `pencil` | Owner, Admin, Finance | → D-15 (panneau édition) |
| Désactiver | `trash-2` | Owner, Admin, Finance (le créateur du lien peut désactiver les siens, tout rôle créateur — F-011/docs/15) | ouvre DM-07 |
| Basculer vue cartes/tableau | `layers` | Tous | préférence locale |

- **Modals/drawers/sheets opened**: DM-05, DM-06, DM-07.
- **States**: loading skeleton cartes; empty — illustration + « Créez votre premier lien en 30 secondes. » + CTA « Nouveau lien »; error standard; lien expiré — pill `timer-off` « Expiré » sur la carte.
- **Events/notifications triggered**: none.

### D-14 — Nouveau lien / New payment link
- **Route**: `/links/new` · **Icon**: lucide `link` · **Access**: Owner, Admin, Finance, Developer · **Purpose**: create a link in under 30 seconds with a live preview.
- **Layout zones**: two columns — form (gauche) · **aperçu téléphone en direct** de la page hébergée (droite, cadre téléphone); success screen après création (URL + QR + partage).
- **Data displayed**: preview rendered from form state (mêmes règles visuelles que docs/09).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Titre | text | 2–80 car. | vide | « Donnez un titre à votre lien. » |
| Type de lien | radio: Simple / Catalogue (produits) | requis | Simple | — |
| Type de montant (si Simple) | radio: Montant fixe / Montant libre | requis | Fixe | — |
| Montant | number FCFA | entier, 100 – 5 000 000 (plafond payeur docs/14 C-02; le plafond effectif peut être réduit par le tier) | vide | « Montant entre 100 FCFA et 5 000 000 FCFA. » |
| Montant minimum (si libre) | number FCFA | entier ≥ 100 | 100 | « Minimum 100 FCFA. » |
| Produits (si Catalogue) | répéteur 1–20 lignes: nom (2–80 car.) + image (JPG/PNG ≤ 2 Mo, facultatif) + prix (entier, 100 – 5 000 000 FCFA) | ≥ 1 produit valide | 1 ligne vide | « Ajoutez au moins un produit (nom et prix). » · « Prix entre 100 FCFA et 5 000 000 FCFA. » |
| Sélection de quantité (si Catalogue) | switch — le payeur choisit la quantité par produit (rendu par C-03, docs/14) | — | activé | — |
| Description (facultatif) | textarea | ≤ 240 car. | vide | « 240 caractères maximum. » |
| Image (facultatif) | fichier | JPG/PNG ≤ 2 Mo | — | « Image trop lourde (2 Mo max). » |
| Réutilisable | switch: Réutilisable / Usage unique | — | Réutilisable | — |
| Expiration (facultatif) | date-time picker | > maintenant | jamais | « La date d'expiration doit être future. » |
| Message de succès personnalisé (facultatif) | text | ≤ 160 car. | « Paiement reçu — merci ! » | « 160 caractères maximum. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer le lien | — (primary) | Owner, Admin, Finance, Developer | `POST /payment_links` → écran succès (URL + `qr_png_url` + boutons Copier/QR/Partager) |
| Copier l'URL (succès) | `copy` | idem | presse-papiers |
| Partager (succès) | `share-2` | idem | ouvre DM-06 |
| Annuler | — (link) | idem | → `/links` (confirmation si formulaire modifié: « Abandonner ce lien ? ») |

- **Modals/drawers/sheets opened**: DM-05, DM-06.
- **States**: loading (bouton spinner); erreur validation inline; erreur serveur « Impossible de créer le lien. Réessayez. »; succès = success moment (check scale-in).
- **Events/notifications triggered**: none.

### D-15 — Détail du lien / Link detail
- **Route**: `/links/:id` · **Icon**: lucide `link` · **Access**: Tous (édition Owner, Admin, Finance) · **Purpose**: performance stats and edition of one link.
- **Layout zones**: header (titre + StatusPill actif/expiré + actions) → stat tiles (Encaissé, Paiements, Conversion vues→payés) → **Tableau** transactions filtrées sur ce lien → panneau d'édition (mêmes champs que D-14, montant non modifiable si des paiements existent).
- **Data displayed**: `GET /payment_links/{id}` (titre, montant, `active`, `expires_at`, URL, QR); `GET /charges?payment_link={id}` (**API additions** — filtre) pour le tableau; compteur de vues (analytics — **API additions**).
- **Inputs**: formulaire d'édition = champs de D-14 (mêmes validations; « Montant » verrouillé avec tooltip « Non modifiable : des paiements existent déjà » si ≥ 1 paiement).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer les modifications | — (primary) | Owner, Admin, Finance | `PATCH /payment_links/{id}` (**API additions**) |
| Copier l'URL | `copy` | Tous | presse-papiers |
| Afficher le QR | `qr-code` | Tous | DM-05 |
| Partager | `share-2` | Tous | DM-06 |
| Désactiver | `trash-2` | Owner, Admin, Finance (ou créateur du lien) | DM-07 → `POST /payment_links/{id}/deactivate` |

- **Modals/drawers/sheets opened**: DM-05, DM-06, DM-07, DM-01 (clic ligne transaction).
- **States**: loading; lien introuvable → D-45 (404); désactivé — bandeau ink « Ce lien est désactivé. Il ne peut plus recevoir de paiements. »; permission: Viewer sans panneau d'édition.
- **Events/notifications triggered**: none.

### D-16 — Session de paiement / Checkout session detail
- **Route**: `/checkout-sessions/:id` · **Icon**: lucide `shopping-cart` · **Access**: Tous · **Purpose**: read-only inspection of an e-commerce checkout session and its charge.
- **Layout zones**: header (montant XL + StatusBadge) → carte panier (snapshot) → carte charge liée → carte technique (`success_url`, `cancel_url`, `created_at`).
- **Data displayed**: `GET /checkout_sessions/{id}` — statut, montant, cart snapshot, URLs; charge liée → lien vers DM-01.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Voir la transaction | `arrow-left-right` | Tous | ouvre DM-01 sur la charge liée |
| Copier l'identifiant | `copy` | Tous | presse-papiers |

- **Modals/drawers/sheets opened**: DM-01.
- **States**: loading; session expirée — StatusBadge `expired`; introuvable → D-45.
- **Events/notifications triggered**: none.

### D-17 — Clients / Customers
- **Route**: `/customers` · **Icon**: lucide `user-round` · **Access**: Tous · **Purpose**: light CRM over phone-keyed customers auto-created from charges.
- **Layout zones**: header (titre + recherche) → **Tableau** → pagination.
- **Data displayed**: `GET /customers` — téléphone, nom, total dépensé, nb transactions, dernière activité.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Recherche | text (`search`) | ≥ 2 car. (téléphone ou nom) | vide | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir la fiche | — (row click) | Tous | → D-18 |
| Encaisser ce client | `hand-coins` | Owner, Admin, Finance | ouvre DM-04 pré-rempli (téléphone + nom) |

- **Modals/drawers/sheets opened**: DM-04.
- **States**: loading skeleton; empty « Aucun client pour l'instant — ils apparaîtront après votre premier encaissement. »; error standard.
- **Events/notifications triggered**: none.

### D-18 — Fiche client / Customer detail
- **Route**: `/customers/:id` · **Icon**: lucide `user-round` · **Access**: Tous · **Purpose**: one customer's profile, history, subscriptions and notes.
- **Layout zones**: header (nom + téléphone + total dépensé) → colonnes: transactions (**Tableau**, `GET /charges?customer_phone=`) · abonnements actifs (cartes, `GET /subscriptions?customer=` — filtre en **API additions**) · notes internes.
- **Data displayed**: `GET /customers/{id}`; charges; subscriptions; notes (**API additions**).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nouvelle note | textarea | 1–500 car. | vide | « La note ne peut pas être vide. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Encaisser | `hand-coins` | Owner, Admin, Finance | DM-04 pré-rempli |
| Ajouter une note | `pencil` | Owner, Admin, Finance | `POST /customers/{id}/notes` (**API additions**) |
| Modifier le nom | `pencil` | Owner, Admin | `PATCH /customers/{id}` (**API additions**) |

- **Modals/drawers/sheets opened**: DM-01 (ligne transaction), DM-04.
- **States**: loading; introuvable → D-45; client sans nom — affiche « Client 6 70 00 00 43 ».
- **Events/notifications triggered**: none.

### D-19 — Paiements sortants / Payouts
- **Route**: `/payouts` · **Icon**: lucide `send` · **Access**: Tous (création Finance/Admin/Owner) · **Purpose**: monitor all payouts and batches, entry point to approvals.
- **Layout zones**: header (titre + « Nouveau paiement ») → tabs → contenu.
- **Tabs**:
  - **Onglet «Paiements»** — **Tableau** de tous les payout items (`GET /payouts`): StatusBadge (dont `pending_approval` en warning-600 outline), bénéficiaire (nom + téléphone), ChannelChip, AmountText (négatif ink), référence, lot parent (lien), date. Filtres statut + période.
  - **Onglet «Lots»** — **Tableau** des batches (`GET /payout_batches` — liste en **API additions**): statut (draft pill / pending_approval / processing avec barre de progression / completed / partially_failed), nb d'éléments, total, créé par, approuvé par, date. Clic → D-24.
  - **Onglet «Bénéficiaires»** — raccourci vers D-25 (même contenu, embarqué).
- **Data displayed**: cf. tabs; bandeau bloquant si solde du portefeuille insuffisant pour des lots en attente: « Solde du portefeuille de paiement insuffisant. » + CTA « Approvisionner » (DM-16).
- **Inputs**: filtres (Statut multi-chips; Période date-range — mêmes règles que D-12).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Nouveau paiement | `send` (primary) | Owner, Admin, Finance | → D-20 |
| Ouvrir un lot | — (row click) | Tous | → D-24 |
| Exporter | `download` | Owner, Admin, Finance, Viewer | DM-35 |

- **Modals/drawers/sheets opened**: DM-16, DM-35.
- **States**: loading; empty « Aucun paiement sortant. Payez un fournisseur ou lancez une paie en quelques clics. » + CTA; canal indisponible (`GET /channels` = down) — note `triangle-alert` « Orange Money est indisponible — les paiements Orange sont suspendus. »; permission: Viewer/Developer sans « Nouveau paiement ».
- **Events/notifications triggered**: none (les événements partent des flux de création/approbation).

### D-20 — Nouveau paiement / New payout
- **Route**: `/payouts/new` · **Icon**: lucide `send` · **Access**: Owner, Admin, Finance · **Purpose**: start a single payout or a bulk run (step 1 of the payout wizard).
- **Layout zones**: choix segmenté **Individuel / En masse**; formulaire (individuel) ou zone d'upload CSV + listes de paie enregistrées (en masse); stepper « 1 Saisie · 2 Colonnes · 3 Validation · 4 Révision » (en masse).
- **Data displayed**: bénéficiaires (`GET /beneficiaries` — **API additions**) pour le picker; listes de paie (`GET /payroll_lists` — **API additions**); solde portefeuille (`GET /balance`).
- **Inputs** (Individuel):

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Bénéficiaire | combobox (recherche) ou « Nouveau numéro » | requis | vide | « Choisissez ou saisissez un bénéficiaire. » |
| Téléphone (nouveau bénéficiaire) | tel `+237` | préfixe MTN/Orange valide; auto-détection canal, modifiable | vide | « Numéro mobile money invalide. » |
| Nom du bénéficiaire | text | 2–80 car.; hint vérification de nom opérateur si dispo | vide | « Entrez le nom du bénéficiaire. » |
| Canal | select MTN MoMo / Orange Money | requis (pré-rempli par préfixe) | auto | « Choisissez un canal. » |
| Montant | number FCFA | entier, 100 – plafond du tier | vide | « Montant entre 100 FCFA et votre plafond ({plafond} FCFA). » |
| Motif / référence | text | 3–60 car., unique par marchand | vide | « Cette référence est déjà utilisée. » |

- **Inputs** (En masse):

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Fichier CSV | drag-drop (`file-up`) | .csv ≤ 5 Mo, ≤ 1 000 lignes (v1, aligné F-015), UTF-8 | — | « Fichier invalide : CSV UTF-8 requis. » · > 1 000 lignes : « Fichier trop grand — 1 000 lignes maximum, divisez le fichier. » |
| Liste de paie enregistrée | select | — | — | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Continuer (individuel) | — (primary) | Owner, Admin, Finance | → D-23 (révision, 1 élément) |
| Télécharger le modèle CSV | `download` | idem | fichier `modele-paiements-ijimpay.csv` (colonnes: telephone, nom, montant, motif) |
| Importer le fichier | `file-up` | idem | upload → D-21 |
| Utiliser une liste de paie | `users` | idem | pré-charge les éléments → D-23 |
| Annuler | — (link) | idem | → `/payouts` (confirm si saisie) |

- **Modals/drawers/sheets opened**: none.
- **States**: canal down — option désactivée avec `triangle-alert`; loading upload (barre de progression); erreur parse « Impossible de lire ce fichier. Vérifiez qu'il s'agit d'un CSV. ».
- **Events/notifications triggered**: none.

### D-21 — Paiement en masse · Correspondance des colonnes / CSV column mapper
- **Route**: `/payouts/new/mapping` · **Icon**: lucide `table` · **Access**: Owner, Admin, Finance · **Purpose**: map the uploaded CSV's columns to required payout fields (bulk step 2/4).
- **Layout zones**: stepper; aperçu des 5 premières lignes du fichier (**Tableau** brut); 4 sélecteurs de correspondance; note « {n} lignes détectées ».
- **Data displayed**: colonnes détectées + échantillon (parse local); auto-mapping proposé quand l'en-tête correspond (telephone/nom/montant/motif).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Colonne « Téléphone » | select (colonnes du fichier) | requis, unique | auto | « Associez une colonne au téléphone. » |
| Colonne « Nom » | select | requis, unique | auto | « Associez une colonne au nom. » |
| Colonne « Montant » | select | requis, unique | auto | « Associez une colonne au montant. » |
| Colonne « Motif / référence » | select ou « Générer automatiquement » | — | auto | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Valider les colonnes | — (primary) | Owner, Admin, Finance | lance la validation ligne à ligne → D-22 |
| Retour | `arrow-left` | idem | → D-20 (fichier conservé) |

- **Modals/drawers/sheets opened**: none.
- **States**: deux colonnes mappées sur le même champ — erreur inline « Chaque colonne ne peut être utilisée qu'une fois. »; fichier sans en-têtes — les selects affichent « Colonne A/B/C… ».
- **Events/notifications triggered**: none.

### D-22 — Paiement en masse · Rapport de validation / Validation report
- **Route**: `/payouts/new/validation` · **Icon**: lucide `list-checks` · **Access**: Owner, Admin, Finance · **Purpose**: show per-row validation results before anything is created (bulk step 3/4).
- **Layout zones**: stepper; bandeau résumé (« 47 lignes valides · 3 erreurs ») ; **Tableau** des erreurs uniquement (toggle « Voir toutes les lignes ») : n° ligne, champ fautif, valeur, erreur FR; pied avec choix de poursuite.
- **Data displayed**: résultat de validation locale + serveur (`POST /payout_batches/validate` — **API additions**): erreurs typées — « Numéro invalide », « Montant non entier », « Montant sous 100 FCFA », « Référence en double (ligne {n}) », « Préfixe inconnu — canal indéterminé », « Nom manquant ».
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Corriger une valeur en ligne | édition inline dans le tableau | mêmes règles que D-20 individuel | valeur du fichier | messages ci-dessus |
| Exclure les lignes en erreur et continuer | checkbox | — | non coché | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Revalider | `rotate-cw` | Owner, Admin, Finance | rejoue la validation après corrections |
| Continuer vers la révision | — (primary, actif si 0 erreur ou exclusion cochée) | idem | → D-23 |
| Télécharger le rapport d'erreurs | `download` | idem | CSV des lignes en erreur |
| Retour | `arrow-left` | idem | → D-21 |

- **Modals/drawers/sheets opened**: none.
- **States**: 100 % valide — bandeau succès « Toutes les lignes sont valides. »; 100 % erreurs — bouton continuer désactivé « Aucune ligne valide — corrigez le fichier. »; validation en cours (barre de progression « Validation {x}/{n}… »).
- **Events/notifications triggered**: none.

### D-23 — Paiement · Révision / Payout review
- **Route**: `/payouts/new/review` · **Icon**: lucide `check-check` · **Access**: Owner, Admin, Finance · **Purpose**: final review of totals, fees and funding before creating the payout(s) (bulk step 4/4; also the single-payout confirm page).
- **Layout zones**: récapitulatif (nb d'éléments, total brut, frais, total à débiter) → **Tableau** par élément (bénéficiaire, canal, montant, motif) → carte financement → pied d'action.
- **Data displayed**: totaux calculés; frais (barème par canal — endpoint en **API additions**); vérification de financement `GET /balance` → « Solde du portefeuille de paiement : 1 200 000 FCFA — suffisant ✓ » (success-600) ou « insuffisant » (danger-600) + CTA « Approvisionner » (DM-16); mention du circuit d'approbation: « Ce paiement nécessitera l'approbation d'un Admin ou du Owner. » si seuil dépassé ou créateur = Finance.
- **Inputs**: aucun (tout est en lecture; retour pour corriger).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer le paiement | `send` (primary; jamais focus par défaut) | Owner, Admin, Finance | Individuel: confirmation restatée « Envoyer {montant} FCFA à {nom} ({numéro}) ? » (ConfirmModal docs/07 §5) + re-prompt 2FA DM-28 (parité avec MS-08 mobile) → `POST /payouts` (Idempotency-Key) → succès/`processing`: → D-19 (onglet Paiements) avec toast « Paiement envoyé à {nom}. » et la ligne en tête · `pending_approval`: → D-19 avec bandeau « En attente d'approbation par un Admin ou le Owner. » · En masse: `POST /payout_batches` → statut `draft` puis `pending_approval` si requis → D-24 |
| Approvisionner | `plus-circle` | Owner, Admin, Finance | DM-16 |
| Retour | `arrow-left` | idem | étape précédente (état conservé) |

- **Modals/drawers/sheets opened**: DM-16, DM-28 (individuel).
- **States**: solde insuffisant — bouton « Confirmer » désactivé + bandeau danger; canal down — éléments concernés marqués `triangle-alert` « suspendu — sera réessayé »; erreur idempotence (rejeu) — redirection vers le lot existant + toast « Ce lot existe déjà. »; échec individuel au submit (`provider_error` sur `POST /payouts`) — la page reste sur D-23 avec bloc danger « Le paiement n'a pas pu être envoyé ({failure_code humanisé}). Aucune somme n'a été débitée. » + boutons « Réessayer » (rejoue avec la même Idempotency-Key) et « Voir mes paiements » → D-19; si le payout est créé puis échoue côté provider, il apparaît `failed` dans D-19 avec retry DM-12.
- **Events/notifications triggered**: si `pending_approval`: push + e-mail + cloche aux approbateurs « Un lot de paiements attend votre approbation » (matrice 08 §C).

### D-24 — Détail du lot / Payout batch detail
- **Route**: `/payouts/batches/:id` · **Icon**: lucide `layers` · **Access**: Tous (approbation Admin/Owner) · **Purpose**: track one batch item-by-item, approve/reject, retry failures, download the report.
- **Layout zones**: header statut (pill/StatusBadge + barre de progression si `processing`) → panneau d'approbation (si `pending_approval`, visible Admin/Owner non-créateurs) → **Tableau** par élément → bande d'audit (« Créé par Aïcha K. le 02 août 2026, 14:05 · Approuvé par J. Ngoum ») → pied (rapport).
- **Data displayed**: `GET /payout_batches/{id}` — statut, totaux, `created_by`, `approved_by`; par élément: bénéficiaire, ChannelChip, AmountText, StatusBadge, `failure_code` humanisé (ex. `payout_limit_exceeded` → « Plafond de paiement dépassé »); résumé d'échec partiel « 47/50 réussis — 3 échecs à revoir ».
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approuver | `check-check` (primary) | Admin, Owner (≠ créateur) | ouvre DM-08 (récap + 2FA) → `POST /payout_batches/{id}/approve` |
| Rejeter | — (destructive outline) | Admin, Owner (≠ créateur) | ouvre DM-09 |
| Réessayer l'élément | `rotate-cw` (par ligne échouée) | Owner, Admin, Finance | DM-12 → `POST /payouts/{id}/retry` (**API additions**) |
| Télécharger le rapport | `download` | Tous | CSV/PDF du lot (`GET /payout_batches/{id}/report` — **API additions**) |

- **Modals/drawers/sheets opened**: DM-08, DM-09, DM-12.
- **States**: `processing` — barre de progression + compte en direct « 32/50 traités »; `pending_approval` vu par le créateur — panneau remplacé par « En attente d'approbation par un Admin ou le Owner. »; `pending_approval` (expiration, aligné F-016) — bandeau warning pour approbateurs et créateur « Sans approbation, ce lot expirera le {date} (7 jours). » avec compte à rebours en jours ; rappel automatique N-12 renvoyé aux approbateurs à 48 h ; à 7 jours sans approbation le lot repasse en `draft` + bandeau au créateur « Lot expiré sans approbation — repassé en brouillon. » (+ cloche); `partially_failed` — bandeau warning avec résumé; rejeté — bandeau danger « Rejeté par {nom} : “{motif}” ».
- **Events/notifications triggered**: approbation → traitement; fin de lot → push + e-mail (rapport joint) « Lot terminé : 50/50 réussis » ou « Lot partiellement échoué »; rejet → push + cloche au créateur.

### D-25 — Bénéficiaires / Beneficiaries
- **Route**: `/payouts/beneficiaries` · **Icon**: lucide `users` · **Access**: Tous (CRUD Owner, Admin, Finance) · **Purpose**: directory of payout recipients and named payroll lists for one-click monthly runs.
- **Layout zones**: header (titre + « Ajouter ») → tabs → contenu.
- **Tabs**:
  - **Onglet «Bénéficiaires»** — **Tableau**: nom (+ badge `badge-check` « Nom vérifié » si confirmé opérateur), téléphone, ChannelChip, nb de paiements reçus, dernier paiement. Recherche.
  - **Onglet «Listes de paie»** — cartes de listes nommées (« Salaires — équipe boutique », 12 membres, total 1 450 000 FCFA); clic → DM-30.
- **Data displayed**: `GET /beneficiaries`, `GET /payroll_lists` (**API additions**).
- **Inputs**: recherche (≥ 2 car.).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ajouter un bénéficiaire | `plus-circle` (primary) | Owner, Admin, Finance | ouvre DM-10 |
| Modifier | `pencil` | Owner, Admin, Finance | DM-10 pré-rempli |
| Supprimer | `trash-2` | Owner, Admin | DM-11 |
| Nouvelle liste de paie | `users` | Owner, Admin, Finance | DM-30 |
| Payer cette liste | `send` | Owner, Admin, Finance | → D-23 pré-chargé avec la liste |

- **Modals/drawers/sheets opened**: DM-10, DM-11, DM-30.
- **States**: empty « Aucun bénéficiaire enregistré. Ajoutez vos fournisseurs et employés pour payer en un clic. »; loading; error standard.
- **Events/notifications triggered**: none.

### D-26 — Abonnements / Subscriptions
- **Route**: `/subscriptions` · **Icon**: lucide `repeat` · **Access**: Tous (actions Owner, Admin, Finance) · **Purpose**: list and manage recurring billing subscriptions and dunning.
- **Layout zones**: header (titre + « Nouvel abonnement ») → tabs → **Tableau** → pagination.
- **Tabs**:
  - **Onglet «Abonnements»** — **Tableau** (`GET /subscriptions`): client (nom + téléphone), plan, StatusBadge-like pills `active` (success) / `past_due` (warning) / `paused` (ink) / `canceled` (ink barré), prochaine échéance, montant. Filtre statut. Sélection multiple sur les `past_due` → action groupée « Relancer ».
  - **Onglet «Plans»** — raccourci embarqué de D-27.
- **Data displayed**: cf. tabs.
- **Inputs**: filtre statut (multi-chips), recherche client.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Nouvel abonnement | `repeat` (primary) | Owner, Admin, Finance | ouvre DM-33 |
| Ouvrir le détail | — (row click) | Tous | → D-28 |
| Relancer (sélection past_due) | `message-circle` | Owner, Admin, Finance | DM-34 (confirmation groupée) |

- **Modals/drawers/sheets opened**: DM-33, DM-34.
- **States**: empty « Aucun abonnement. Créez un plan puis abonnez vos clients réguliers. » + CTA « Créer un plan »; loading; error standard.
- **Events/notifications triggered**: relance → SMS/WhatsApp au client avec lien de paiement (dunning, matrice 08 §C).

### D-27 — Plans / Plans
- **Route**: `/subscriptions/plans` · **Icon**: lucide `repeat` · **Access**: Tous (CRUD Owner, Admin, Finance) · **Purpose**: manage the catalog of recurring plans.
- **Layout zones**: header (titre + « Nouveau plan ») → **Tableau**: nom, montant, intervalle (Semaine/Mois/Année), abonnés actifs, créé le.
- **Data displayed**: `GET /plans` (**API additions** — liste; création dans 02).
- **Inputs**: aucun (formulaire dans DM-13).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Nouveau plan | `plus-circle` (primary) | Owner, Admin, Finance | ouvre DM-13 → `POST /plans` |
| Modifier | `pencil` | Owner, Admin, Finance | DM-13 (montant verrouillé si abonnés actifs, tooltip « Créez un nouveau plan pour changer le prix ») |
| Archiver | `trash-2` | Owner, Admin | confirm inline « Archiver ce plan ? Les abonnements en cours continuent. » → `POST /plans/{id}/archive` (**API additions**) |

- **Modals/drawers/sheets opened**: DM-13.
- **States**: empty « Aucun plan. Un plan définit un montant et une fréquence (ex. 5 000 FCFA / mois). »; loading; error.
- **Events/notifications triggered**: none.

### D-28 — Détail de l'abonnement / Subscription detail
- **Route**: `/subscriptions/:id` · **Icon**: lucide `repeat` · **Access**: Tous (actions Owner, Admin, Finance) · **Purpose**: one subscription's invoice history, retry ladder and lifecycle actions.
- **Layout zones**: header (client + plan + pill statut + prochaine échéance) → **Tableau** factures (`GET /subscriptions/{id}/invoices`): période, montant, statut `open`/`paid`/`uncollectible`, tentatives (`attempt_count`) avec échelle de relance visualisée (T+0 → +6 h → +24 h → +72 h, points faits/à venir) → zone actions.
- **Data displayed**: `GET /subscriptions/{id}` (statut, `current_period_start/end`, `retry_state`); invoices; chaque tentative liée à sa charge (clic → DM-01).
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Mettre en pause / Reprendre | `circle-pause` / `circle-play` | Owner, Admin, Finance | DM-15 → `POST /subscriptions/{id}/pause` ou `/resume` |
| Annuler l'abonnement | `trash-2` | Owner, Admin | DM-14 → `POST /subscriptions/{id}/cancel` |
| Envoyer un lien de paiement | `message-circle` | Owner, Admin, Finance | relance manuelle: SMS/WhatsApp avec lien pour la facture ouverte (`POST /invoices/{id}/send_link` — **API additions**) |

- **Modals/drawers/sheets opened**: DM-14, DM-15, DM-01.
- **States**: `past_due` — bandeau warning « Paiement en retard depuis le {date}. Prochaine tentative : {date-heure}. »; `canceled` — actions masquées, bandeau ink; loading; introuvable → D-45.
- **Events/notifications triggered**: relance manuelle → SMS/WhatsApp client; pause/annulation → événements `subscription.*` webhooks.

### D-29 — Solde / Balance
- **Route**: `/balance` · **Icon**: lucide `wallet` · **Access**: Tous (approvisionnement Owner, Admin, Finance) · **Purpose**: the merchant's three balances and the full ledger of balance movements.
- **Layout zones**: 3 tuiles AmountText 40 px (Disponible · En attente · Portefeuille de paiement) → bouton « Approvisionner » → **Tableau** des mouvements → pagination.
- **Data displayed**: `GET /balance` (available, pending; portefeuille de paiement — champ en **API additions**); `GET /balance_transactions` — chaque ligne: icône par type (encaissement `hand-coins`, frais — ink, paiement sortant `send`, règlement `landmark`, ajustement, approvisionnement `plus-circle`), libellé, AmountText signé (+vert / −ink), solde après, date; lien vers l'objet source (charge/payout/settlement).
- **Inputs**: filtre type (multi-select) + période (mêmes règles que D-12).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approvisionner | `plus-circle` (primary) | Owner, Admin, Finance | ouvre DM-16 → `POST /topups` |
| Exporter le relevé | `download` | Owner, Admin, Finance, Viewer | DM-35 (CSV/PDF par période) |
| Ouvrir l'objet lié | — (row click) | Tous | DM-01 (charge) ou navigation payout/settlement |

- **Modals/drawers/sheets opened**: DM-16, DM-35, DM-01.
- **States**: loading skeleton tuiles; empty mouvements « Aucun mouvement pour cette période. »; approvisionnement en attente — ligne fantôme warning « Approvisionnement attendu — réf. {code} » jusqu'au match; error standard.
- **Events/notifications triggered**: réception d'un approvisionnement (auto-match) → push + e-mail + cloche « Approvisionnement de 500 000 FCFA reçu ».

### D-30 — Règlements / Settlements
- **Route**: `/settlements` · **Icon**: lucide `landmark` · **Access**: Tous (paramètres Owner, Admin) · **Purpose**: list of transfers of collected funds to the merchant's own account.
- **Layout zones**: header (titre + lien « Paramètres de règlement ») → **Tableau** → pagination.
- **Data displayed**: `GET /settlements` — période couverte, montant, destination (masquée: « MoMo •• 43 »), statut `paid` (success) / `in_transit` (pill warning « En cours »), date.
- **Inputs**: filtre période.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir le détail | — (row click) | Tous | → D-31 |
| Paramètres de règlement | `settings` | Owner, Admin | ouvre DM-17 (fréquence T+1/hebdo + compte destinataire; 2FA + délai 24 h) |

- **Modals/drawers/sheets opened**: DM-17.
- **States**: empty « Aucun règlement pour l'instant. Vos encaissements seront versés à J+1. »; bandeau après changement de compte: warning « Nouveau compte de règlement actif dans 24 h (sécurité). »; loading; error.
- **Events/notifications triggered**: settlement payé → push + e-mail (relevé joint) + SMS (matrice 08 §C).

### D-31 — Détail du règlement / Settlement detail
- **Route**: `/settlements/:id` · **Icon**: lucide `landmark` · **Access**: Tous · **Purpose**: composition of one settlement and its downloadable statement.
- **Layout zones**: header (montant XL + statut + destination + date) → **Tableau** des transactions incluses (TxRow condensé + frais) → pied téléchargements.
- **Data displayed**: `GET /settlements/{id}` — inclut la liste des transactions (brut, frais, net) et totaux.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Télécharger le relevé PDF | `download` | Tous | `GET /settlements/{id}/statement.pdf` (**API additions**) |
| Télécharger le CSV | `download` | Tous | `GET /settlements/{id}/statement.csv` (**API additions**) |
| Ouvrir une transaction | — (row click) | Tous | DM-01 |

- **Modals/drawers/sheets opened**: DM-01.
- **States**: `in_transit` — bandeau info « Virement en cours vers MoMo •• 43. »; loading; introuvable → D-45.
- **Events/notifications triggered**: none.

### D-32 — Clés API / API keys
- **Route**: `/developers/keys` · **Icon**: lucide `key-round` · **Access**: Owner, Admin, Developer · **Purpose**: create, reveal-once, track and revoke API keys per environment.
- **Layout zones**: bandeau docs `book-open` « Démarrez en 5 minutes → docs.ijimpay.com » → section **Clés de test** → section **Clés réelles** (chacune: clés secrètes **Tableau** + clé publiable inline avec `copy`).
- **Data displayed**: `GET /api_keys` (**API additions**) — nom, préfixe (`sk_live_a1b2…`), créée le, dernière utilisation (`last_used_at`, « Jamais » si null), état. Clés publiables affichées en clair (copiables).
- **Inputs**: aucun (nom de clé saisi dans DM-18).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer une clé secrète | `key-round` (primary, par section) | Owner, Admin, Developer | ouvre DM-18 → `POST /api_keys` (**API additions**) → révélation unique |
| Copier la clé publiable | `copy` | Owner, Admin, Developer | presse-papiers |
| Faire tourner la clé | `rotate-cw` (par ligne de clé secrète) | Owner, Admin, Developer | ouvre DM-18 (variante rotation : nom pré-rempli + fenêtre de grâce 0 / 24 h / 72 h) → `POST /api_keys/{id}/rotate` (**API additions**) → révélation unique de la nouvelle clé ; l'ancienne clé reste valide pendant la fenêtre puis est auto-révoquée (F-024) |
| Révoquer | `trash-2` | Owner, Admin, Developer | ouvre DM-19 → `DELETE /api_keys/{id}` (**API additions**) |

- **Modals/drawers/sheets opened**: DM-18, DM-19.
- **States**: clé en rotation — bandeau warning sur la ligne de l'ancienne clé « Ancienne clé expire le {date}. » (visible jusqu'à l'auto-révocation); section réelle verrouillée tant que tier 0 — « Les clés réelles seront disponibles après vérification. » + CTA `/settings/verification`; loading; error standard; permission-denied (Finance/Viewer): page masquée du menu; accès direct → écran « Accès réservé aux développeurs. Demandez au propriétaire du compte. ».
- **Events/notifications triggered**: création/révocation de clé → e-mail sécurité aux Owner/Admin + cloche.

### D-33 — Webhooks / Webhooks
- **Route**: `/developers/webhooks` · **Icon**: lucide `webhook` · **Access**: Owner, Admin, Developer · **Purpose**: manage webhook endpoints and their health.
- **Layout zones**: header (titre + « Ajouter un point de terminaison ») → **Tableau**: URL, événements abonnés (chips, « +3 » si déborde), état — `circle-check` « Sain » / `triangle-alert` « En échec depuis {durée} » / ink « Désactivé », mode test/réel.
- **Data displayed**: `GET /webhook_endpoints`.
- **Inputs**: aucun (formulaire dans DM-20).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ajouter un point de terminaison | `webhook` (primary) | Owner, Admin, Developer | ouvre DM-20 → `POST /webhook_endpoints` → révélation unique du secret (DM-21) |
| Ouvrir le détail | — (row click) | Owner, Admin, Developer | → D-34 |
| Supprimer | `trash-2` | Owner, Admin | confirm inline « Supprimer ce point de terminaison ? Les événements ne seront plus livrés. » → `DELETE /webhook_endpoints/{id}` |

- **Modals/drawers/sheets opened**: DM-20, DM-21.
- **States**: empty « Aucun webhook. Recevez les événements (paiement reçu, échec…) directement sur votre serveur. » + CTA + lien docs; endpoint auto-désactivé après 7 jours d'échecs — pill danger « Désactivé automatiquement »; loading; error.
- **Events/notifications triggered**: endpoint en échec 24 h → e-mail + push Developer + cloche (matrice 08 §C).

### D-34 — Détail du webhook / Webhook endpoint detail
- **Route**: `/developers/webhooks/:id` · **Icon**: lucide `webhook` · **Access**: Owner, Admin, Developer · **Purpose**: delivery log, redelivery and ping for one endpoint.
- **Layout zones**: header (URL + état + événements abonnés) → actions → **Tableau** des livraisons → pagination.
- **Data displayed**: `GET /webhook_endpoints/{id}`; livraisons (`webhook_deliveries` — endpoint liste en **API additions**): événement (type + id), code HTTP, tentative n°, prochaine relance (`next_retry_at`), livré le.
- **Inputs**: filtre « Échecs uniquement » (toggle).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer un ping | `webhook` | Owner, Admin, Developer | `POST /webhook_endpoints/{id}/ping` → toast avec code réponse « Ping envoyé — réponse 200 » |
| Relivrer | `rotate-cw` (par ligne) | Owner, Admin, Developer | `POST /webhook_deliveries/{id}/redeliver` (**API additions**) |
| Modifier | `pencil` | Owner, Admin, Developer | ouvre DM-20 pré-rempli |
| Régénérer le secret | `key-round` | Owner, Admin | confirm + DM-21 (nouveau secret, révélation unique) — `POST /webhook_endpoints/{id}/roll_secret` (**API additions**) |
| Réactiver | — (primary, dans le bandeau désactivé uniquement) | Owner, Admin, Developer | `POST /webhook_endpoints/{id}/enable` (**API additions**, F-027/F-058) → endpoint réactivé, toast « Point de terminaison réactivé. » + rappel « Relivrez les événements manqués depuis /developers/events (journal 90 jours). » |
| Voir l'événement | — (row click) | Owner, Admin, Developer | ouvre DM-31 |

- **Modals/drawers/sheets opened**: DM-20, DM-21, DM-31.
- **States**: en échec — bandeau warning avec dernière erreur (« Timeout après 10 s » / « HTTP 500 ») et prochaine relance; auto-désactivé (7 jours d'échecs, F-058) — bandeau danger « Point de terminaison désactivé le {date} après 7 jours d'échecs. Les événements ne sont plus livrés. » + bouton [Réactiver] (cf. Actions); loading; introuvable → D-45.
- **Events/notifications triggered**: none.

### D-35 — Événements / Events
- **Route**: `/developers/events` · **Icon**: lucide `scroll-text` · **Access**: Owner, Admin, Developer · **Purpose**: browse the immutable 90-day event log with full payloads.
- **Layout zones**: barre de filtres → **Tableau**: type (`charge.succeeded`…), objet lié (id cliquable), date → pagination. Bouton « Envoyer un événement de test » (mode test uniquement).
- **Data displayed**: `GET /events`; détail JSON via DM-31 (`GET /events/{id}`).
- **Inputs**: filtre type (select des types de 02 §2.7), période, recherche par id.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Voir le JSON | — (row click) | Owner, Admin, Developer | ouvre DM-31 |
| Envoyer un événement de test | `flask-conical` (test uniquement) | Owner, Admin, Developer | ouvre DM-22 |

- **Modals/drawers/sheets opened**: DM-31, DM-22.
- **States**: empty « Aucun événement sur cette période. »; rétention — note pied « Les événements sont conservés 90 jours. »; loading; error.
- **Events/notifications triggered**: none.

### D-36 — Journal API / API logs
- **Route**: `/developers/logs` · **Icon**: lucide `scroll-text` · **Access**: Owner, Admin, Developer · **Purpose**: inspect recent API requests for debugging.
- **Layout zones**: barre de filtres → **Tableau**: méthode, chemin, statut HTTP (pill: 2xx success / 4xx warning / 5xx danger), `request_id`, clé utilisée (préfixe), latence ms, date → pagination.
- **Data displayed**: `GET /api_logs` (**API additions**).
- **Inputs**: filtre « Erreurs uniquement (≥ 400) » toggle, période, recherche `request_id`.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Voir la requête | — (row click) | Owner, Admin, Developer | ouvre DM-32 (corps requête/réponse expurgés) |
| Copier le request_id | `copy` | Owner, Admin, Developer | presse-papiers |

- **Modals/drawers/sheets opened**: DM-32.
- **States**: empty « Aucune requête enregistrée sur cette période. »; loading; error.
- **Events/notifications triggered**: none.

### D-37 — Équipe / Team
- **Route**: `/team` · **Icon**: lucide `users-round` · **Access**: Tous (gestion Owner, Admin) · **Purpose**: manage members, roles, invitations and their security posture.
- **Layout zones**: header (titre + « Inviter ») → **Tableau** membres → section « Invitations en attente » (**Tableau**: destinataire, rôle, envoyée le, expire le).
- **Data displayed**: `GET /members` (**API additions**) — nom, téléphone/e-mail, rôle (`shield` + libellé: Propriétaire/Admin/Développeur/Finance/Lecteur), 2FA (`circle-check` « Activée » / `triangle-alert` « Non activée »), dernière activité; `GET /invites` (**API additions**).
- **Inputs**: aucun (formulaires en modals).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Inviter un membre | `user-round-plus` (primary) | Owner, Admin | ouvre DM-23 → `POST /invites` (**API additions**) |
| Changer le rôle | `shield` | Owner, Admin (Owner requis pour toucher un Admin) | ouvre DM-24 → `PATCH /members/{id}` (**API additions**) |
| Retirer | `trash-2` | Owner, Admin | ouvre DM-25 → `DELETE /members/{id}` (**API additions**); impossible sur le dernier Owner |
| Renvoyer l'invitation | `rotate-cw` | Owner, Admin | `POST /invites/{id}/resend` (**API additions**); cooldown 60 s |
| Annuler l'invitation | `trash-2` | Owner, Admin | `DELETE /invites/{id}` (**API additions**) |

- **Modals/drawers/sheets opened**: DM-23, DM-24, DM-25.
- **States**: loading; permission (Viewer/Developer/Finance): lecture seule, boutons masqués; garde-fou: tentative de retrait du dernier Owner → erreur « Impossible : un compte doit avoir au moins un Propriétaire. ».
- **Events/notifications triggered**: invitation → e-mail/SMS à l'invité; changement de rôle / arrivée → e-mail + cloche aux Owner/Admin (matrice 08 §C).

### D-38 — Paramètres · Entreprise / Business settings
- **Route**: `/settings/business` · **Icon**: lucide `settings` · **Access**: Tous (édition Owner, Admin) · **Purpose**: business profile and branding used on checkout and receipts.
- **Layout zones**: onglets Paramètres (Entreprise · Vérification · Notifications · Sécurité — navigation secondaire, chacun étant une page D-38…D-41) → formulaire profil → carte logo.
- **Data displayed**: `GET /merchant` — nom commercial, raison sociale, secteur, adresse, logo actuel (aperçu sur mini-checkout et mini-reçu).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom commercial | text | 2–80 car. | actuel | « Entrez le nom commercial. » |
| Secteur d'activité | select (liste D-07) | requis | actuel | « Choisissez un secteur. » |
| Ville / Adresse | text | mêmes règles D-07 | actuel | « Entrez l'adresse. » |
| E-mail de contact | email | RFC 5322 | actuel | « Adresse e-mail invalide. » |
| Logo | fichier | JPG/PNG carré ≥ 256 px, ≤ 1 Mo | actuel | « Logo carré, 1 Mo max. » |
| Seuil d'approbation des paiements sortants (FCFA) | number FCFA (carte « Approbation des paiements » sous le profil; PRD docs/01 §4 « configurable thresholds », référencé par D-23/F-014/F-016 « si seuil dépassé ») | entier ≥ 0 (0 = tout paiement requiert approbation) ; les paiements créés par Finance requièrent toujours l'approbation quel que soit le seuil | 500 000 | « Entrez un seuil en FCFA (0 pour tout approuver). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer | — (primary) | Owner, Admin | `PATCH /merchant` (**API additions**) ; si le seuil d'approbation a été modifié : re-prompt 2FA DM-28 obligatoire avant l'appel + e-mail sécurité aux Owner/Admin « Seuil d'approbation modifié : {ancien} → {nouveau} FCFA » |
| Supprimer le logo | `trash-2` | Owner, Admin | confirm inline « Retirer le logo ? Le checkout affichera le nom seul. » |

- **Modals/drawers/sheets opened**: DM-28 (si seuil modifié).
- **States**: loading; note: raison sociale/forme juridique verrouillées après vérification (tooltip « Modifiable via le support — donnée vérifiée »); error standard.
- **Events/notifications triggered**: none.

### D-39 — Paramètres · Vérification / Verification
- **Route**: `/settings/verification` · **Icon**: lucide `badge-check` · **Access**: Tous (upload Owner, Admin) · **Purpose**: KYB tier status and per-document lifecycle after onboarding.
- **Layout zones**: carte tier (barre Tier 0 → 1 → 2 avec limites: « Tier 1 — jusqu'à 2 000 000 FCFA/mois ») → liste documents (cartes avec statut) → historique des décisions.
- **Data displayed**: `GET /merchant` (tier, `kyb_status`); `GET /merchant/kyb_documents` (**API additions**) — par doc: type, statut (envoyé warning / approuvé success / rejeté danger + motif FR exact des ops), date.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nouveau document (re-upload) | fichier | PDF/JPG/PNG ≤ 10 Mo | — | « Fichier trop lourd (10 Mo max) ou format non accepté. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Remplacer le document | `upload` | Owner, Admin | `POST /merchant/kyb_documents` (**API additions**) — statut repasse à « envoyé » |
| Demander le Tier 2 | `badge-check` | Owner | `POST /merchant/kyb/request_tier` (**API additions**) — affiche la liste des docs additionnels requis |

- **Modals/drawers/sheets opened**: none.
- **States**: en revue — bandeau info « Documents en cours de revue (sous 24 h ouvrées). »; rejeté — carte danger avec motif; approuvé complet — success moment discret + « Vous pouvez passer en mode réel. »; loading.
- **Events/notifications triggered**: décision KYB (côté ops) → push + e-mail + SMS.

### D-40 — Paramètres · Notifications / Notification preferences
- **Route**: `/settings/notifications` · **Icon**: lucide `bell` · **Access**: Tous (préférences personnelles; ligne SMS critique verrouillée pour Viewer) · **Purpose**: per-user matrix of event × channel preferences.
- **Layout zones**: **Tableau** matrice — lignes = événements (Paiement reçu, Paiement échoué, Lot à approuver, Lot terminé, Règlement envoyé, Approvisionnement reçu, Vérification, Webhook en échec, Équipe, Sécurité, Facture d'abonnement échouée), colonnes = Push (app mobile) / E-mail / SMS — switches conformes à la matrice de docs/08 §C (les cases « — » de la matrice sont non modifiables, affichées désactivées avec tooltip « Non disponible pour cet événement »).
- **Data displayed**: `GET /me/notification_preferences` (**API additions**).
- **Inputs**: switches par cellule (défauts = matrice 08 §C); « Résumé quotidien par e-mail » switch (opt-out).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| (enregistrement automatique) | — | Tous | `PATCH /me/notification_preferences` (**API additions**) à chaque bascule; toast « Préférence enregistrée » |

- **Modals/drawers/sheets opened**: none.
- **States**: loading; erreur de sauvegarde — switch revient + toast « Impossible d'enregistrer. Réessayez. »; note sécurité: les alertes de sécurité (nouvel appareil, mot de passe) ne sont pas désactivables (switches verrouillés).
- **Events/notifications triggered**: none.

### D-41 — Paramètres · Sécurité / Security
- **Route**: `/settings/security` · **Icon**: lucide `lock` · **Access**: Tous (chacun pour son propre compte) · **Purpose**: password, TOTP, active sessions and enrolled mobile devices.
- **Layout zones**: carte mot de passe → carte 2FA (TOTP) → **Tableau** sessions actives (appareil, navigateur, IP approximative/ville, dernière activité, « Cette session ») → **Tableau** appareils mobiles inscrits (`smartphone`: modèle, inscrit le, dernier accès).
- **Data displayed**: `GET /me/sessions`, `GET /me/devices` (**API additions**); état 2FA.
- **Inputs** (changement de mot de passe):

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Mot de passe actuel | password | requis | vide | « Mot de passe actuel incorrect. » |
| Nouveau mot de passe | password + jauge | règles D-05 | vide | « 8 caractères minimum, avec au moins un chiffre. » |
| Confirmer | password | identique | vide | « Les mots de passe ne correspondent pas. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Changer le mot de passe | `lock` | Tous | `POST /me/password` (**API additions**) → révoque les autres sessions + e-mail sécurité |
| Activer la 2FA | `shield` (primary si non activée) | Tous | ouvre DM-26 (QR TOTP + code de confirmation) |
| Désactiver la 2FA | — (destructive outline) | Tous | DM-28 (re-prompt) → `DELETE /me/totp` (**API additions**); interdit si Owner/Admin d'un marchand Tier ≥ 1 — message « La 2FA est obligatoire pour votre rôle. » |
| Révoquer la session | `log-out` (par ligne) | Tous | DM-27 → `DELETE /me/sessions/{id}` (**API additions**) |
| Révoquer l'appareil | `smartphone` + `trash-2` | Tous | DM-27 → `DELETE /me/devices/{id}` (**API additions**) |

- **Modals/drawers/sheets opened**: DM-26, DM-27, DM-28.
- **States**: loading; 2FA activée — carte success « 2FA activée le {date} » + codes de secours restants (« 8/10 codes de secours disponibles » + « Régénérer »); error standard.
- **Events/notifications triggered**: mot de passe changé / 2FA modifiée / appareil révoqué → push + e-mail + SMS sécurité (matrice 08 §C).

### D-42 — Mon profil / User profile
- **Route**: `/profile` · **Icon**: lucide `user-round` · **Access**: Tous · **Purpose**: personal identity, language and merchant memberships of the signed-in user.
- **Layout zones**: carte identité (avatar initiales, nom, téléphone, e-mail) → préférence de langue → **Tableau** « Mes espaces » (marchand, rôle, rejoint le) → lien vers D-41.
- **Data displayed**: `GET /me` (**API additions**) — nom, téléphone (vérifié `circle-check`), e-mail, locale; memberships.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom complet | text | 2–60 car. | actuel | « Entrez votre nom. » |
| Adresse e-mail | email | RFC 5322; re-vérification par lien | actuel | « Adresse e-mail invalide. » |
| Langue | select Français / English | — | Français | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer | — (primary) | Tous | `PATCH /me` (**API additions**) |
| Changer de numéro | `message-square` | Tous | flux OTP double (ancien + nouveau numéro) via D-03 |
| Quitter un espace | `log-out` | Tous (sauf dernier Owner de l'espace) | confirm « Quitter {Marchand} ? Vous perdrez l'accès immédiatement. » → `DELETE /me/memberships/{id}` (**API additions**) |

- **Modals/drawers/sheets opened**: none (OTP via page D-03).
- **States**: loading; e-mail non vérifié — chip warning « Non vérifié » + « Renvoyer le lien »; dernier Owner — bouton Quitter masqué avec note « Transférez la propriété avant de quitter. ».
- **Events/notifications triggered**: changement de numéro/e-mail → e-mail + SMS sécurité.

### D-43 — Centre de notifications / Notification tray
- **Route**: overlay global (panneau 400 px ancré sous `bell`; pas d'URL propre) · **Icon**: lucide `bell` · **Access**: Tous · **Purpose**: in-dashboard feed of the bell-column events from the 08 §C matrix.
- **Layout zones**: header (« Notifications » + « Tout marquer comme lu ») → liste (icône par type, titre FR, extrait, horodatage relatif « il y a 5 min ») → pied « Voir les préférences » → D-40.
- **Data displayed**: `GET /me/notifications` (**API additions**) — type, titre, corps, lu/non-lu (point brand-600), objet lié.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir la notification | — (row click) | Tous | marque lue + navigue vers l'objet (DM-01, D-24, D-31…) |
| Tout marquer comme lu | `circle-check` | Tous | `POST /me/notifications/mark_all_read` (**API additions**) |

- **Modals/drawers/sheets opened**: none (navigue).
- **States**: empty « Rien de nouveau. Les paiements reçus, approbations et alertes apparaîtront ici. »; loading skeleton 5 lignes; temps réel: nouvelles entrées glissent en tête.
- **Events/notifications triggered**: none (consommateur).

### D-44 — Palette de commande / ⌘K palette
- **Route**: overlay global (Ctrl/⌘ K partout; pas d'URL) · **Icon**: lucide `search` · **Access**: Tous (résultats filtrés par rôle) · **Purpose**: jump to any object or action from the keyboard.
- **Layout zones**: input plein-largeur → groupes de résultats: **Actions** (Créer un lien, Encaisser, Nouveau paiement — selon rôle) · **Pages** (toutes les entrées sidebar) · **Transactions** (par réf ou téléphone) · **Liens** (par titre) · **Clients** (par nom/téléphone); navigation clavier ↑↓ + Entrée.
- **Data displayed**: recherche fédérée `GET /search?q=` (**API additions**) — débounce 250 ms, max 5 par groupe.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Recherche | text | ≥ 2 car. pour la recherche d'objets (les actions/pages filtrent dès 1 car.) | vide | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Exécuter le résultat | — (Entrée/clic) | selon résultat | navigation ou ouverture du modal correspondant (ex. « Encaisser » → DM-04) |

- **Modals/drawers/sheets opened**: DM-01, DM-04 (selon résultat).
- **States**: vide (avant saisie) — raccourcis récents + « Astuce : collez une référence de transaction. »; aucun résultat « Aucun résultat pour “{q}”. »; loading spinner inline dans l'input.
- **Events/notifications triggered**: none.

### D-45 — Page introuvable / 404
- **Route**: toute route inconnue · **Icon**: lucide `search` (illustration de la banque d'illustrations, pas une icône agrandie) · **Access**: Tous · **Purpose**: dead-end recovery.
- **Layout zones**: illustration centrée + titre + actions.
- **Data displayed**: aucune. Copy: « Page introuvable. Cette page n'existe pas ou a été déplacée. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Retour à l'accueil | `layout-dashboard` (primary) | Tous | → `/` |
| Contacter l'aide | `life-buoy` | Tous | → support |

- **Modals/drawers/sheets opened**: none. · **States**: statique. · **Events/notifications triggered**: none.

### D-46 — Erreur serveur / 500
- **Route**: rendue sur erreur applicative irrécupérable · **Icon**: lucide `triangle-alert` · **Access**: Tous · **Purpose**: fail gracefully with a support handle.
- **Layout zones**: illustration + titre + `request_id` monospace copiable.
- **Data displayed**: `request_id` de la réponse en erreur. Copy: « Une erreur est survenue de notre côté. Réessayez ; si le problème persiste, transmettez ce code au support : req_8fK2… »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer | `rotate-cw` (primary) | Tous | recharge la route courante |
| Copier le code | `copy` | Tous | presse-papiers |

- **Modals/drawers/sheets opened**: none. · **States**: statique. · **Events/notifications triggered**: erreur remontée au monitoring (interne, invisible).

### D-47 — Maintenance / Maintenance
- **Route**: servie par l'edge pendant une maintenance planifiée · **Icon**: lucide `wrench` · **Access**: Tous · **Purpose**: planned downtime message.
- **Layout zones**: illustration + titre + fenêtre horaire.
- **Data displayed**: fenêtre de maintenance (config edge). Copy: « Maintenance en cours. Ijim Pay revient vers {heure}. Les paiements en cours ne sont pas perdus. » + lien statut status.ijimpay.com.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Voir la page de statut | `activity` | Tous | → status.ijimpay.com (nouvel onglet) |

- **Modals/drawers/sheets opened**: none. · **States**: auto-refresh toutes les 60 s. · **Events/notifications triggered**: none.

### D-48 — Session expirée / Forced logout
- **Route**: interception globale sur 401 (toast + redirection `/login?reason=expired`) · **Icon**: lucide `log-out` · **Access**: Tous · **Purpose**: expel an expired/revoked session without losing work context.
- **Layout zones**: toast danger persistant « Votre session a expiré. » puis page D-01 avec bandeau info; le deep-link d'origine est mémorisé et restauré après reconnexion.
- **Data displayed**: aucune.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Se reconnecter | `log-in` (primary, dans le toast) | Tous | → `/login` (retour à la page d'origine après succès) |

- **Modals/drawers/sheets opened**: none. · **States**: formulaires en cours — brouillon conservé en localStorage 15 min (liens, paiements) et restauré avec toast « Brouillon restauré. ». · **Events/notifications triggered**: none.

### D-49 — Récupération de compte / Account recovery
- **Route**: `/account-recovery` · **Icon**: lucide `life-buoy` · **Access**: public (depuis D-04 « Je n'ai plus accès à mon numéro ») · **Purpose**: re-verify identity when the user lost phone/SIM and/or 2FA, per F-033 (docs/16) — email OTP + ID upload → ops manual review.
- **Layout zones**: centered card; 3 sous-étapes avec points de progression (E-mail → Pièce d'identité → Envoyé); retour `arrow-left` vers `/forgot-password`.
- **Data displayed**: statut de la demande en cours (`POST /auth/recovery` — API addition): `envoyée | en revue | approuvée | refusée (motif)`.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Adresse e-mail du compte | email | RFC 5322; doit être l'e-mail au dossier (réponse anti-énumération identique) | vide | « Adresse e-mail invalide. » |
| Code reçu par e-mail | 6 chiffres | 6 chiffres, expire 10 min, 5 essais | vide | « Code invalide ou expiré. » |
| Nouveau numéro de téléphone | tel `+237` | préfixe mobile CM valide | vide | « Ce numéro n'est pas un mobile camerounais valide. » |
| Pièce d'identité du dirigeant (CNI/passeport) | fichier (`upload`) | PDF/JPG/PNG ≤ 10 Mo; doit correspondre au dossier KYB | — | « Fichier trop lourd (10 Mo max) ou format non accepté (PDF, JPG, PNG). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Recevoir le code par e-mail | `message-square` | public | `POST /auth/recovery` (API addition) — étape 1; cooldown 30 s « Renvoyer (28 s) » |
| Envoyer la demande | — (primary) | public | `POST /auth/recovery` (API addition, multipart) → écran « Demande envoyée » ; la revue ops (SLA 24 h) compare aux `kyb_documents` (F-033) |
| Retour à la connexion | — (link) | public | → D-01 |

- **Modals/drawers/sheets opened**: none.
- **States**: upload en cours (barre de progression); demande envoyée / en revue — écran « **Demande en cours de revue.** Notre équipe vérifie votre identité (sous 24 h ouvrées). Vous serez prévenu par e-mail. »; refusée — carte danger « Demande refusée : {motif FR exact des ops}. » + bouton « Soumettre une nouvelle demande » (re-upload); approuvée — le numéro est mis à jour, toutes les sessions et appareils sont révoqués, la 2FA devra être réactivée à la prochaine connexion et les paiements sortants sont verrouillés 48 h (bandeau sur D-01 : « Compte récupéré — reconnectez-vous avec votre nouveau numéro. »); compte sans e-mail au dossier — écran « Contactez le support avec votre pièce d'identité » (pas de flux automatisé); rate-limited.
- **Events/notifications triggered**: décision → e-mail au demandeur; approbation → N-22 à tous les contacts Owner/Admin (F-033) + e-mail/SMS sécurité.

---

## 2. Modals & Drawers/Sheets

### DM-01 — Détail de la transaction / Transaction drawer
- **Type/anchor**: drawer droit 480 px, URL `/transactions/:id` (liste conservée derrière) · **Opened from**: D-11, D-12, D-15, D-16, D-17, D-18, D-28, D-29, D-31, D-43, D-44.
- **Contents (data)**: `GET /charges/{id}` — header AmountText XL + StatusBadge + ChannelChip; **timeline** créé → en attente → résultat (horodatages, `provider_ref`, nb de sondages); carte client (téléphone, nom, lien D-18); détail frais « Brut 5 000 FCFA · Frais −100 FCFA · Net 4 900 FCFA »; objets liés (lien de paiement, facture, remboursement); livraisons webhook de cette charge (endpoint, code, tentative) avec `rotate-cw` relivrer.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Rembourser | `undo-2` | Owner, Admin, Finance | ouvre DM-02 (seulement si `status=succeeded` et canal supporte) |
| Renvoyer le reçu | `receipt` | Owner, Admin, Finance | ouvre DM-03 |
| Copier la référence | `copy` | Tous | presse-papiers |
| Relivrer le webhook | `rotate-cw` | Owner, Admin, Developer | `POST /webhook_deliveries/{id}/redeliver` (**API additions**) |
| Fermer | `x` | Tous | retour à la liste (Échap aussi) |

- **States**: pending — timeline avec `clock` pulsant + mise à jour en direct; failed — bloc danger avec `failure_code` humanisé + doc_url; refunded — lien vers l'objet refund.
- **Confirm rules**: aucune (les confirmations vivent dans DM-02/DM-03).

### DM-02 — Rembourser / Refund modal
- **Type/anchor**: modal centré · **Opened from**: DM-01.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Montant à rembourser | number FCFA | entier, 100 ≤ montant ≤ net de la charge | net complet | « Le remboursement ne peut pas dépasser 4 900 FCFA. » |
| Motif | select: Erreur de montant / Produit indisponible / Demande du client / Autre + texte libre si Autre | requis | — | « Choisissez un motif. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer le remboursement | `undo-2` (destructive, non focus par défaut) | Owner, Admin, Finance | `POST /charges/{id}/refund` (Idempotency-Key) → toast « Remboursement lancé » |
| Annuler | — | idem | ferme |

- **Confirm rules**: le bouton restate le montant et le destinataire — « Rembourser 4 900 FCFA au 6 70 00 00 43 » (ConfirmModal, docs/07 §5).
- **States**: solde disponible insuffisant → erreur « Solde disponible insuffisant pour ce remboursement. »; canal non supporté → modal remplacé par info « Remboursement manuel requis pour ce canal » + marquage suivi.
- **Events**: webhook `refund.succeeded`; SMS reçu de remboursement au client.

### DM-03 — Renvoyer le reçu / Resend receipt
- **Type/anchor**: modal · **Opened from**: DM-01.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Canal d'envoi | radio: WhatsApp `message-circle` / SMS `message-square` / E-mail | requis | SMS | — |
| Destinataire | tel ou email selon canal | valide selon type | téléphone du client | « Numéro ou e-mail invalide. » |

- **Actions**: « Envoyer » → `POST /charges/{id}/receipt/send` (**API additions**) → toast « Reçu envoyé. » · « Annuler ».

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer | `share-2` | Owner, Admin, Finance | cf. ci-dessus |
| Annuler | — | idem | ferme |

- **Confirm rules**: aucune (non destructif). · **States**: envoi en cours; échec « Échec de l'envoi. Réessayez. ».

### DM-04 — Encaisser / Quick charge modal
- **Type/anchor**: modal centré · **Opened from**: D-11, D-17, D-18, D-44.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Montant | number FCFA | entier 100 – 5 000 000 (plafond payeur docs/14 C-02; réduit par le plafond du tier le cas échéant) | vide | « Montant entre 100 FCFA et 5 000 000 FCFA. » |
| Téléphone du client | tel `+237` | préfixe MTN/Orange; auto-détecte le canal | vide (pré-rempli depuis D-17/D-18) | « Numéro mobile money invalide. » |
| Canal | segmented MTN / Orange | requis (auto) | auto | — |
| Description (facultatif) | text | ≤ 120 car. | vide | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer la demande | `hand-coins` (primary) | Owner, Admin, Finance | `POST /charges` (Idempotency-Key) → bascule le modal en écran d'attente |
| Annuler | — | idem | ferme (confirm si champs remplis) |

- **States**: attente — `clock` pulsant + copy progressive 15 s/45 s (« Demandez au client de valider sur son téléphone… » → « Toujours en attente — le client a-t-il reçu la demande ? »); succès — success moment + « Paiement reçu — 5 000 FCFA »; échec — danger + failure_code humanisé + « Réessayer »; limite anti-spam (3 demandes en attente par payeur) → « Ce client a déjà des demandes en attente. Patientez ou annulez-les. ».
- **Confirm rules**: bouton restate implicite (montant affiché en XL au-dessus du bouton). · **Events**: webhook `charge.succeeded|failed|expired`; push mobile + cloche.

### DM-05 — QR du lien / Link QR modal
- **Type/anchor**: modal · **Opened from**: D-13, D-14, D-15.
- **Contents**: QR grand format (depuis `qr_png_url`), URL en clair, aperçu « carte comptoir » A6.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Télécharger PNG | `download` | Tous | fichier PNG 1024 px |
| Télécharger SVG | `download` | Tous | fichier SVG |
| Imprimer la carte comptoir (A6) | `printer` | Tous | fenêtre d'impression, gabarit A6 monochrome |
| Copier l'URL | `copy` | Tous | presse-papiers |

- **Confirm rules**: aucune. · **States**: QR loading shimmer.

### DM-06 — Partager le lien / Share modal
- **Type/anchor**: modal · **Opened from**: D-13, D-14, D-15.
- **Contents**: message pré-rempli FR éditable: « Bonjour ! Payez {titre} ({montant} FCFA) en toute sécurité via Ijim Pay : {url} ».
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Message | textarea | ≤ 320 car., doit contenir {url} | modèle ci-dessus | « Le message doit contenir le lien. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| WhatsApp | `message-circle` | Tous | ouvre `wa.me` avec le message |
| SMS | `message-square` | Tous | ouvre `sms:` avec le message |
| Copier le message | `copy` | Tous | presse-papiers |

- **Confirm rules**: aucune. · **States**: statique.

### DM-07 — Désactiver le lien / Deactivate link confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-13, D-15.
- **Contents**: « Désactiver “{titre}” ? Les personnes ouvrant ce lien verront “Ce lien n'est plus actif”. Les paiements déjà reçus sont conservés. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Désactiver | `trash-2` (destructive, non focus) | Owner, Admin, Finance (ou créateur du lien) | `POST /payment_links/{id}/deactivate` → toast « Lien désactivé. » |
| Annuler | — (focus par défaut) | idem | ferme |

- **Confirm rules**: destructive → bouton danger, jamais focus par défaut. · **States**: loading bouton.

### DM-08 — Approuver le lot / Approve batch modal
- **Type/anchor**: ConfirmModal + 2FA · **Opened from**: D-24.
- **Contents**: récapitulatif restaté — « Approuver le paiement de **1 450 000 FCFA** vers **12 bénéficiaires** ? » + top 3 des éléments + total frais + solde après opération.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Code 2FA | 6 chiffres TOTP | requis | vide | « Code incorrect. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approuver et payer | `check-check` (primary, non focus par défaut) | Admin, Owner (≠ créateur du lot) | `POST /payout_batches/{id}/approve` → statut `processing` → D-24 en direct |
| Annuler | — (focus par défaut) | idem | ferme |

- **Confirm rules**: restate montant + nb bénéficiaires; maker–checker: le créateur ne voit jamais ce modal; 2FA obligatoire.
- **States**: 2FA non activée → blocage « Activez la 2FA pour approuver des paiements. » + CTA D-41; solde insuffisant → bouton désactivé + bandeau danger. · **Events**: push/cloche créateur « Lot approuvé »; webhooks `payout.*`, `payout_batch.completed` en fin de traitement.

### DM-09 — Rejeter le lot / Reject batch modal
- **Type/anchor**: modal · **Opened from**: D-24.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Motif du rejet | textarea | 10–300 car. (minimum aligné sur F-018/docs/13 ≥ 10) | vide | « Expliquez le motif (10 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Rejeter le lot | — (destructive, non focus) | Admin, Owner (≠ créateur) | `POST /payout_batches/{id}/reject` (**API additions**) → retour `draft`, motif visible sur D-24 |
| Annuler | — | idem | ferme |

- **Confirm rules**: motif obligatoire. · **States**: loading. · **Events**: push + cloche au créateur « Lot rejeté : “{motif}” ».

### DM-10 — Bénéficiaire / Beneficiary create-edit drawer
- **Type/anchor**: drawer droit · **Opened from**: D-25.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom | text | 2–80 car. | vide/actuel | « Entrez le nom. » |
| Téléphone | tel `+237` | préfixe MTN/Orange valide; unicité par marchand | vide/actuel | « Ce numéro existe déjà dans vos bénéficiaires. » |
| Canal | select (auto par préfixe) | requis | auto | « Choisissez un canal. » |
| Étiquette (facultatif) | text (ex. « Fournisseur », « Employé ») | ≤ 30 car. | vide | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer | — (primary) | Owner, Admin, Finance | `POST /beneficiaries` ou `PATCH /beneficiaries/{id}` (**API additions**); lance la vérification de nom opérateur si dispo → badge `badge-check` |
| Annuler | — | idem | ferme (confirm si modifié) |

- **Confirm rules**: aucune. · **States**: vérification du nom en cours (spinner inline) → « Nom vérifié : J. FOTSO ✓ » ou warning « Nom opérateur différent : “FOTSO JEAN” ».

### DM-11 — Supprimer le bénéficiaire / Delete beneficiary confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-25.
- **Contents**: « Supprimer {nom} ({téléphone}) ? L'historique des paiements est conservé. Retiré de {n} liste(s) de paie. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Supprimer | `trash-2` (destructive, non focus) | Owner, Admin | `DELETE /beneficiaries/{id}` (**API additions**) |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: destructive standard. · **States**: loading.

### DM-12 — Réessayer l'élément / Retry payout item confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-24.
- **Contents**: « Réessayer le paiement de **250 000 FCFA** vers **J. Fotso (6 90 00 00 00)** ? Échec précédent : {failure_code humanisé}. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer | `rotate-cw` (primary, non focus) | Owner, Admin, Finance | `POST /payouts/{id}/retry` (**API additions**, Idempotency-Key) |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: restate montant + bénéficiaire (money-moving). · **States**: canal down — bouton désactivé « Canal indisponible ».

### DM-13 — Plan / Plan create-edit modal
- **Type/anchor**: modal · **Opened from**: D-27, D-26.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom du plan | text | 2–60 car. | vide/actuel | « Donnez un nom au plan. » |
| Montant | number FCFA | entier 100 – 5 000 000 (aligné C-02 docs/14); verrouillé si abonnés actifs | vide | « Montant entre 100 FCFA et 5 000 000 FCFA. » |
| Intervalle | select: Semaine / Mois / Année | requis | Mois | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer / Enregistrer | — (primary) | Owner, Admin, Finance | `POST /plans` ou `PATCH /plans/{id}` (**API additions** pour PATCH) |
| Annuler | — | idem | ferme |

- **Confirm rules**: aucune. · **States**: loading.

### DM-14 — Annuler l'abonnement / Cancel subscription confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-28.
- **Contents**: « Annuler l'abonnement de {client} au plan {plan} ({montant} FCFA / {intervalle}) ? Aucune nouvelle facture ne sera émise. Cette action est définitive. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Annuler l'abonnement | `trash-2` (destructive, non focus) | Owner, Admin | `POST /subscriptions/{id}/cancel` |
| Garder l'abonnement | — (focus) | idem | ferme |

- **Confirm rules**: destructive; restate client + plan + montant. · **Events**: webhook `subscription.canceled`.

### DM-15 — Pause / Reprise d'abonnement / Pause-resume confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-28.
- **Contents**: pause: « Mettre en pause l'abonnement de {client} ? Aucune facture pendant la pause. » / reprise: « Reprendre l'abonnement ? La prochaine facture partira le {date}. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Mettre en pause / Reprendre | `circle-pause` / `circle-play` (primary) | Owner, Admin, Finance | `POST /subscriptions/{id}/pause` ou `/resume` |
| Annuler | — | idem | ferme |

- **Confirm rules**: restate client + effet. · **States**: loading.

### DM-16 — Approvisionner / Top-up modal
- **Type/anchor**: modal · **Opened from**: D-19, D-23, D-29.
- **Contents**: deux voies (F-019). **Voie 1 — Dépôt mobile money** : instructions pas-à-pas par canal **servies par la config opérateur** (source unique côté ops — jamais codées en dur, mêmes cartes USSD que docs/09/docs/14; codes vérifiés avec les telcos, famille MTN `*126#` — ex. « Composez *126# → Transfert → {numéro Ijim Pay} → montant → référence {code} », renvoyées par `POST /topups` avec le code de référence, affiché en monospace copiable). **Voie 2 — Transfert interne** : virer du solde disponible vers le portefeuille de paiement, instantané (`POST /balance_transfers` — **API additions**, F-019 alt path).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Source | radio: Dépôt mobile money / Transfert depuis le solde disponible | requis | Dépôt mobile money | — |
| Montant prévu | number FCFA | entier ≥ 1 000; en transfert interne: ≤ solde disponible | vide | « Minimum 1 000 FCFA. » · transfert interne au-dessus du solde : « Le montant dépasse votre solde disponible ({solde} FCFA). » |
| Canal d'envoi (dépôt uniquement) | radio MTN / Orange | requis | MTN | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Générer la référence (dépôt) | `plus-circle` (primary) | Owner, Admin, Finance | `POST /topups` → affiche instructions (config opérateur) + code |
| Transférer maintenant (transfert interne) | `wallet` (primary, non focus par défaut) | Owner, Admin, Finance | confirmation restatée « Transférer {montant} FCFA du solde disponible vers le portefeuille de paiement ? » → `POST /balance_transfers` (**API additions**) — instantané, success moment + soldes D-29 mis à jour |
| Copier la référence | `copy` | idem | presse-papiers |
| Fermer | `x` | idem | ferme (le top-up reste attendu, ligne fantôme sur D-29) |

- **Confirm rules**: dépôt — aucune (l'argent bouge hors plateforme); transfert interne — money-moving: confirmation restatant le montant (docs/07 §5). · **States**: attente de réception (dépôt) — le modal peut rester ouvert et bascule en success moment à l'auto-match (push temps réel); échec du transfert interne (`insufficient_balance`) — bloc danger « Solde disponible insuffisant — le transfert n'a pas été effectué. » + montant maximal proposé. · **Events**: `balance.topup.received` webhook + push + e-mail + cloche (dépôt); écriture ledger `balance_transfer` visible sur D-29 (transfert interne).

### DM-17 — Paramètres de règlement / Settlement settings modal
- **Type/anchor**: modal + 2FA · **Opened from**: D-30.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Fréquence | radio: Quotidien (T+1) / Hebdomadaire (lundi) | requis | T+1 | — |
| Type de compte | radio MoMo / OM / Banque | requis | actuel | — |
| Numéro / RIB | tel ou RIB | mêmes règles que D-09 | actuel | « Ce numéro ne correspond pas à l'opérateur choisi. » |
| Nom du titulaire | text | doit correspondre au KYB | actuel | « Le titulaire doit correspondre à l'entreprise vérifiée. » |
| Code 2FA | 6 chiffres | requis si compte modifié | vide | « Code incorrect. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer | `landmark` (primary, non focus) | Owner, Admin | `PATCH /merchant/settlement_account` (**API additions**) → changement de compte actif après 24 h (anti-piratage), bandeau sur D-30 |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: restate nouveau compte masqué « Règlements vers MoMo •• 43 dès le {date+24 h} »; 2FA obligatoire pour changer le compte. · **Events**: e-mail + SMS sécurité « Compte de règlement modifié ».

### DM-18 — Créer une clé secrète / Create API key modal
- **Type/anchor**: modal en 2 étapes (nommer → révéler) · **Opened from**: D-32.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom de la clé | text | 2–40 car. (ex. « Serveur boutique ») | vide | « Nommez cette clé pour la retrouver. » |
| Fenêtre de grâce (variante rotation uniquement, F-024) | radio: 0 (immédiat) / 24 h / 72 h | requis en rotation | 24 h | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer la clé | `key-round` (primary) | Owner, Admin, Developer | `POST /api_keys` (**API additions**) → étape révélation: clé complète affichée UNE SEULE FOIS, monospace, bouton Copier; « Je l'ai enregistrée » ferme |
| Copier la clé | `copy` | idem | presse-papiers |
| J'ai enregistré ma clé | — (primary étape 2) | idem | ferme; la clé ne sera plus jamais affichée |

- **Confirm rules**: étape 2 non fermable par clic extérieur (évite perte de clé). · **States**: warning permanent étape 2 « Cette clé ne sera plus jamais affichée. ».

### DM-19 — Révoquer la clé / Revoke key confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-32.
- **Contents**: « Révoquer “{nom}” (sk_live_a1b2…) ? Toutes les requêtes utilisant cette clé échoueront immédiatement. Dernière utilisation : il y a 2 h. »
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Tapez RÉVOQUER pour confirmer (clés réelles utilisées < 7 jours) | text | égal à « RÉVOQUER » | vide | « Tapez RÉVOQUER pour confirmer. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Révoquer | `trash-2` (destructive, non focus) | Owner, Admin, Developer | `DELETE /api_keys/{id}` (**API additions**) |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: friction renforcée (saisie) uniquement pour clé live récemment utilisée. · **Events**: e-mail sécurité Owner/Admin.

### DM-20 — Point de terminaison webhook / Webhook endpoint drawer
- **Type/anchor**: drawer droit · **Opened from**: D-33, D-34.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| URL | url | https:// obligatoire, hôte public | vide/actuel | « URL HTTPS publique requise. » |
| Événements | multi-select à cases (types de 02 §2.7 + « Tous les événements ») | ≥ 1 | Tous | « Choisissez au moins un événement. » |
| Description (facultatif) | text | ≤ 80 car. | vide | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer / Enregistrer | — (primary) | Owner, Admin, Developer | `POST /webhook_endpoints` ou `PATCH /webhook_endpoints/{id}` (**API additions** pour PATCH) → si création: DM-21 |
| Annuler | — | idem | ferme |

- **Confirm rules**: aucune. · **States**: test d'accessibilité optionnel au submit (ping) — warning non bloquant « Votre serveur n'a pas répondu au ping. Le point de terminaison est quand même créé. ».

### DM-21 — Secret du webhook / Webhook secret reveal
- **Type/anchor**: modal (étape post-création, non fermable par clic extérieur) · **Opened from**: DM-20, D-34 (roll).
- **Contents**: secret `whsec_…` monospace, affiché une seule fois; extrait de code de vérification de signature (copiable) + lien docs.
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Copier le secret | `copy` | Owner, Admin, Developer | presse-papiers |
| J'ai enregistré le secret | — (primary) | idem | ferme définitivement |

- **Confirm rules**: même règle reveal-once que DM-18. · **States**: warning « Ce secret ne sera plus jamais affiché. ».

### DM-22 — Envoyer un événement de test / Send test event modal
- **Type/anchor**: modal (mode test uniquement) · **Opened from**: D-35.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Type d'événement | select (types 02 §2.7) | requis | charge.succeeded | — |
| Point de terminaison | select (endpoints test) | requis | premier | « Créez d'abord un point de terminaison de test. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer | `flask-conical` (primary) | Owner, Admin, Developer | `POST /test_events` (**API additions**) → toast avec code réponse |
| Annuler | — | idem | ferme |

- **Confirm rules**: aucune. · **States**: réponse affichée inline (code + latence).

### DM-23 — Inviter un membre / Invite member modal
- **Type/anchor**: modal · **Opened from**: D-37.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Téléphone ou e-mail | text | règles D-01; pas déjà membre | vide | « Cette personne est déjà membre. » |
| Rôle | radio cards avec explication: Admin « Gère tout sauf la fermeture du compte » · Développeur « Clés API, webhooks, journaux » · Finance « Encaisse, paie, exporte — crée les paiements mais ne les approuve pas » · Lecteur « Consulte tout, ne modifie rien » | requis | Lecteur | « Choisissez un rôle. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer l'invitation | `user-round-plus` (primary) | Owner, Admin (Admin ne peut pas inviter un Admin — Owner requis) | `POST /invites` (**API additions**) — expire 7 jours |
| Annuler | — | idem | ferme |

- **Confirm rules**: aucune. · **Events**: e-mail/SMS d'invitation avec lien D-06.

### DM-24 — Changer le rôle / Change role modal
- **Type/anchor**: modal · **Opened from**: D-37.
- **Inputs**: mêmes radio cards de rôle que DM-23 (+ « Propriétaire » visible uniquement pour l'Owner — transfert de propriété avec saisie du mot de passe).

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nouveau rôle | radio cards | ≠ rôle actuel | actuel | — |
| Mot de passe (si transfert de propriété) | password | requis | vide | « Mot de passe incorrect. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Changer le rôle | `shield` (primary, non focus si downgrade) | Owner, Admin (Owner pour toucher Admin/Owner) | `PATCH /members/{id}` (**API additions**) |
| Annuler | — | idem | ferme |

- **Confirm rules**: transfert de propriété — double confirmation restatée « {Nom} deviendra Propriétaire ; vous deviendrez Admin. » + mot de passe. · **Events**: e-mail aux deux parties + cloche.

### DM-25 — Retirer le membre / Remove member confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-37.
- **Contents**: « Retirer {nom} de {Marchand} ? Son accès est coupé immédiatement. Ses actions passées restent dans l'historique. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Retirer | `trash-2` (destructive, non focus) | Owner, Admin (Owner requis pour retirer un Admin) | `DELETE /members/{id}` (**API additions**); sessions révoquées |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: destructive; impossible sur le dernier Owner (bouton absent). · **Events**: e-mail au retiré + cloche Owner/Admin.

### DM-26 — Activer la 2FA / TOTP enrollment modal
- **Type/anchor**: modal 3 étapes (QR → code → codes de secours) · **Opened from**: D-41.
- **Contents**: étape 1: QR TOTP + secret manuel; étape 3: 10 codes de secours à télécharger.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Code de l'application (étape 2) | 6 chiffres | valide TOTP | vide | « Code incorrect — vérifiez l'heure de votre téléphone. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Vérifier et activer | `shield` (primary) | Tous | `POST /me/totp` puis `POST /me/totp/verify` (**API additions**) |
| Télécharger les codes de secours | `download` | Tous | fichier texte; obligatoire avant fermeture (bouton final désactivé sinon) |
| Terminer | — (primary étape 3) | Tous | ferme |

- **Confirm rules**: étape 3 non fermable tant que les codes ne sont pas téléchargés ou copiés. · **Events**: e-mail + SMS sécurité « 2FA activée ».

### DM-27 — Révoquer session/appareil / Revoke session-device confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-41.
- **Contents**: « Déconnecter {appareil/navigateur} ({ville}, dernière activité {date}) ? La personne devra se reconnecter. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Déconnecter | `log-out` (destructive, non focus) | Tous (ses propres sessions) | `DELETE /me/sessions/{id}` ou `/me/devices/{id}` (**API additions**) |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: destructive standard. · **Events**: push à l'appareil révoqué (si mobile).

### DM-28 — Confirmation 2FA / 2FA re-prompt modal
- **Type/anchor**: modal générique réutilisé par DM-08, DM-17, D-41 (désactivation 2FA) · **Opened from**: cf. appelants.
- **Contents**: « Pour continuer, confirmez avec votre code 2FA. » + rappel de l'action demandée.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Code 2FA ou code de secours | 6 chiffres ou code de secours 10 car. | valide; 5 essais | vide | « Code incorrect. {n} essais restants. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer | `shield` (primary) | selon l'appelant | `POST /auth/step_up` (**API additions**) → poursuit l'action appelante avec jeton élevé (validité 5 min) |
| Annuler | — | idem | annule l'action appelante |

- **Confirm rules**: 5 échecs → verrou 15 min + e-mail sécurité. · **States**: compte sans 2FA → remplacé par CTA d'activation (D-41).

### DM-29 — Sélecteur de marchand / Merchant switcher popover
- **Type/anchor**: popover sous le topbar · **Opened from**: chrome §0.2.
- **Contents**: liste des marchands de l'utilisateur (`GET /me` memberships): nom, rôle, tier chip; coche sur l'actuel.
- **Inputs**: filtre texte si > 5 marchands.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Changer d'espace | — (row click) | Tous | recharge le contexte marchand → `/` du marchand choisi |
| Créer un nouvel espace | `plus-circle` | Tous | flux type D-02 étape entreprise → onboarding |

- **Confirm rules**: aucune. · **States**: un seul marchand → le popover n'existe pas (nom statique).

### DM-30 — Liste de paie / Payroll list drawer
- **Type/anchor**: drawer droit large (640 px) · **Opened from**: D-25.
- **Contents**: nom de la liste + **Tableau** membres (bénéficiaire, montant par défaut) + total.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nom de la liste | text | 2–60 car. | vide/actuel | « Nommez la liste (ex. Salaires boutique). » |
| Membre (ajout) | combobox bénéficiaires | bénéficiaire existant, pas de doublon | — | « Déjà dans la liste. » |
| Montant par membre | number FCFA | entier ≥ 100 | vide | « Minimum 100 FCFA. » |
| Programmation active (section « Paie programmée », F-016) | switch | — | désactivé | — |
| Jour du mois (si programmée) | select 1–28 (28 max pour exister chaque mois) | requis si active | 28 | « Choisissez un jour entre 1 et 28. » |
| Heure (si programmée) | time picker | requis si active | 08:00 | « Choisissez une heure. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer la liste | — (primary) | Owner, Admin, Finance | `POST /payroll_lists` ou `PATCH /payroll_lists/{id}` (**API additions**); si la programmation a changé: `POST /payroll_lists/{id}/schedule` (**API additions**, API-ADD-5 docs/16). À la date programmée, le système crée un lot `draft` depuis la liste, vérifie le financement puis le passe en `pending_approval` (N-12 aux approbateurs, F-016); solde insuffisant au jour J → lot reste `draft` + N-14 « Paie non lancée — solde insuffisant » |
| Retirer un membre | `trash-2` (par ligne) | idem | retire localement (persisté au save) |
| Payer cette liste | `send` | idem | → D-23 pré-chargé |
| Supprimer la liste | `trash-2` (footer) | Owner, Admin | confirm « Supprimer la liste “{nom}” ? Les bénéficiaires sont conservés. » → `DELETE /payroll_lists/{id}` (**API additions**) |

- **Confirm rules**: suppression = destructive standard. · **States**: total recalculé en direct.

### DM-31 — Visionneuse d'événement / Event JSON viewer drawer
- **Type/anchor**: drawer droit · **Opened from**: D-34, D-35, D-43.
- **Contents**: `GET /events/{id}` — type, date, JSON pretty-print avec coloration, id de l'objet lié (lien).
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Copier le JSON | `copy` | Owner, Admin, Developer | presse-papiers |
| Ouvrir l'objet lié | — (link) | idem | DM-01 / D-24 / D-28 selon type |

- **Confirm rules**: aucune. · **States**: JSON > 100 Ko — replié par sections.

### DM-32 — Détail de requête API / API log detail drawer
- **Type/anchor**: drawer droit · **Opened from**: D-36.
- **Contents**: `GET /api_logs/{id}` (**API additions**) — méthode, chemin, statut, latence, clé (préfixe), IP, corps requête/réponse **expurgés** (secrets/PII masqués `•••`), erreur (`code`, `message`, `doc_url`).
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Copier le request_id | `copy` | Owner, Admin, Developer | presse-papiers |
| Voir la doc de l'erreur | `book-open` | idem | ouvre `doc_url` (nouvel onglet) |

- **Confirm rules**: aucune. · **States**: corps non journalisé (> 32 Ko) — note « Corps tronqué ».

### DM-33 — Nouvel abonnement / New subscription modal
- **Type/anchor**: modal · **Opened from**: D-26.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Client | combobox clients ou nouveau téléphone + nom | requis; tel valide | vide | « Choisissez un client ou saisissez un numéro valide. » |
| Plan | select plans actifs | requis | — | « Choisissez un plan. » |
| Date de début | date picker | ≥ aujourd'hui | aujourd'hui | « La date de début ne peut pas être passée. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Créer l'abonnement | `repeat` (primary) | Owner, Admin, Finance | `POST /subscriptions` → toast + ligne en tête de D-26; première facture au start_date |
| Annuler | — | idem | ferme |

- **Confirm rules**: restate « {client} paiera {montant} FCFA / {intervalle} — il devra approuver chaque prélèvement sur son téléphone. » · **Events**: SMS d'information au client (première échéance).

### DM-34 — Relancer les impayés / Bulk dunning confirm
- **Type/anchor**: ConfirmModal · **Opened from**: D-26.
- **Contents**: « Envoyer une relance à {n} client(s) en retard ? Chacun recevra un SMS/WhatsApp avec un lien de paiement pour sa facture ouverte. »
- **Inputs**: aucun.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Relancer {n} clients | `message-circle` (primary, non focus) | Owner, Admin, Finance | `POST /invoices/send_links` (bulk — **API additions**); max 1 relance manuelle / 24 h / client |
| Annuler | — (focus) | idem | ferme |

- **Confirm rules**: restate le nombre; anti-spam 24 h (clients déjà relancés exclus avec note « {m} déjà relancés aujourd'hui — exclus »). · **Events**: SMS/WhatsApp aux clients.

### DM-35 — Exporter / Export modal
- **Type/anchor**: modal · **Opened from**: D-12, D-19, D-29.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Période | date-range | ≤ 366 jours | filtres courants | « Période invalide. » |
| Format | radio CSV / PDF (PDF pour relevés uniquement) | requis | CSV | — |
| Canal | radio: Tous / MTN / Orange (relevé par rail, PRD docs/01 §6 · F-013) | requis | Tous | — |
| Colonnes | multi-select (toutes cochées) | ≥ 1 | toutes | « Choisissez au moins une colonne. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Exporter | `download` (primary) | Owner, Admin, Finance, Viewer (lecture — F-013) | ≤ 10 000 lignes: téléchargement direct (`POST /exports` — **API additions**); au-delà: job asynchrone → « Vous recevrez le fichier par e-mail d'ici quelques minutes. » |
| Annuler | — | idem | ferme |

- **Confirm rules**: aucune. · **States**: génération en cours (barre); échec « Export impossible. Réessayez. ». · **Events**: export async prêt → e-mail (lien signé 72 h, N-09) + cloche.

---

## 3. API additions needed (referenced above; absent from docs/02)

Auth & user: `POST /auth/login` · `POST /auth/login/totp` · `POST /auth/signup/otp` · `POST /auth/signup` · `POST /auth/otp` · `POST /auth/otp/verify` · `POST /auth/password/forgot` · `POST /auth/password/reset` · `POST /auth/recovery` (récupération de compte D-49, API-ADD-13 docs/16) · `POST /auth/step_up` · `GET /me` · `PATCH /me` · `POST /me/password` · `POST /me/totp` · `POST /me/totp/verify` · `DELETE /me/totp` · `GET /me/sessions` · `DELETE /me/sessions/{id}` · `GET /me/devices` · `DELETE /me/devices/{id}` · `GET /me/notifications` · `POST /me/notifications/mark_all_read` · `GET /me/notification_preferences` · `PATCH /me/notification_preferences` · `DELETE /me/memberships/{id}`.

Merchant & team: `GET /merchant` · `PATCH /merchant` · `PATCH /merchant/settlement_account` · `GET /merchant/kyb_documents` · `POST /merchant/kyb_documents` · `POST /merchant/kyb/request_tier` · `GET /members` · `PATCH /members/{id}` · `DELETE /members/{id}` · `GET /invites` · `POST /invites` · `POST /invites/{id}/resend` · `DELETE /invites/{id}` · `GET /invites/{token}` · `POST /invites/{token}/accept` · `POST /invites/{token}/decline`.

Money & objects: `GET /payout_batches` (liste) · `POST /payout_batches/validate` · `POST /payout_batches/{id}/reject` · `GET /payout_batches/{id}/report` · `POST /payouts/{id}/retry` · `GET/POST/PATCH/DELETE /beneficiaries` (+`/{id}`) · `GET/POST/PATCH/DELETE /payroll_lists` (+`/{id}`) · `POST /payroll_lists/{id}/schedule` (paie programmée, API-ADD-5 docs/16) · `POST /balance_transfers` (transfert interne disponible → portefeuille, API-ADD-7 docs/16) · `GET /plans` · `PATCH /plans/{id}` · `POST /plans/{id}/archive` · `POST /invoices/{id}/send_link` · `POST /invoices/send_links` · `PATCH /payment_links/{id}` · `GET /charges?payment_link=` (filtre) · `GET /subscriptions?customer=` (filtre) · `POST /charges/{id}/receipt/send` · `GET /customers/{id}` · `PATCH /customers/{id}` · `POST /customers/{id}/notes` · `GET /settlements/{id}/statement.pdf` · `GET /settlements/{id}/statement.csv` · champ portefeuille de paiement dans `GET /balance` · barème de frais `GET /fees` · stats accueil `GET /stats/volume` · vues de lien `GET /payment_links/{id}/stats`.

Developer & platform: `GET/POST/DELETE /api_keys` (+`/{id}`) · `POST /api_keys/{id}/rotate` (rotation avec fenêtre de grâce, F-024) · `PATCH /webhook_endpoints/{id}` · `POST /webhook_endpoints/{id}/enable` (réactivation après auto-désactivation, API-ADD-10 docs/16) · `POST /webhook_endpoints/{id}/roll_secret` · `GET /webhook_endpoints/{id}/deliveries` · `POST /webhook_deliveries/{id}/redeliver` · `GET /api_logs` · `GET /api_logs/{id}` · `POST /test_events` · `POST /exports` · `GET /search`.

## 4. Icon additions needed (absent from docs/07 §4 map)

`rotate-cw` (réessayer/relivrer — déjà utilisé dans docs/08, à officialiser) · `eye` / `eye-off` (afficher le mot de passe) · `upload` (téléverser) · `camera` (capture document mobile) · `file-up` (import CSV) · `table` (correspondance de colonnes) · `list-checks` (rapport de validation) · `calendar` (sélecteur de période) · `user-round-plus` (inviter) · `smartphone` (appareil mobile — déjà utilisé dans docs/08) · `plus-circle` est déjà au map (top-up) · `x` (fermer modal/drawer) · `wrench` (maintenance) · `log-in` (se reconnecter) · `circle-pause` / `circle-play` (pause / reprise d'abonnement — D-28/DM-15; jamais `clock`, réservé au statut Pending par docs/07 §4; déjà listés dans les ajouts d'icônes de docs/16) · `wifi-off` (hors ligne — alternative à `cloud-off` si distinction nécessaire; sinon réutiliser `cloud-off`).

---

```
INVENTORY: pages=49 tabs=7 modals=35 forms=44 tables=32 actions=227
```
