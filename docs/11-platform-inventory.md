# Ijim Pay — Inventaire de la plateforme / Platform Inventory & Master Index

Status: Draft v1 · Generated 2026-08-02 from the five final spec docs.

## 1. Purpose

This document is the **single source of truth for what exists** on the Ijim Pay platform: every page, screen, modal, sheet, flow and notification, across the four product surfaces (merchant dashboard `app.ijimpay.com`, ops console `ops.ijimpay.com`, hosted checkout `pay.ijimpay.com`, Android merchant app) plus the owned docs site `docs.ijimpay.com`. It is an **inventory and index only** — all behavioral detail lives in the five spec docs it indexes: [docs/12](12-dashboard-pages-spec.md) (dashboard), [docs/13](13-ops-console-spec.md) (ops console), [docs/14](14-checkout-pages-spec.md) (checkout), [docs/15](15-mobile-app-screens-spec.md) (mobile app) and [docs/16](16-flows-catalog.md) (end-to-end flows + message catalog). When this document and a spec doc disagree, the spec doc wins and this inventory must be regenerated. Reference docs 00–10 provide the contract (PRD, API, data model, design system); docs/12/14/15 supersede docs/08 §A, docs/09 and docs/10 respectively.

## 2. Master counts

All numbers are taken from each doc's `INVENTORY` block and re-verified against the ID sequences (D-01…D-49, DM-01…DM-35, O-01…O-24, OM-01…OM-14, C-01…C-22, CM-01…CM-06, M-01…M-51 minus the intentionally unassigned M-33, MS-01…MS-19, F-001…F-074, N-01…N-35). All sequence counts match the declared blocks.

| Doc | Surface | Pages / Écrans | Onglets (tabs) | Modales / Feuilles / Drawers | Formulaires | Tableaux | Actions |
|---|---|---|---|---|---|---|---|
| docs/12 | Dashboard marchand (`app.ijimpay.com`) | **49** (D-01…D-49) | 7 | **35** (DM-01…DM-35) | 44 | 32 | 227 |
| docs/13 | Console ops (`ops.ijimpay.com`) | **24** (O-01…O-24) | 19 | **14** (OM-01…OM-14) | 21 | 31 | 90 |
| docs/14 | Checkout hébergé (`pay.ijimpay.com`) | **22** (C-01…C-22) | 0 | **6** (CM-01…CM-06) | 4 | 2 | 54 |
| docs/15 | Application mobile (Android) | **50** (M-01…M-51, M-33 non assigné) | 3 | **19** (MS-01…MS-19) | 18 | 69 ¹ | 133 |
| **Total UI** | 4 surfaces | **145** | **29** | **74** | **87** | **134 ¹** | **504** |

¹ docs/15 counts every markdown table (69), not rendered UI data tables as docs/12/13 do — flagged [MINOR] in `docs/review/gaps-consistency.md`; the mobile app renders essentially zero data tables, so the cross-surface "Tableaux" total is not comparable.

Flows and notifications (docs/16):

| Groupe d'acteurs | Flows | IDs |
|---|---|---|
| Marchand (Merchant) | **36** | F-001…F-034, F-070, F-071 |
| Payeur (Payer) | **10** | F-035…F-044 |
| Ops | **10** | F-045…F-054 |
| Système (System) | **11** | F-055…F-065 |
| Développeur (Developer) | **7** | F-066…F-069, F-072…F-074 |
| **Total flows** | **74** | F-001…F-074 |
| **Notifications (Annexe A docs/16)** | **35** | N-01…N-35 |

**Grand total: 145 pages/écrans + 74 modales/feuilles = 219 surfaces UI spécifiées · 29 onglets · 74 flows · 35 notifications.**

## 3. Full platform sitemap

```
Ijim Pay
├── app.ijimpay.com — Dashboard marchand (docs/12)
│   ├── Public / auth
│   │   ├── /login ............................ D-01 Connexion
│   │   ├── /signup ........................... D-02 Inscription
│   │   ├── /verify-otp ....................... D-03 Vérification OTP
│   │   ├── /forgot-password .................. D-04 Mot de passe oublié
│   │   ├── /reset-password?token=… ........... D-05 Réinitialiser le mot de passe
│   │   ├── /accept-invite?token=… ............ D-06 Accepter l'invitation
│   │   └── /account-recovery ................. D-49 Récupération de compte (F-033)
│   ├── Onboarding (Owner)
│   │   ├── /onboarding/business .............. D-07 Infos entreprise (étape 1/4)
│   │   ├── /onboarding/documents ............. D-08 Documents (2/4)
│   │   ├── /onboarding/account ............... D-09 Compte de règlement (3/4)
│   │   └── /onboarding/done .................. D-10 Terminé (4/4)
│   ├── / ..................................... D-11 Accueil
│   ├── Encaissements
│   │   ├── /transactions (+/:id → DM-01) ..... D-12 Transactions
│   │   ├── /links ............................ D-13 Liens de paiement
│   │   │   ├── /links/new .................... D-14 Nouveau lien
│   │   │   └── /links/:id .................... D-15 Détail du lien
│   │   ├── /checkout-sessions/:id ............ D-16 Session de paiement
│   │   └── /customers (+/:id) ................ D-17 Clients · D-18 Fiche client
│   ├── Décaissements
│   │   ├── /payouts [Paiements|Lots|Bénéficiaires] D-19 Paiements sortants
│   │   │   ├── /payouts/new .................. D-20 Nouveau paiement
│   │   │   ├── /payouts/new/mapping .......... D-21 Correspondance des colonnes
│   │   │   ├── /payouts/new/validation ....... D-22 Rapport de validation
│   │   │   ├── /payouts/new/review ........... D-23 Révision
│   │   │   └── /payouts/batches/:id .......... D-24 Détail du lot
│   │   ├── /payouts/beneficiaries [Bénéficiaires|Listes de paie] D-25
│   │   └── /subscriptions [2 onglets] ........ D-26 Abonnements
│   │       ├── /subscriptions/plans .......... D-27 Plans
│   │       └── /subscriptions/:id ............ D-28 Détail de l'abonnement
│   ├── Finances
│   │   ├── /balance .......................... D-29 Solde
│   │   └── /settlements (+/:id) .............. D-30 Règlements · D-31 Détail du règlement
│   ├── Développeurs (Owner/Admin/Developer)
│   │   ├── /developers/keys .................. D-32 Clés API
│   │   ├── /developers/webhooks (+/:id) ...... D-33 Webhooks · D-34 Détail du webhook
│   │   ├── /developers/events ................ D-35 Événements
│   │   └── /developers/logs .................. D-36 Journal API
│   ├── /team ................................. D-37 Équipe
│   ├── Paramètres
│   │   ├── /settings/business ................ D-38 Entreprise (+ seuil d'approbation)
│   │   ├── /settings/verification ............ D-39 Vérification
│   │   ├── /settings/notifications ........... D-40 Notifications
│   │   └── /settings/security ................ D-41 Sécurité
│   ├── /profile .............................. D-42 Mon profil
│   ├── Overlays globaux ...................... D-43 Centre de notifications · D-44 Palette ⌘K
│   ├── États globaux ......................... D-45 404 · D-46 500 · D-47 Maintenance · D-48 Session expirée
│   └── Modales/drawers ....................... DM-01 … DM-35
├── ops.ijimpay.com — Console ops interne (docs/13, SSO + WebAuthn)
│   ├── /login ................................ O-01 Connexion SSO
│   ├── / ..................................... O-02 Accueil ops
│   ├── KYB: /queue/kyb (+/:reviewId) ......... O-03 File KYB · O-04 Revue KYB
│   ├── Marchands: /merchants (+/:id) ......... O-05 · O-06 Fiche marchand
│   ├── Transactions: /transactions (+/:id) ... O-07 Recherche · O-08 Détail
│   ├── Réconciliation: /reconciliation ....... O-09 (+/items/:id → O-10 Enquête d'écart)
│   ├── /providers ............................ O-11 Fournisseurs (santé)
│   ├── Risque: /risk (+/cases/:id) ........... O-12 · O-13 Dossier · O-14 Pack STR (…/str-pack)
│   ├── Grand livre: /adjustments (+/new) ..... O-15 Ajustements · O-16 Nouvel ajustement
│   ├── /approvals ............................ O-17 Approbations (double contrôle)
│   ├── /audit ................................ O-18 Journal d'audit
│   ├── /staff ................................ O-19 Personnel & rôles
│   ├── /settings ............................. O-20 Paramètres ops
│   ├── États globaux ......................... O-21 404 · O-22 500 · O-23 Maintenance · O-24 Session expirée
│   └── Modales ............................... OM-01 … OM-14
├── pay.ijimpay.com — Checkout hébergé payeur (docs/14, public)
│   ├── /l/{slug} ............................. C-01 fixe · C-02 libre · C-03 catalogue
│   │   └── états: C-05 inactif · C-06 expiré · C-07 déjà payé
│   ├── /c/{session_id} ....................... C-04 Session e-commerce
│   ├── /l|c/…#pay ............................ C-08 Numéro et opérateur
│   ├── /pay/{charge_id} ...................... C-09 attente · C-10 réussi · C-11 échoué · C-12 expiré · C-13 429
│   ├── Reçus: /r/{receipt_id} (+.pdf) ........ C-14 Reçu public · C-15 Reçu PDF/thermique
│   ├── Imprimés .............................. C-16 Affiche QR comptoir A6
│   ├── C-17 Mode dégradé navigateur intégré
│   ├── Widget ................................ C-18 chargeur · C-19 modal iframe · C-20 iframe bloqué
│   ├── États globaux ......................... C-21 404 · C-22 erreur/maintenance
│   └── Feuilles .............................. CM-01 … CM-06
├── Application mobile Android (docs/15, slugs d'écran)
│   ├── Boot/auth: splash M-01 · welcome M-02 · phone-entry M-03 · otp M-04 · create-pin M-05 · biometric-optin M-06 · login-pin M-07 · app-lock M-50
│   ├── Onboarding KYB: business-info M-08 · doc-capture M-09 · settlement-account M-10 · verification-status M-11
│   ├── Accueil: home M-12 · notifications M-13 · provider-status-detail M-14
│   ├── Encaisser: amount-pad M-15 · channel-payer M-16 · charge-waiting M-17 · charge-success M-18 · charge-failed M-19 · qr-display M-20
│   ├── Liens: links-list M-21 · link-create M-22 · link-created M-23 · link-detail M-24 · link-qr M-25
│   ├── Activité: activity-list M-26 · tx-detail M-27 · receipt-preview M-28
│   ├── Paiements sortants: payouts-home M-29 · approval-detail M-30 · approval-confirm M-31 · beneficiary-list M-32 · payout-single M-34 · topup-instructions M-35 (M-33 non assigné — le sheet bénéficiaire est MS-12)
│   ├── Menu: menu M-36 · balance-detail M-37 · settlements-list M-38 · settlement-detail M-39 · team-list M-40 · business-profile M-41 · verification-docs M-42 · security M-43 · notifications-prefs M-44 · language M-45 · help M-46 · about M-47 · printer-pairing M-51
│   ├── États bloquants: force-update M-48 · maintenance M-49
│   └── Feuilles/sheets ....................... MS-01 … MS-19
└── docs.ijimpay.com — Site de documentation (surface possédée, F-074; quickstart F-066, recettes personas, checklist go-live F-068, référence des codes d'erreur)
```

## 4. Master indexes

### 4.1 Dashboard marchand — pages (docs/12)

| ID | Nom (FR) | Route | Ancre |
|---|---|---|---|
| D-01 | Connexion | `/login` | [D-01](12-dashboard-pages-spec.md#d-01--connexion--login) |
| D-02 | Inscription | `/signup` | [D-02](12-dashboard-pages-spec.md#d-02--inscription--signup) |
| D-03 | Vérification OTP | `/verify-otp` | [D-03](12-dashboard-pages-spec.md#d-03--vérification-otp--otp-verification) |
| D-04 | Mot de passe oublié | `/forgot-password` | [D-04](12-dashboard-pages-spec.md#d-04--mot-de-passe-oublié--forgot-password) |
| D-05 | Réinitialiser le mot de passe | `/reset-password?token=…` | [D-05](12-dashboard-pages-spec.md#d-05--réinitialiser-le-mot-de-passe--reset-password) |
| D-06 | Accepter l'invitation | `/accept-invite?token=…` | [D-06](12-dashboard-pages-spec.md#d-06--accepter-linvitation--accept-invite) |
| D-07 | Onboarding · Infos entreprise | `/onboarding/business` | [D-07](12-dashboard-pages-spec.md#d-07--onboarding--infos-entreprise--business-info) |
| D-08 | Onboarding · Documents | `/onboarding/documents` | [D-08](12-dashboard-pages-spec.md#d-08--onboarding--documents--documents) |
| D-09 | Onboarding · Compte de règlement | `/onboarding/account` | [D-09](12-dashboard-pages-spec.md#d-09--onboarding--compte-de-règlement--settlement-account) |
| D-10 | Onboarding · Terminé | `/onboarding/done` | [D-10](12-dashboard-pages-spec.md#d-10--onboarding--terminé--done) |
| D-11 | Accueil | `/` | [D-11](12-dashboard-pages-spec.md#d-11--accueil--home) |
| D-12 | Transactions | `/transactions` (détail `/transactions/:id` → DM-01) | [D-12](12-dashboard-pages-spec.md#d-12--transactions--transactions) |
| D-13 | Liens de paiement | `/links` | [D-13](12-dashboard-pages-spec.md#d-13--liens-de-paiement--payment-links) |
| D-14 | Nouveau lien | `/links/new` | [D-14](12-dashboard-pages-spec.md#d-14--nouveau-lien--new-payment-link) |
| D-15 | Détail du lien | `/links/:id` | [D-15](12-dashboard-pages-spec.md#d-15--détail-du-lien--link-detail) |
| D-16 | Session de paiement | `/checkout-sessions/:id` | [D-16](12-dashboard-pages-spec.md#d-16--session-de-paiement--checkout-session-detail) |
| D-17 | Clients | `/customers` | [D-17](12-dashboard-pages-spec.md#d-17--clients--customers) |
| D-18 | Fiche client | `/customers/:id` | [D-18](12-dashboard-pages-spec.md#d-18--fiche-client--customer-detail) |
| D-19 | Paiements sortants | `/payouts` | [D-19](12-dashboard-pages-spec.md#d-19--paiements-sortants--payouts) |
| D-20 | Nouveau paiement | `/payouts/new` | [D-20](12-dashboard-pages-spec.md#d-20--nouveau-paiement--new-payout) |
| D-21 | Paiement en masse · Correspondance des colonnes | `/payouts/new/mapping` | [D-21](12-dashboard-pages-spec.md#d-21--paiement-en-masse--correspondance-des-colonnes--csv-column-mapper) |
| D-22 | Paiement en masse · Rapport de validation | `/payouts/new/validation` | [D-22](12-dashboard-pages-spec.md#d-22--paiement-en-masse--rapport-de-validation--validation-report) |
| D-23 | Paiement · Révision | `/payouts/new/review` | [D-23](12-dashboard-pages-spec.md#d-23--paiement--révision--payout-review) |
| D-24 | Détail du lot | `/payouts/batches/:id` | [D-24](12-dashboard-pages-spec.md#d-24--détail-du-lot--payout-batch-detail) |
| D-25 | Bénéficiaires | `/payouts/beneficiaries` | [D-25](12-dashboard-pages-spec.md#d-25--bénéficiaires--beneficiaries) |
| D-26 | Abonnements | `/subscriptions` | [D-26](12-dashboard-pages-spec.md#d-26--abonnements--subscriptions) |
| D-27 | Plans | `/subscriptions/plans` | [D-27](12-dashboard-pages-spec.md#d-27--plans--plans) |
| D-28 | Détail de l'abonnement | `/subscriptions/:id` | [D-28](12-dashboard-pages-spec.md#d-28--détail-de-labonnement--subscription-detail) |
| D-29 | Solde | `/balance` | [D-29](12-dashboard-pages-spec.md#d-29--solde--balance) |
| D-30 | Règlements | `/settlements` | [D-30](12-dashboard-pages-spec.md#d-30--règlements--settlements) |
| D-31 | Détail du règlement | `/settlements/:id` | [D-31](12-dashboard-pages-spec.md#d-31--détail-du-règlement--settlement-detail) |
| D-32 | Clés API | `/developers/keys` | [D-32](12-dashboard-pages-spec.md#d-32--clés-api--api-keys) |
| D-33 | Webhooks | `/developers/webhooks` | [D-33](12-dashboard-pages-spec.md#d-33--webhooks--webhooks) |
| D-34 | Détail du webhook | `/developers/webhooks/:id` | [D-34](12-dashboard-pages-spec.md#d-34--détail-du-webhook--webhook-endpoint-detail) |
| D-35 | Événements | `/developers/events` | [D-35](12-dashboard-pages-spec.md#d-35--événements--events) |
| D-36 | Journal API | `/developers/logs` | [D-36](12-dashboard-pages-spec.md#d-36--journal-api--api-logs) |
| D-37 | Équipe | `/team` | [D-37](12-dashboard-pages-spec.md#d-37--équipe--team) |
| D-38 | Paramètres · Entreprise | `/settings/business` | [D-38](12-dashboard-pages-spec.md#d-38--paramètres--entreprise--business-settings) |
| D-39 | Paramètres · Vérification | `/settings/verification` | [D-39](12-dashboard-pages-spec.md#d-39--paramètres--vérification--verification) |
| D-40 | Paramètres · Notifications | `/settings/notifications` | [D-40](12-dashboard-pages-spec.md#d-40--paramètres--notifications--notification-preferences) |
| D-41 | Paramètres · Sécurité | `/settings/security` | [D-41](12-dashboard-pages-spec.md#d-41--paramètres--sécurité--security) |
| D-42 | Mon profil | `/profile` | [D-42](12-dashboard-pages-spec.md#d-42--mon-profil--user-profile) |
| D-43 | Centre de notifications | overlay global (sous `bell`, pas d'URL) | [D-43](12-dashboard-pages-spec.md#d-43--centre-de-notifications--notification-tray) |
| D-44 | Palette de commande | overlay global (Ctrl/⌘ K, pas d'URL) | [D-44](12-dashboard-pages-spec.md#d-44--palette-de-commande--k-palette) |
| D-45 | Page introuvable | toute route inconnue | [D-45](12-dashboard-pages-spec.md#d-45--page-introuvable--404) |
| D-46 | Erreur serveur | erreur applicative irrécupérable | [D-46](12-dashboard-pages-spec.md#d-46--erreur-serveur--500) |
| D-47 | Maintenance | servie par l'edge | [D-47](12-dashboard-pages-spec.md#d-47--maintenance--maintenance) |
| D-48 | Session expirée | interception 401 → `/login?reason=expired` | [D-48](12-dashboard-pages-spec.md#d-48--session-expirée--forced-logout) |
| D-49 | Récupération de compte | `/account-recovery` | [D-49](12-dashboard-pages-spec.md#d-49--récupération-de-compte--account-recovery) |

Modales/drawers dashboard (type entre parenthèses lorsqu'utile) :

| ID | Nom (FR) | Ancre |
|---|---|---|
| DM-01 | Détail de la transaction (drawer) | [DM-01](12-dashboard-pages-spec.md#dm-01--détail-de-la-transaction--transaction-drawer) |
| DM-02 | Rembourser | [DM-02](12-dashboard-pages-spec.md#dm-02--rembourser--refund-modal) |
| DM-03 | Renvoyer le reçu | [DM-03](12-dashboard-pages-spec.md#dm-03--renvoyer-le-reçu--resend-receipt) |
| DM-04 | Encaisser | [DM-04](12-dashboard-pages-spec.md#dm-04--encaisser--quick-charge-modal) |
| DM-05 | QR du lien | [DM-05](12-dashboard-pages-spec.md#dm-05--qr-du-lien--link-qr-modal) |
| DM-06 | Partager le lien | [DM-06](12-dashboard-pages-spec.md#dm-06--partager-le-lien--share-modal) |
| DM-07 | Désactiver le lien | [DM-07](12-dashboard-pages-spec.md#dm-07--désactiver-le-lien--deactivate-link-confirm) |
| DM-08 | Approuver le lot | [DM-08](12-dashboard-pages-spec.md#dm-08--approuver-le-lot--approve-batch-modal) |
| DM-09 | Rejeter le lot | [DM-09](12-dashboard-pages-spec.md#dm-09--rejeter-le-lot--reject-batch-modal) |
| DM-10 | Bénéficiaire (drawer création/édition) | [DM-10](12-dashboard-pages-spec.md#dm-10--bénéficiaire--beneficiary-create-edit-drawer) |
| DM-11 | Supprimer le bénéficiaire | [DM-11](12-dashboard-pages-spec.md#dm-11--supprimer-le-bénéficiaire--delete-beneficiary-confirm) |
| DM-12 | Réessayer l'élément | [DM-12](12-dashboard-pages-spec.md#dm-12--réessayer-lélément--retry-payout-item-confirm) |
| DM-13 | Plan (création/édition) | [DM-13](12-dashboard-pages-spec.md#dm-13--plan--plan-create-edit-modal) |
| DM-14 | Annuler l'abonnement | [DM-14](12-dashboard-pages-spec.md#dm-14--annuler-labonnement--cancel-subscription-confirm) |
| DM-15 | Pause / Reprise d'abonnement | [DM-15](12-dashboard-pages-spec.md#dm-15--pause--reprise-dabonnement--pause-resume-confirm) |
| DM-16 | Approvisionner | [DM-16](12-dashboard-pages-spec.md#dm-16--approvisionner--top-up-modal) |
| DM-17 | Paramètres de règlement | [DM-17](12-dashboard-pages-spec.md#dm-17--paramètres-de-règlement--settlement-settings-modal) |
| DM-18 | Créer une clé secrète | [DM-18](12-dashboard-pages-spec.md#dm-18--créer-une-clé-secrète--create-api-key-modal) |
| DM-19 | Révoquer la clé | [DM-19](12-dashboard-pages-spec.md#dm-19--révoquer-la-clé--revoke-key-confirm) |
| DM-20 | Point de terminaison webhook (drawer) | [DM-20](12-dashboard-pages-spec.md#dm-20--point-de-terminaison-webhook--webhook-endpoint-drawer) |
| DM-21 | Secret du webhook | [DM-21](12-dashboard-pages-spec.md#dm-21--secret-du-webhook--webhook-secret-reveal) |
| DM-22 | Envoyer un événement de test | [DM-22](12-dashboard-pages-spec.md#dm-22--envoyer-un-événement-de-test--send-test-event-modal) |
| DM-23 | Inviter un membre | [DM-23](12-dashboard-pages-spec.md#dm-23--inviter-un-membre--invite-member-modal) |
| DM-24 | Changer le rôle | [DM-24](12-dashboard-pages-spec.md#dm-24--changer-le-rôle--change-role-modal) |
| DM-25 | Retirer le membre | [DM-25](12-dashboard-pages-spec.md#dm-25--retirer-le-membre--remove-member-confirm) |
| DM-26 | Activer la 2FA (TOTP) | [DM-26](12-dashboard-pages-spec.md#dm-26--activer-la-2fa--totp-enrollment-modal) |
| DM-27 | Révoquer session/appareil | [DM-27](12-dashboard-pages-spec.md#dm-27--révoquer-sessionappareil--revoke-session-device-confirm) |
| DM-28 | Confirmation 2FA (re-prompt) | [DM-28](12-dashboard-pages-spec.md#dm-28--confirmation-2fa--2fa-re-prompt-modal) |
| DM-29 | Sélecteur de marchand (popover) | [DM-29](12-dashboard-pages-spec.md#dm-29--sélecteur-de-marchand--merchant-switcher-popover) |
| DM-30 | Liste de paie (drawer, avec programmation) | [DM-30](12-dashboard-pages-spec.md#dm-30--liste-de-paie--payroll-list-drawer) |
| DM-31 | Visionneuse d'événement (drawer JSON) | [DM-31](12-dashboard-pages-spec.md#dm-31--visionneuse-dévénement--event-json-viewer-drawer) |
| DM-32 | Détail de requête API (drawer) | [DM-32](12-dashboard-pages-spec.md#dm-32--détail-de-requête-api--api-log-detail-drawer) |
| DM-33 | Nouvel abonnement | [DM-33](12-dashboard-pages-spec.md#dm-33--nouvel-abonnement--new-subscription-modal) |
| DM-34 | Relancer les impayés | [DM-34](12-dashboard-pages-spec.md#dm-34--relancer-les-impayés--bulk-dunning-confirm) |
| DM-35 | Exporter | [DM-35](12-dashboard-pages-spec.md#dm-35--exporter--export-modal) |

### 4.2 Console ops — pages (docs/13)

| ID | Nom (FR) | Route | Ancre |
|---|---|---|---|
| O-01 | Connexion SSO | `/login` | [O-01](13-ops-console-spec.md#o-01--connexion-sso--sso-login) |
| O-02 | Accueil ops | `/` | [O-02](13-ops-console-spec.md#o-02--accueil-ops--ops-home) |
| O-03 | File KYB | `/queue/kyb` | [O-03](13-ops-console-spec.md#o-03--file-kyb--kyb-queue) |
| O-04 | Revue KYB | `/queue/kyb/:reviewId` | [O-04](13-ops-console-spec.md#o-04--revue-kyb--kyb-review-detail) |
| O-05 | Marchands | `/merchants` | [O-05](13-ops-console-spec.md#o-05--marchands--merchants) |
| O-06 | Fiche marchand | `/merchants/:id` | [O-06](13-ops-console-spec.md#o-06--fiche-marchand--merchant-detail) |
| O-07 | Recherche transactions | `/transactions` | [O-07](13-ops-console-spec.md#o-07--recherche-transactions--transactions-search) |
| O-08 | Détail transaction | `/transactions/:id` | [O-08](13-ops-console-spec.md#o-08--détail-transaction--transaction-detail) |
| O-09 | Réconciliation | `/reconciliation` | [O-09](13-ops-console-spec.md#o-09--réconciliation--reconciliation) |
| O-10 | Enquête d'écart | `/reconciliation/items/:id` | [O-10](13-ops-console-spec.md#o-10--enquête-décart--discrepancy-investigation) |
| O-11 | Fournisseurs | `/providers` | [O-11](13-ops-console-spec.md#o-11--fournisseurs--providers-health) |
| O-12 | Risque | `/risk` | [O-12](13-ops-console-spec.md#o-12--risque--risk) |
| O-13 | Dossier de risque | `/risk/cases/:id` | [O-13](13-ops-console-spec.md#o-13--dossier-de-risque--risk-case-detail) |
| O-14 | Pack d'export STR | `/risk/cases/:id/str-pack` | [O-14](13-ops-console-spec.md#o-14--pack-dexport-str--str-export-pack-builder) |
| O-15 | Ajustements | `/adjustments` | [O-15](13-ops-console-spec.md#o-15--ajustements--adjustments) |
| O-16 | Nouvel ajustement | `/adjustments/new` | [O-16](13-ops-console-spec.md#o-16--nouvel-ajustement--adjustment-creation) |
| O-17 | Approbations | `/approvals` | [O-17](13-ops-console-spec.md#o-17--approbations--dual-control-approvals) |
| O-18 | Journal d'audit | `/audit` | [O-18](13-ops-console-spec.md#o-18--journal-daudit--audit-log) |
| O-19 | Personnel & rôles | `/staff` | [O-19](13-ops-console-spec.md#o-19--personnel--rôles--staff--roles-admin) |
| O-20 | Paramètres ops | `/settings` | [O-20](13-ops-console-spec.md#o-20--paramètres-ops--ops-settings) |
| O-21 | Page introuvable | toute route inconnue | [O-21](13-ops-console-spec.md#o-21--page-introuvable--404) |
| O-22 | Erreur serveur | erreur irrécupérable | [O-22](13-ops-console-spec.md#o-22--erreur-serveur--500) |
| O-23 | Maintenance | servie par l'edge | [O-23](13-ops-console-spec.md#o-23--maintenance--maintenance) |
| O-24 | Session expirée | interception 401 | [O-24](13-ops-console-spec.md#o-24--session-expirée--forced-logout) |

Modales ops :

| ID | Nom (FR) | Ancre |
|---|---|---|
| OM-01 | Rejeter le dossier KYB (bibliothèque de motifs) | [OM-01](13-ops-console-spec.md#om-01--rejeter-le-dossier-kyb--kyb-rejection-reason-template-library) |
| OM-02 | Approuver le palier | [OM-02](13-ops-console-spec.md#om-02--approuver-le-palier--approve-tier) |
| OM-03 | Demander des informations (KYB) | [OM-03](13-ops-console-spec.md#om-03--demander-des-informations--request-more-info-kyb) |
| OM-04 | Modifier les limites (double contrôle) | [OM-04](13-ops-console-spec.md#om-04--modifier-les-limites--limit-change-dual-control) |
| OM-05 | Suspendre / Réactiver le marchand (DC) | [OM-05](13-ops-console-spec.md#om-05--suspendre--réactiver-le-marchand--suspendreactivate-dual-control) |
| OM-06 | Impersonation lecture seule | [OM-06](13-ops-console-spec.md#om-06--impersonation-lecture-seule--read-only-impersonation) |
| OM-07 | Re-interroger le provider | [OM-07](13-ops-console-spec.md#om-07--re-interroger-le-provider--manual-re-poll) |
| OM-08 | Confirmer la résolution d'écart | [OM-08](13-ops-console-spec.md#om-08--confirmer-la-résolution-décart--discrepancy-resolution-confirm) |
| OM-09 | Disjoncteur provider (DC) | [OM-09](13-ops-console-spec.md#om-09--disjoncteur-provider--circuit-breaker-dual-control) |
| OM-10 | Aperçu d'écriture comptable | [OM-10](13-ops-console-spec.md#om-10--aperçu-décriture-comptable--journal-entry-preview) |
| OM-11 | Décision de seconde approbation | [OM-11](13-ops-console-spec.md#om-11--décision-de-seconde-approbation--second-approver-decision) |
| OM-12 | Inviter / modifier un membre du personnel | [OM-12](13-ops-console-spec.md#om-12--inviter--modifier-un-membre-du-personnel--staff-inviteedit) |
| OM-13 | Importer un rapport provider | [OM-13](13-ops-console-spec.md#om-13--importer-un-rapport-provider--provider-report-import) |
| OM-14 | Ajouter une note (pièce jointe) | [OM-14](13-ops-console-spec.md#om-14--ajouter-une-note--add-note-with-attachment) |

### 4.3 Checkout hébergé — écrans (docs/14)

| ID | Nom (FR) | Route / condition | Ancre |
|---|---|---|---|
| C-01 | Lien de paiement, montant fixe | `/l/{slug}` (fixe, actif) | [C-01](14-checkout-pages-spec.md#c-01--lien-de-paiement-montant-fixe--payment-link-landing-fixed-amount) |
| C-02 | Lien de paiement, montant libre | `/l/{slug}` (libre) | [C-02](14-checkout-pages-spec.md#c-02--lien-de-paiement-montant-libre--payment-link-landing-open-amount) |
| C-03 | Lien catalogue avec quantité | `/l/{slug}` (catalogue) | [C-03](14-checkout-pages-spec.md#c-03--lien-catalogue-avec-quantité--catalog-link-with-quantity) |
| C-04 | Session de paiement e-commerce | `/c/{session_id}` | [C-04](14-checkout-pages-spec.md#c-04--session-de-paiement-e-commerce--checkout-session-landing) |
| C-05 | Lien inactif | `/l/{slug}` (`active=false`) | [C-05](14-checkout-pages-spec.md#c-05--lien-inactif--inactive-deactivated-link) |
| C-06 | Lien ou session expiré | `/l/{slug}` ou `/c/{id}` expiré | [C-06](14-checkout-pages-spec.md#c-06--lien-ou-session-expiré--expired-link-or-session) |
| C-07 | Lien déjà payé | `/l/{slug}` (usage unique consommé) | [C-07](14-checkout-pages-spec.md#c-07--lien-déjà-payé--single-use-link-already-paid) |
| C-08 | Numéro et opérateur | `/l/{slug}#pay` ou `/c/{id}#pay` | [C-08](14-checkout-pages-spec.md#c-08--numéro-et-opérateur--phone--channel) |
| C-09 | En attente d'approbation | `/pay/{charge_id}` (`pending`) | [C-09](14-checkout-pages-spec.md#c-09--en-attente-dapprobation--waiting-for-approval) |
| C-10 | Paiement réussi | `/pay/{charge_id}` (`succeeded`) | [C-10](14-checkout-pages-spec.md#c-10--paiement-réussi--success) |
| C-11 | Paiement échoué | `/pay/{charge_id}` (`failed`) | [C-11](14-checkout-pages-spec.md#c-11--paiement-échoué--failure) |
| C-12 | Demande expirée | `/pay/{charge_id}` (`expired`) | [C-12](14-checkout-pages-spec.md#c-12--demande-expirée--charge-expired) |
| C-13 | Trop de demandes | à la place de C-09 sur 429 `rate_limited` | [C-13](14-checkout-pages-spec.md#c-13--trop-de-demandes--rate-limited) |
| C-14 | Reçu public | `/r/{receipt_id}` (sans JS) | [C-14](14-checkout-pages-spec.md#c-14--reçu-public--public-receipt) |
| C-15 | Reçu PDF & impression thermique | `/r/{receipt_id}.pdf` + gabarit 80 mm | [C-15](14-checkout-pages-spec.md#c-15--reçu-pdf--impression-thermique--receipt-pdf--thermal-print-layout) |
| C-16 | Affiche QR comptoir A6 | gabarit imprimable (`qr_png_url` + PDF A6) | [C-16](14-checkout-pages-spec.md#c-16--affiche-qr-comptoir-a6--static-counter-qr--a6-print-layout) |
| C-17 | Mode dégradé navigateur intégré | webviews FB/IG/WhatsApp dégradés | [C-17](14-checkout-pages-spec.md#c-17--mode-dégradé-navigateur-intégré--in-app-browser-degraded-mode) |
| C-18 | Widget : chargeur | script `@ijimpay/checkout` ≤ 30 KB | [C-18](14-checkout-pages-spec.md#c-18--widget--chargeur--widget-loader) |
| C-19 | Widget : modal iframe | `/c/{session_id}?widget=1` en overlay | [C-19](14-checkout-pages-spec.md#c-19--widget--modal-iframe--widget-modal) |
| C-20 | Widget : iframe bloqué | fallback redirection du chargeur | [C-20](14-checkout-pages-spec.md#c-20--widget--iframe-bloqué--widget-blocked-iframe-fallback) |
| C-21 | Page introuvable | slug/id inconnu | [C-21](14-checkout-pages-spec.md#c-21--page-introuvable--not-found-404) |
| C-22 | Erreur ou maintenance | 5xx / fenêtre de maintenance | [C-22](14-checkout-pages-spec.md#c-22--erreur-ou-maintenance--server-error--maintenance) |

Feuilles checkout : CM-01 Changer d'opérateur · CM-02 Changer de numéro · CM-03 Renvoyer la demande · CM-04 Annuler le paiement (sessions) · CM-05 Contacter le commerçant · CM-06 Détail de la commande — ancres : [CM-01](14-checkout-pages-spec.md#cm-01--changer-dopérateur--change-channel-sheet) · [CM-02](14-checkout-pages-spec.md#cm-02--changer-de-numéro--change-number-sheet) · [CM-03](14-checkout-pages-spec.md#cm-03--renvoyer-la-demande--resend-request-confirm) · [CM-04](14-checkout-pages-spec.md#cm-04--annuler-le-paiement--cancel-payment-confirm-sessions) · [CM-05](14-checkout-pages-spec.md#cm-05--contacter-le-commerçant--contact-merchant-sheet) · [CM-06](14-checkout-pages-spec.md#cm-06--détail-de-la-commande--order-details-sheet).

### 4.4 Application mobile — écrans (docs/15)

| ID | Nom (FR) | Slug | Ancre |
|---|---|---|---|
| M-01 | Écran de démarrage | `splash` | [M-01](15-mobile-app-screens-spec.md#m-01--écran-de-démarrage--splash) |
| M-02 | Bienvenue | `welcome` | [M-02](15-mobile-app-screens-spec.md#m-02--bienvenue--welcome) |
| M-03 | Numéro de téléphone | `phone-entry` | [M-03](15-mobile-app-screens-spec.md#m-03--numéro-de-téléphone--phone-entry) |
| M-04 | Code de vérification | `otp` | [M-04](15-mobile-app-screens-spec.md#m-04--code-de-vérification--otp) |
| M-05 | Créer votre PIN | `create-pin` | [M-05](15-mobile-app-screens-spec.md#m-05--créer-votre-pin--create-pin) |
| M-06 | Activer la biométrie | `biometric-optin` | [M-06](15-mobile-app-screens-spec.md#m-06--activer-la-biométrie--biometric-opt-in) |
| M-07 | Connexion | `login-pin` | [M-07](15-mobile-app-screens-spec.md#m-07--connexion--login-returning) |
| M-08 | Votre entreprise | `business-info` | [M-08](15-mobile-app-screens-spec.md#m-08--votre-entreprise--business-info) |
| M-09 | Capture des documents | `doc-capture` (boucle par document) | [M-09](15-mobile-app-screens-spec.md#m-09--capture-des-documents--doc-capture-per-doc-loop) |
| M-10 | Compte de règlement | `settlement-account` | [M-10](15-mobile-app-screens-spec.md#m-10--compte-de-règlement--settlement-account) |
| M-11 | Statut de vérification | `verification-status` | [M-11](15-mobile-app-screens-spec.md#m-11--statut-de-vérification--verification-status) |
| M-12 | Accueil | `home` | [M-12](15-mobile-app-screens-spec.md#m-12--accueil--home) |
| M-13 | Notifications | `notifications` | [M-13](15-mobile-app-screens-spec.md#m-13--notifications) |
| M-14 | État des opérateurs | `provider-status-detail` | [M-14](15-mobile-app-screens-spec.md#m-14--état-des-opérateurs--provider-status-detail) |
| M-15 | Montant | `amount-pad` | [M-15](15-mobile-app-screens-spec.md#m-15--montant--amount-pad) |
| M-16 | Canal & payeur | `channel-payer` | [M-16](15-mobile-app-screens-spec.md#m-16--canal--payeur--channel--payer) |
| M-17 | En attente du client | `charge-waiting` | [M-17](15-mobile-app-screens-spec.md#m-17--en-attente-du-client--charge-waiting) |
| M-18 | Paiement reçu | `charge-success` | [M-18](15-mobile-app-screens-spec.md#m-18--paiement-reçu--charge-success) |
| M-19 | Paiement échoué | `charge-failed` | [M-19](15-mobile-app-screens-spec.md#m-19--paiement-échoué--charge-failed) |
| M-20 | QR de paiement | `qr-display` | [M-20](15-mobile-app-screens-spec.md#m-20--qr-de-paiement--qr-display) |
| M-21 | Liens de paiement | `links-list` | [M-21](15-mobile-app-screens-spec.md#m-21--liens-de-paiement--links-list) |
| M-22 | Créer un lien | `link-create` | [M-22](15-mobile-app-screens-spec.md#m-22--créer-un-lien--link-create-bottom-sheet-single-screen) |
| M-23 | Lien créé | `link-created` | [M-23](15-mobile-app-screens-spec.md#m-23--lien-créé--link-created-share) |
| M-24 | Détail du lien | `link-detail` | [M-24](15-mobile-app-screens-spec.md#m-24--détail-du-lien--link-detail) |
| M-25 | QR du lien | `link-qr` | [M-25](15-mobile-app-screens-spec.md#m-25--qr-du-lien--link-qr) |
| M-26 | Activité | `activity-list` | [M-26](15-mobile-app-screens-spec.md#m-26--activité--activity-list) |
| M-27 | Détail de la transaction | `tx-detail` | [M-27](15-mobile-app-screens-spec.md#m-27--détail-de-la-transaction--tx-detail-bottom-sheet-full-height) |
| M-28 | Reçu | `receipt-preview` | [M-28](15-mobile-app-screens-spec.md#m-28--reçu--receipt-preview--share) |
| M-29 | Paiements sortants | `payouts-home` | [M-29](15-mobile-app-screens-spec.md#m-29--paiements-sortants--payouts-home) |
| M-30 | Validation du lot | `approval-detail` | [M-30](15-mobile-app-screens-spec.md#m-30--validation-du-lot--approval-detail) |
| M-31 | Confirmer l'approbation | `approval-confirm` | [M-31](15-mobile-app-screens-spec.md#m-31--confirmer-lapprobation--approval-confirm) |
| M-32 | Bénéficiaires | `beneficiary-list` | [M-32](15-mobile-app-screens-spec.md#m-32--bénéficiaires--beneficiary-list) |
| M-33 | *(non assigné — le sheet bénéficiaire de docs/10 est MS-12)* | — | — |
| M-34 | Paiement individuel | `payout-single` | [M-34](15-mobile-app-screens-spec.md#m-34--paiement-individuel--single-payout) |
| M-35 | Approvisionner | `topup-instructions` | [M-35](15-mobile-app-screens-spec.md#m-35--approvisionner--top-up-instructions) |
| M-36 | Menu | `menu` | [M-36](15-mobile-app-screens-spec.md#m-36--menu) |
| M-37 | Solde | `balance-detail` | [M-37](15-mobile-app-screens-spec.md#m-37--solde--balance-detail) |
| M-38 | Règlements | `settlements-list` | [M-38](15-mobile-app-screens-spec.md#m-38--règlements--settlements-list) |
| M-39 | Détail du règlement | `settlement-detail` | [M-39](15-mobile-app-screens-spec.md#m-39--détail-du-règlement--settlement-detail) |
| M-40 | Équipe | `team-list` | [M-40](15-mobile-app-screens-spec.md#m-40--équipe--team-list) |
| M-41 | Profil de l'entreprise | `business-profile` | [M-41](15-mobile-app-screens-spec.md#m-41--profil-de-lentreprise--business-profile) |
| M-42 | Documents de vérification | `verification-docs` | [M-42](15-mobile-app-screens-spec.md#m-42--documents-de-vérification--verification-docs) |
| M-43 | Sécurité | `security` | [M-43](15-mobile-app-screens-spec.md#m-43--sécurité--security) |
| M-44 | Préférences de notification | `notifications-prefs` | [M-44](15-mobile-app-screens-spec.md#m-44--préférences-de-notification--notification-prefs) |
| M-45 | Langue | `language` | [M-45](15-mobile-app-screens-spec.md#m-45--langue--language) |
| M-46 | Aide & support | `help` | [M-46](15-mobile-app-screens-spec.md#m-46--aide--support--help) |
| M-47 | À propos | `about` | [M-47](15-mobile-app-screens-spec.md#m-47--à-propos--about) |
| M-48 | Mise à jour requise | `force-update` | [M-48](15-mobile-app-screens-spec.md#m-48--mise-à-jour-requise--force-update) |
| M-49 | Maintenance | `maintenance` | [M-49](15-mobile-app-screens-spec.md#m-49--maintenance) |
| M-50 | Verrouillage | `app-lock` | [M-50](15-mobile-app-screens-spec.md#m-50--verrouillage--app-lock) |
| M-51 | Imprimante (feature flag `pos_printer`, v1.2) | `printer-pairing` | [M-51](15-mobile-app-screens-spec.md#m-51--imprimante--printer-pairing-feature-flag-pos_printer-v12) |

Feuilles mobiles : MS-01 Feuille d'erreur · MS-02 Autorisation notifications · MS-03 Autorisation caméra · MS-04 Autorisation contacts · MS-05 Partager · MS-06 Désactiver le lien · MS-07 Rembourser · MS-08 Confirmation PIN/biométrie · MS-09 Rejeter le lot · MS-10 Inviter un membre · MS-11 Retirer le membre · MS-12 Ajouter un bénéficiaire · MS-13 Supprimer le bénéficiaire · MS-14 Se déconnecter · MS-15 Révoquer l'appareil · MS-16 Changer de compte · MS-17 Signaler un problème · MS-18 Reprise de capture · MS-19 Filtres (activité) — voir [docs/15](15-mobile-app-screens-spec.md#ms-01--feuille-derreur--error-sheet-global) à partir de MS-01.

### 4.5 Index des flows (docs/16)

| ID | Nom (FR) | Acteur | Écrans touchés (principaux) |
|---|---|---|---|
| F-001 | Inscription marchand | Marchand | D-02, D-03 · M-02–M-05 · D-11/M-12 |
| F-002 | Connexion (avec 2FA) | Marchand | D-01 · M-07 |
| F-003 | Mot de passe oublié | Marchand | D-04, D-05 |
| F-004 | Réinitialisation du PIN (app) | Marchand | M-07, M-04, M-05, M-06 |
| F-005 | Vérification KYB (onboarding) | Marchand | D-07–D-10 · M-08–M-11 |
| F-006 | Nouvelle soumission après rejet | Marchand | D-39 · M-42 |
| F-007 | Passage en mode réel | Marchand | bascule §0.2 (docs/12), D-32, D-39 · docs site (F-068) |
| F-008 | Encaissement au comptoir par téléphone | Marchand | M-15–M-19 · DM-04 |
| F-009 | Encaissement au comptoir par QR | Marchand | M-15, M-16, M-20, M-18 · C-01/C-02 |
| F-010 | Créer et partager un lien | Marchand | D-14, D-15, DM-05, DM-06 · M-22–M-25 |
| F-011 | Désactiver un lien | Marchand | D-13/D-15, DM-07 · M-21/M-24, MS-06 |
| F-012 | Remboursement | Marchand | DM-01, DM-02 · M-27, MS-07 |
| F-013 | Exporter les relevés | Marchand | D-12/D-29/D-31, DM-35 · M-36 |
| F-014 | Paiement sortant individuel | Marchand | D-20, D-23, D-19 · M-34, MS-08 |
| F-015 | Paiement en masse CSV | Marchand | D-20–D-24 |
| F-016 | Paie programmée | Marchand | D-25, DM-30, D-24 · M-30 |
| F-017 | Approbation d'un lot | Marchand | D-24, DM-08, DM-28 · M-30, M-31 |
| F-018 | Rejet d'un lot | Marchand | D-24, DM-09 · MS-09 |
| F-019 | Approvisionnement du portefeuille | Marchand | D-29, DM-16 · M-35 |
| F-020 | Règlement reçu & changement de calendrier | Marchand | D-30, D-31, DM-17 · M-38, M-39 |
| F-021 | Inviter un membre | Marchand | D-37, DM-23, D-06 · M-40, MS-10 |
| F-022 | Changer un rôle | Marchand | D-37, DM-24 |
| F-023 | Retirer un membre | Marchand | D-37, DM-25 · MS-11 |
| F-024 | Créer / faire tourner une clé API | Marchand | D-32, DM-18 |
| F-025 | Révoquer une clé API | Marchand | D-32, DM-19 |
| F-026 | Configurer un webhook | Marchand | D-33, DM-20, DM-21 |
| F-027 | Webhook en échec & réactivation | Marchand | D-34, D-35 |
| F-028 | Créer un plan | Marchand | D-27, DM-13 |
| F-029 | Abonner un client | Marchand | D-26, DM-33 |
| F-030 | Relance manuelle | Marchand | D-28, DM-34 |
| F-031 | Enregistrement d'un appareil | Marchand | M-03–M-06 · D-41, M-43 |
| F-032 | Révocation d'un appareil | Marchand | D-41, DM-27 · M-43, MS-15 |
| F-033 | Récupération de compte | Marchand | D-04, D-49 |
| F-034 | Changement d'entreprise | Marchand | DM-29 · MS-16 |
| F-035 | Payer via lien réutilisable | Payeur | C-01/C-02, C-08, C-09, C-10 |
| F-036 | Payer via lien à usage unique | Payeur | C-01, C-07–C-10 |
| F-037 | Payer via session de checkout | Payeur | C-04, C-08–C-10, CM-04 |
| F-038 | QR statique au comptoir | Payeur | C-16, C-02, C-08–C-10 |
| F-039 | Réessayer après échec | Payeur | C-11, C-12 |
| F-040 | Renvoyer la demande | Payeur | C-09, CM-03 |
| F-041 | Changer de numéro / canal en cours | Payeur | C-09/C-11, CM-01, CM-02, C-08 |
| F-042 | Vérifier un reçu | Payeur | C-14, C-15 |
| F-043 | Approbation du cycle d'abonnement | Payeur | (USSD téléphone), C-14 |
| F-044 | Paiement via lien de relance | Payeur | C-01, C-08–C-10 |
| F-045 | Approbation KYB | Ops | O-03, O-04, OM-02 |
| F-046 | Rejet KYB | Ops | O-04, OM-01 |
| F-047 | Montée en niveau (Tier 2) | Ops | O-06, OM-02/OM-04, O-17, OM-11 |
| F-048 | Suspension d'un marchand | Ops | O-06, OM-05, O-17 · bannière §0.3.6 docs/12 |
| F-049 | Rapprochement (import → correspondance → résolution) | Ops | O-09, O-10, OM-13, OM-07, OM-08 |
| F-050 | Ajustement manuel (double contrôle) | Ops | O-15, O-16, OM-10, O-17, OM-11 |
| F-051 | Coupe-circuit & rétablissement | Ops | O-11, OM-09, O-17 |
| F-052 | Alerte risque → dossier → résolution | Ops | O-12, O-13 |
| F-053 | Déclaration de soupçon (ANIF) | Ops | O-13, O-14 |
| F-054 | Intégration du personnel ops | Ops | O-19, OM-12, O-18 |
| F-055 | Cycle de vie d'un encaissement | Système | C-09 · D-35 · M-18 |
| F-056 | Expiration des encaissements | Système | C-12 · M-19 |
| F-057 | Règle de re-vérification sur callback | Système | — (adaptateur provider) |
| F-058 | Échelle de reprise webhook | Système | D-34, D-35 |
| F-059 | Job de règlement T+1 | Système | D-30/D-31 · M-38/M-39 |
| F-060 | Facturation d'abonnement (échelle → past_due → annulation) | Système | D-28, D-26 |
| F-061 | Rapprochement automatique des approvisionnements | Système | DM-16 · M-35 |
| F-062 | Assertions nocturnes du grand livre | Système | O-11 (deltas) |
| F-063 | Moniteur synthétique → statut → bannières | Système | D-11 · M-14 · O-11 · checkout (canaux) |
| F-064 | Alerte de dérive de solde | Système | O-11, O-15 |
| F-065 | Rejeu d'idempotence | Système | — (passerelle API) |
| F-066 | Parcours doré sandbox (< 30 min) | Développeur | D-32, D-33, D-12 · docs site |
| F-067 | Numéros magiques | Développeur | — (contrat simulateur, docs/02 §3) |
| F-068 | Liste de contrôle avant mise en production | Développeur | docs site (état persisté par marchand) |
| F-069 | Vérification de signature webhook | Développeur | — (serveur marchand) |
| F-070 | Activer la 2FA (TOTP) | Marchand | D-41, DM-26 |
| F-071 | Notifications in-app (cloche) | Marchand | D-43, D-40 |
| F-072 | Intégration checkout e-commerce | Développeur | C-04, C-09/C-10 · D-33 |
| F-073 | Intégrer via SDK JS/TS ou PHP | Développeur | docs site |
| F-074 | Démarrage via le site de docs | Développeur | docs.ijimpay.com |

Notifications : N-01…N-35 — catalogue complet (canaux, copie FR/EN) dans [docs/16 Annexe A](16-flows-catalog.md#appendix-a--catalogue-complet-des-messages--complete-message-catalog).

## 5. Consolidated additions needed

### 5.1 API additions needed (merged + deduped — for updating docs/02)

Merged from docs/12 §3, docs/14, docs/15 and docs/16 (API-ADD-1…21). Docs/16's converged names win where docs diverged. Items 1–46 belong in the merchant/public API (docs/02); the ops console's internal API (docs/13) is a separate block below.

**Auth & session**
1. `POST /auth/signup/otp` · `POST /auth/signup`
2. `POST /auth/otp` · `POST /auth/otp/verify` (API-ADD-1, canonical)
3. `POST /auth/login` · `POST /auth/login/totp`
4. `POST /auth/password/forgot` · `POST /auth/password/reset`
5. `POST /auth/recovery` (API-ADD-13, D-49/F-033)
6. `POST /auth/step_up` (re-prompt 2FA DM-28)
7. App-only: `POST /auth/pin/set` · `POST /auth/token/refresh` · `POST /auth/logout`

**Utilisateur (`/me`)**
8. `GET /me` · `PATCH /me` · `POST /me/password`
9. `POST /me/totp` · `POST /me/totp/verify` · `DELETE /me/totp` (API-ADD-21, F-070)
10. `GET /me/sessions` · `DELETE /me/sessions/{id}`
11. `GET /me/devices` · `DELETE /me/devices/{id}` + app-side `POST /devices` (jeton FCM) · `POST /devices/{id}/enroll` (API-ADD-12, F-031/F-032)
12. `GET /me/notifications` · `POST /me/notifications/mark_all_read` (API-ADD-20, D-43/F-071; app: `GET /notifications`, `POST /notifications/mark_all_read`)
13. `GET /me/notification_preferences` · `PATCH /me/notification_preferences` (docs/15 uses `PUT` — converge on one verb)
14. `GET /me/memberships` · `POST /me/switch` · `DELETE /me/memberships/{id}` (API-ADD-14, F-034)

**Marchand & KYB**
15. `GET /merchant` · `PATCH /merchant` (incl. seuil d'approbation D-38) · `PUT /merchant/profile` (brouillons onboarding, API-ADD-2)
16. `PATCH /merchant/settlement_account` (DM-17, cooldown 24 h)
17. `GET /merchant/kyb_documents` · `POST /merchant/kyb_documents` · `POST /merchant/kyb/submit` · `POST /merchant/kyb/request_tier` (docs/15's unprefixed `GET/POST /kyb_documents` to converge on these)
18. `POST /merchant/logo` (M-41)
19. `GET/PUT /merchant/golive_checklist` (API-ADD-16, F-068)

**Équipe & invitations**
20. `GET /members` · `PATCH /members/{id}` · `DELETE /members/{id}` (API-ADD-8)
21. `GET /invites` · `POST /invites` · `POST /invites/{id}/resend` · `DELETE /invites/{id}`
22. `GET /invites/{token}` · `POST /invites/{token}/accept` · `POST /invites/{token}/decline` (D-06)

**Argent & objets**
23. `GET /payout_batches` (liste) · `POST /payout_batches/validate` (API-ADD-19, D-22/F-015)
24. `POST /payout_batches/{id}/reject` (API-ADD-6) · `GET /payout_batches/{id}/report`
25. `POST /payouts/{id}/retry` (DM-12)
26. `GET/POST/PATCH/DELETE /beneficiaries` (+`/{id}`)
27. `GET/POST/PATCH/DELETE /payroll_lists` (+`/{id}`) · `POST /payroll_lists/{id}/schedule` (API-ADD-5, F-016/DM-30)
28. `POST /balance_transfers` (API-ADD-7, DM-16 voie 2) · champ portefeuille de paiement dans `GET /balance`
29. `GET /fees` (barème par canal, API-ADD-19, D-23/F-015)
30. `GET /plans` · `PATCH /plans/{id}` · `POST /plans/{id}/archive`
31. `POST /invoices/{id}/send_link` · `POST /invoices/send_links` (API-ADD-11, F-030/DM-34)
32. `PATCH /payment_links/{id}` (API-ADD-3, édition + réactivation) · `GET /payment_links/{id}/stats`
33. Filtres : `GET /charges?payment_link=` · `GET /subscriptions?customer=`
34. `POST /charges/{id}/cancel` (API-ADD-18, renvoi côté marchand M-27)
35. `POST /charges/{id}/receipt/send` (DM-03) · `GET /receipts/{charge_id}.pdf`
36. `GET /customers/{id}` · `PATCH /customers/{id}` · `POST /customers/{id}/notes`
37. `GET /settlements/{id}/statement.pdf` · `GET /settlements/{id}/statement.csv`
38. `GET /stats/volume` (graphique D-11)
39. `POST /exports` · `GET /exports/{id}` (API-ADD-4; supersedes docs/15's `GET /exports/charges?month=` — flagged [MINOR])
40. `GET /search` (palette ⌘K D-44)

**Développeur & plateforme**
41. `GET/POST/DELETE /api_keys` (+`/{id}`) · `POST /api_keys/{id}/rotate` (API-ADD-9, fenêtre de grâce F-024)
42. `PATCH /webhook_endpoints/{id}` · `POST /webhook_endpoints/{id}/enable` · `POST /webhook_endpoints/{id}/roll_secret` · `GET /webhook_endpoints/{id}/deliveries` · `POST /webhook_deliveries/{id}/redeliver` (API-ADD-10)
43. `GET /api_logs` · `GET /api_logs/{id}` · `POST /test_events`
44. App : `GET /app/config` (version min, maintenance, codes USSD, FAQ, feature flags) · `GET /channels/{channel}/history` · `POST /support/tickets`

**Surface publique payeur (`pay.ijimpay.com`, API-ADD-17 / docs/14)**
45. `GET /v1/public/links/{slug}` · `GET /v1/public/checkout_sessions/{id}` · `POST /v1/public/charges` · `GET /v1/public/charges/{id}` · `GET /v1/public/charges/{id}/stream` (SSE) · `POST /v1/public/charges/{id}/resend` · `GET /v1/public/receipts/{receipt_id}` (+`/pdf`)
46. Champs/objets : `payment_link` — champs catalogue (`product_name`, `image_url`, `unit_price`, `max_quantity`), `min_amount` (montant libre) ; profil public marchand — `support_phone`, `support_whatsapp`, `fee_passthrough`, `accepted_channels[]` ; nouveaux types push (docs/15) : `kyb.decision`, `security.new_device`, `team.member_changed`, `payout_batch.pending_approval`, `payout_batch.partially_failed`.

**API interne ops (docs/13 — à spécifier dans un futur `docs/17-internal-api.md`, pas dans docs/02)** : ~70 endpoints `https://api.ijimpay.com/internal/v1/*` (auth SSO/WebAuthn, files KYB, marchands, transactions/re-poll, réconciliation, providers/disjoncteur, risque/STR, grand livre/ajustements, demandes d'approbation DC, journal d'audit, personnel, paramètres) + nouveaux codes d'erreur (`approver_is_maker`, `approval_request_expired`, `unbalanced_journal_entry`, `webauthn_required`, `last_ops_admin`) — liste complète dans [docs/13 §API additions](13-ops-console-spec.md#api-additions-needed).

### 5.2 Icon additions needed (merged — for updating docs/07 §4)

Union of the five docs' lists (one icon = one meaning; source docs in parentheses):

`rotate-cw` réessayer/relivrer/re-poll (12/13/14/15/16) · `eye` / `eye-off` afficher-masquer mot de passe et montants (12/15) — **attention** : docs/13 emploie aussi `eye` pour l'impersonation lecture seule, à arbitrer (deux sens) · `upload` téléverser/import CSV-rapport (12/16) · `camera` capture document (12/15) · `file-up` import CSV (12) · `table` correspondance de colonnes (12) · `list-checks` rapport de validation (12) · `calendar` sélecteur de période/date (12/15) · `user-round-plus` inviter (12) · `smartphone` appareils enrôlés (12/15/16) · `x` fermer modal/drawer/widget (12/14/15) · `wrench` maintenance (12/13) · `log-in` se reconnecter (12/13) · `circle-pause` / `circle-play` pause-reprise d'abonnement (12/16) · `wifi-off` hors ligne (alternative à `cloud-off`, 12) · `scale` réconciliation & écarts (13/16) · `siren` risque — files, dossiers, alertes (13/16; `flag` n'est pas utilisé) · `octagon-pause` suspension / disjoncteur (13) · `book-text` écritures comptables manuelles (13) · `minus-circle` décrément de quantité (14) · `chevron-down` accordéon (14) · `external-link` ouvrir dans le navigateur (14) · `phone` appeler le commerçant (14) vs `phone-call` repli OTP vocal (16) — **conflit ouvert, un seul nom Lucide à retenir** · `image` (15) · `flashlight` (15) · `delete` retour arrière du pavé (15) · `chevron-right` (15) · `book-user` sélecteur de contacts (15) · `bluetooth` appairage imprimante (15) · `plus` FAB app (15/16) · `play-circle` tutoriels vidéo (15) · `building-2` sélecteur d'entreprise (16) · `file-text` relevés (16).

Rappel de style : la pastille `draft` (contour ink-500, lots de paiement uniquement) n'est **pas** un 9ᵉ statut StatusBadge — les 8 canoniques restent ceux de docs/07 (dont `pending_approval` warning-600 contour).

## 6. Traceability — PRD (docs/01) → pages + flows

| PRD § | Exigence (résumé) | Pages / écrans couvrants | Flows couvrants |
|---|---|---|---|
| §1 Onboarding & comptes | signup, KYB, paliers 0/1/2, rôles, clés API, multi-entreprises | D-01–D-10, D-32, D-37, D-39, D-42, D-49, DM-18/19/23/24/25/26/29 · M-01–M-11, M-40–M-43, MS-10/11/16 · O-03/O-04, OM-01/02/03 | F-001–F-007, F-021–F-025, F-031–F-034, F-045–F-047, F-070 |
| §2 Encaissements (charges) | création API, cycle async, polling+webhooks, idempotence, mode test, détection canal, remboursements | DM-04, D-12, DM-01, DM-02 · M-15–M-20, M-27, MS-07 · C-08–C-13 | F-008, F-009, F-012, F-039–F-041, F-055–F-057, F-065, F-066, F-067 |
| §3 Liens & checkout hébergé | lien < 30 s, page hébergée FR/EN 3G, QR, partage WhatsApp/SMS, catalogue+quantité, widget, redirect | D-13–D-16, DM-05/06/07 · M-21–M-25, MS-05/06 · C-01–C-08, C-16–C-20 | F-010, F-011, F-035–F-038, F-072 |
| §4 Décaissements (payouts) | payout unique + masse CSV, maker–checker à seuil configurable (D-38), lots avec retry + rapport, préfinancement | D-19–D-25, D-38 (seuil), DM-08–DM-12, DM-16, DM-30 · M-29–M-35, MS-08/09/12/13 | F-014–F-019 |
| §5 Abonnements & récurrent | plans, cycle `active→past_due→canceled`, échelle de reprise, relance SMS/WhatsApp, pause/reprise | D-26–D-28, DM-13/14/15/33/34 · C-01 (lien de relance) | F-028–F-030, F-043, F-044, F-060 |
| §6 Grand livre, soldes & règlement | ledger double entrée, disponible vs en attente, T+1 configurable, relevés par rail, réconciliation auto + file d'écarts | D-29–D-31, DM-16/17, DM-35 (filtre Canal) · M-37–M-39 · O-09/O-10/O-15/O-16, OM-08/10/13 | F-013, F-019, F-020, F-049, F-050, F-059, F-061, F-062, F-064 |
| §7 Webhooks & DX | signatures HMAC, échelle 72 h, journal + relivraison, OpenAPI 3.1 + docs site, SDK JS/TS & PHP | D-32–D-36, DM-18–DM-22, DM-31/32 · docs.ijimpay.com | F-024–F-027, F-058, F-066–F-069, F-072–F-074 |
| §8 Dashboard web | transactions + timeline, accueil du jour, liens, payouts + approbations, équipe, section dev, cloche + digests | docs/12 entier — noyau D-11, D-12/DM-01, D-13, D-19, D-37, D-32–D-36, D-43/D-44 | F-013, F-071 (+ N-35 digest) |
| §9 Application mobile | encaisser au comptoir, son+push succès, totaux du jour, liens, approbations payout, solde, tolérance hors-ligne | docs/15 entier — noyau M-12, M-15–M-20, M-21–M-25, M-29–M-31, M-37 | F-002, F-004, F-008, F-009, F-017, F-031 |
| §10 Notifications | push, email, SMS critiques, WhatsApp reçus/relance | D-40, D-43 · M-13, M-44 · Annexe A N-01–N-35 | F-071 (+ toutes les lignes N-xx des flows) |
| §11 Admin (console ops) | file KYB, recherche transactions inter-marchands, file d'écarts, santé providers, ajustements DC + audit, drapeaux de risque | docs/13 entier — O-02–O-20, OM-01–OM-14 | F-045–F-054 |
| §12 Non fonctionnel | dispo 99,9 % + dégradation par provider, durabilité outbox/at-least-once/ledger append-only, FR+EN, checkout Android 8+/2G-3G, auditabilité | bannières §0.3 docs/12 · M-14/M-48/M-49 · C-17 (mode dégradé), C-14 (zéro JS) · O-11, O-18 | F-051, F-055–F-058, F-062–F-065 |

## 7. Open issues — remaining gaps targeting reference docs (docs/01–10)

All `[MUST]` findings in `docs/review/` that targeted docs/12–16 were verified as resolved in the final spec docs (spot-checked: charge cancel/resend API-ADD-17/18, 1 000-line CSV cap, D-49 recovery, D-38 approval threshold, DM-16 internal transfer, DM-30 scheduling, D-32 rotation, D-34 re-enable, DM-35 rail filter + Viewer, catalog fields in D-14, suspension/impersonation banners §0.3, O-21–O-24, OM-02 Tier-2 DC, F-070–F-074, M-33 note). What remains open lands in the **reference docs**, which the spec docs cannot fix themselves:

1. **docs/02 (API spec)** — must absorb the consolidated additions §5.1 items 1–46 (endpoints, filters, fields, SSE stream, public payer surface) and the five new push/event types; until then the five spec docs reference endpoints that do not exist in the contract.
2. **docs/07 (design system)** — §4 icon map must absorb the merged icon list §5.2; two cross-doc conflicts need an arbitration recorded in docs/07: `phone` (docs/14) vs `phone-call` (docs/16) for the "call" concept, and `eye` double-booked (password/amount reveal in 12/15 vs read-only impersonation in 13).
3. **docs/03 (data model)** — additions required by docs/13: `approval_requests`, `str_packs`, `risk_rules`/`risk_flags`/`risk_cases` (+ items & notes), `staff_users`/`staff_webauthn_credentials`, `ops_settings`, `merchant_limits` (+ history), append-only notes tables; plus promote the `payout_inflight` clearing account (used by F-014 postings) from prose into the chart of accounts (review [MINOR], still open).
4. **docs/17-internal-api.md** — does not exist yet; docs/13 defers its ~70 internal endpoints and 5 new error codes there. Creation is a prerequisite for building the ops console.
5. **docs/08 §A, docs/09, docs/10** — superseded by docs/12, docs/14 and docs/15 respectively but not yet marked as such in their own headers; docs/08 §B (ops) and §C (notification matrix) remain authoritative and are referenced by 13/15/16. A supersession banner in each avoids double-maintenance drift.
6. **docs/15 INVENTORY `tables=69`** (review [MINOR], unresolved) — counts markdown tables, not rendered UI tables; either restate the counting rule in docs/15 or align on the docs/12/13 rule. Carried here so the cross-surface totals in §2 stay honest.

---

```
INVENTORY-MASTER: surfaces=4 pages=145 modals=74 tabs=29 flows=74 notifications=35 api_additions=46(+internal) icon_additions=34
```
