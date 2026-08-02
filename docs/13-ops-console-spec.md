# Ijim Pay — Ops Console Spec (`ops.ijimpay.com`)

Status: Draft v1 · Supersedes `docs/08-web-screens.md` §B · Internal-only app, separate deployment, SSO + hardware security keys (WebAuthn) mandatory.

Conventions in this document:
- **Roles** (per docs/08 §B): `ops`, `ops-admin`, `compliance`, `finance-admin`. Read access is stated per page; write actions state the minimum role.
- All copy FR-first; amounts always `12 500 FCFA` (thin space, currency after). Icons: Lucide, per the map in docs/07 §4; new icons are listed only in "Icon additions needed" at the end.
- StatusBadge uses only the 8 canonical statuses (`succeeded`, `pending`, `failed`, `expired`, `refunded`, `processing`, `pending_approval`, plus `draft` rendered as ink-500 outline — see Icon/API additions notes). `pending_approval` renders warning-600 **outlined** (accent-500 is banned from transactional UI).
- The merchant-facing API (docs/02) is referenced exactly where reused. Every internal endpoint (prefix `/internal/v1/...`) does **not** exist in docs/02 and is declared in "API additions needed" at the end — none is invented silently inline; the actions tables use those declared paths.
- **Inventory counting rules** (for the final block): pages = `O-xx` sections; tabs = rows under a page's **Tabs**; modals = `OM-xx` sections; forms = Inputs tables containing ≥ 1 field; tables = data tables rendered in the UI (listed in Layout/Data/Tabs), including one in OM-10; actions = total rows across all **Actions** tables.

## Global chrome

Left sidebar 240px (collapsible): Accueil `layout-dashboard` · File KYB `badge-check` · Marchands `user-round` · Transactions `arrow-left-right` · Réconciliation `scale` · Fournisseurs `activity` · Risque `siren` · Ajustements `book-text` · Approbations `check-check` · Audit `scroll-text` · Personnel `users-round` · Paramètres `settings`. Topbar: environment badge (`PROD` danger-600 outline / `STAGING` info-600), ⌘K global search (marchand, ref charge, téléphone, provider_ref), `bell` notifications, avatar (nom, rôle, "Se déconnecter" `log-out`). No test/live toggle — ops console is live-only; test-mode objects are visible read-only, always flagged `flask-conical` "Mode test".

**Dual-control (two-approver) mechanics — canonical rules, referenced everywhere as "règle DC":**
1. The maker submits a request → object status `pending_approval`; it appears in O-17 and notifies eligible approvers.
2. The approver MUST be a different staff user than the maker (server-enforced, HTTP 403 `approver_is_maker`), MUST hold the required role, and MUST re-authenticate with their hardware key (WebAuthn assertion sent with the approve call).
3. Reject always requires a reason (≥ 10 characters). Approve restates all critical values (amount, merchant, accounts) in OM-11 — the approve button is never default-focused.
4. Requests expire unactioned after 24 h (status `expired`; maker notified) except adjustments (72 h).
5. Everything (create, approve, reject, expire) writes `audit_logs` (append-only, docs/03).
6. Every money-moving approval shows the journal-entry preview (OM-10) before the final confirm — postings per the docs/03 chart of accounts, sum = 0 or the confirm button stays disabled.

**SLA timers — canonical rules:** every queue row carries an SLA chip: elapsed time vs target (KYB 24 h ouvrées · écarts de réconciliation 48 h · dossiers risque 72 h · approbations DC 4 h). < 75 % = ink-500 `clock`; 75–100 % = warning-600 `clock`; breached = danger-600 `triangle-alert` + row pinned to top. Targets editable in O-20 (ops-admin).

---

## Pages

### O-01 — Connexion SSO / SSO Login
- **Route**: `/login` · **Icon**: lucide `lock` · **Access**: public (staff only past IdP) · **Purpose**: authenticate staff via corporate SSO then enforce hardware-key second factor.
- **Layout zones**: centered card (max 400px) on `paper` background — logo, title "Console Ops Ijim Pay", SSO block, security-key step, footer "Accès réservé au personnel autorisé. Toute action est journalisée."
- **Tabs**: none.
- **Data displayed**: IdP name from config; after IdP return: staff name, role, enrolled-key count (`GET /internal/v1/staff/me`).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Adresse e-mail professionnelle | email | domaine ∈ liste autorisée (`@ijimpay.com`) | vide | « Utilisez votre adresse professionnelle @ijimpay.com. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Se connecter via SSO | `log-out` (miroir, entrée) | — | Redirection OIDC vers l'IdP (`GET /internal/v1/auth/sso/start?login_hint=`); au retour, passe à l'étape clé de sécurité |
| Utiliser ma clé de sécurité | `fingerprint` | staff authentifié IdP | WebAuthn `navigator.credentials.get` → `POST /internal/v1/auth/webauthn/verify` → session 8 h, redirection `/` |

- **Modals/drawers/sheets opened**: none.
- **States**: loading (spinner sur bouton); error IdP « Connexion refusée par le fournisseur d'identité. Contactez ops-admin. »; no-key-enrolled « Aucune clé de sécurité enrôlée. Demandez à un ops-admin de vous enrôler (page Personnel). »; session-expired toast « Session expirée — reconnectez-vous. »; rate-limited « Trop de tentatives. Réessayez dans 5 minutes. »
- **Events/notifications triggered**: `audit_logs` login_succeeded / login_failed; e-mail sécurité au staff sur connexion depuis un nouvel appareil.

### O-02 — Accueil ops / Ops Home
- **Route**: `/` · **Icon**: lucide `layout-dashboard` · **Access**: all roles · **Purpose**: one-glance view of every queue, its depth and worst SLA, plus provider health.
- **Layout zones**: header ("Bonjour {prénom}" + date `fr-CM`) · stat tiles row · provider health strip (`activity`, same semantics as merchant Home) · queues table · aside "Mes éléments" (items assigned to me).
- **Tabs**: none.
- **Data displayed**: stat tiles — KYB en attente, Écarts ouverts, Dossiers risque ouverts, Approbations en attente (counts from `GET /internal/v1/queues/summary`); provider health from `GET /channels` (docs/02 §2.8); queues table: file, nombre, plus ancien, SLA le plus critique, assignés; "Mes éléments": type, référence, SLA.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir la file | `arrow-left-right` (nav) | rôle de la file | Navigue vers O-03 / O-09 / O-12 / O-17 selon la ligne |
| Actualiser | `rotate-cw` | tous | Re-fetch `GET /internal/v1/queues/summary` (auto-refresh 60 s par défaut) |

- **Modals/drawers/sheets opened**: none.
- **States**: loading skeleton tiles; empty « Toutes les files sont vides. Belle journée. »; error « Impossible de charger les files (req {request_id}). Réessayer »; permission-denied per-tile: tiles the role can't open are hidden, not disabled.
- **Events/notifications triggered**: none (read-only).

### O-03 — File KYB / KYB Queue
- **Route**: `/queue/kyb` · **Icon**: lucide `badge-check` · **Access**: ops, compliance, ops-admin (finance-admin: read-only) — per docs/16 F-045/F-046, ops works and decides the queue; compliance handles complex/escalated dossiers · **Purpose**: work the queue of merchants awaiting tier verification, ordered by SLA.
- **Layout zones**: header (title + count + SLA summary "3 en dépassement") · toolbar (filters: palier demandé 1/2, statut dossier `nouveau|en revue|infos demandées`, assigné à, tri SLA/date) · content: queue table · aside: quick stats (approuvés/rejetés cette semaine, temps médian).
- **Tabs**: none.
- **Data displayed**: table rows from `GET /internal/v1/kyb/reviews?status=` — merchant `name`, `legal_name`, `tier` actuel → palier demandé (merchants.tier, docs/03), docs count (`kyb_documents`), soumis le, assigné à, SLA chip (règle SLA), StatusBadge dossier (`pending` = nouveau, `processing` = en revue).
- **Inputs**: none (filters only).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Examiner | `badge-check` | ops, compliance | Ouvre O-04 pour la ligne; `POST /internal/v1/kyb/reviews/{id}/claim` (assigne à moi si non assigné) |
| M'assigner | `user-round` | ops, compliance | `POST /internal/v1/kyb/reviews/{id}/claim`; badge "Assigné : {nom}" |
| Exporter la file | `download` | ops-admin | CSV `GET /internal/v1/kyb/reviews/export` (e-mail async si > 10 000 lignes) |

- **Modals/drawers/sheets opened**: none (detail is a full page, O-04).
- **States**: loading skeleton rows; empty « Aucun dossier KYB en attente. La file est à jour. »; error with retry; permission-denied (finance-admin sees table, action buttons hidden); filtered-empty « Aucun dossier ne correspond à ces filtres. Réinitialiser ».
- **Events/notifications triggered**: claim → notification bell à l'ancien assigné si réassignation.

### O-04 — Revue KYB / KYB Review Detail
- **Route**: `/queue/kyb/:reviewId` · **Icon**: lucide `badge-check` · **Access**: ops (decide — F-045/F-046), compliance (decide — cas complexes, documents suspects, escalades), ops-admin (decide), finance-admin (read-only) · **Purpose**: side-by-side verification of uploaded documents against declared business data, ending in approve / reject / request-info.
- **Layout zones**: header (merchant name, palier demandé, SLA chip, assigné) · **left pane: visionneuse de documents** (PDF/image viewer, zoom, rotation, page nav, doc switcher tabs RCCM / Pièce d'identité / Preuve de compte) · **right pane: données déclarées** (form data + checklist) · footer action bar (sticky) · aside historique (previous decisions, notes).
- **Tabs**: none (doc switcher is within the viewer, not page tabs).
- **Data displayed**: documents from `kyb_documents` (docs/03) via `GET /internal/v1/kyb/reviews/{id}` — type, fichier, uploadé le, statut par document; declared data: `merchants.legal_name`, forme juridique, secteur, adresse, owner ID name/number, compte de règlement (canal + numéro, docs/03 `default_settlement_channel`); comparison table: Champ déclaré | Valeur | Concordance (✓/✗ toggle set by reviewer); history: décisions antérieures avec motifs (`audit_logs`).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Concordance — Nom légal | case à cocher (3 états ✓/✗/—) | décision requise sur chaque ligne avant Approuver | — | « Vérifiez chaque élément avant d'approuver. » |
| Concordance — N° RCCM | idem | idem | — | idem |
| Concordance — Identité du dirigeant | idem | idem | — | idem |
| Concordance — Propriété du compte de règlement | idem | idem | — | idem |
| Note interne | texte long | ≤ 2 000 caractères, optionnel | vide | « 2 000 caractères maximum. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Approuver le palier | `circle-check` | ops, compliance | Ouvre OM-02 (choix palier + limites); à la confirmation `POST /internal/v1/kyb/reviews/{id}/approve` — Palier 1 : merchants.tier mis à jour immédiatement, e-mail+push+SMS marchand « Vérification approuvée » (matrice docs/08 §C) ; Palier 2 : règle DC — crée une demande d'approbation ops-admin (O-17, F-047), rien n'est appliqué avant la seconde approbation |
| Rejeter | `circle-x` | ops, compliance | Ouvre OM-01 (bibliothèque de motifs FR); `POST /internal/v1/kyb/reviews/{id}/reject` |
| Demander des informations | `message-square` | ops, compliance | Ouvre OM-03; `POST /internal/v1/kyb/reviews/{id}/request_info` — dossier passe à « infos demandées », SLA en pause |
| Télécharger le document | `download` | ops, compliance | `GET /internal/v1/kyb/documents/{docId}/download` (URL signée 5 min; téléchargement journalisé) |
| Réassigner | `users-round` | ops-admin | `POST /internal/v1/kyb/reviews/{id}/assign` (picker staff) |

- **Modals/drawers/sheets opened**: OM-01, OM-02, OM-03.
- **States**: loading (viewer skeleton + right-pane skeleton); doc-unreadable state per document « Document illisible — impossible d'afficher. Téléchargez-le ou demandez un nouvel envoi. »; error; permission-denied (finance-admin: checklist disabled, footer bar hidden); already-decided banner info-600 « Dossier déjà traité par {nom} le {date}. Lecture seule. »
- **Events/notifications triggered**: approve/reject/request-info → merchant notifications (push + e-mail + SMS per docs/08 §C "KYB approved / rejected"); `audit_logs` entry each decision; bell aux compliance sur réassignation.

### O-05 — Marchands / Merchants
- **Route**: `/merchants` · **Icon**: lucide `user-round` · **Access**: all roles · **Purpose**: search and browse every merchant account.
- **Layout zones**: header · toolbar (search nom/téléphone/ID, filters: statut `pending|active|suspended`, palier 0/1/2, drapeaux de risque oui/non) · content: merchants table.
- **Tabs**: none.
- **Data displayed**: table from `GET /internal/v1/merchants?query=` — nom, legal_name, tier, StatusBadge-like pill statut (`active` success-600 / `pending` warning-600 / `suspended` danger-600 — merchant status pills, distinct from the 8 tx StatusBadge statuses), volume 30 j, solde disponible (AmountText), drapeaux `risk_flags` (docs/03), créé le.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir la fiche | `user-round` | tous | Navigue vers O-06 |
| Exporter | `download` | ops-admin, finance-admin | CSV `GET /internal/v1/merchants/export` |

- **Modals/drawers/sheets opened**: none.
- **States**: loading skeleton; empty « Aucun marchand trouvé pour « {requête} ». »; error; permission-denied n/a (all read).
- **Events/notifications triggered**: none.

### O-06 — Fiche marchand / Merchant Detail
- **Route**: `/merchants/:id` · **Icon**: lucide `user-round` · **Access**: all roles read; write per action · **Purpose**: the complete internal view of one merchant — profile, verification, limits, balances/ledger, activity, risk, team, integration, notes.
- **Layout zones**: header (nom, pill statut, tier badge, drapeaux risque, ID copiable `copy`) · action bar (Impersonner · Suspendre · Modifier les limites) · tab strip · tab content · aside résumé (solde disponible, volume 30 j, dernier règlement, contact Owner).
- **Tabs**:
  1. **Profil** — legal_name, forme, secteur, adresse, country, date de création, `default_settlement_channel`, `settlement_schedule`, contacts Owner (téléphone/e-mail), logo. Read-only; edits happen merchant-side.
  2. **Vérification** — kyb_status, palier actuel, documents table (type, statut, motif de rejet, lien viewer O-04-style read-only), historique des décisions KYB. Bouton « Ouvrir le dossier KYB » si dossier ouvert → O-04.
  3. **Limites & paliers** — current limits table: plafond d'encaissement/jour, plafond de paiement sortant/jour, plafond par transaction, nb max transactions/jour — valeurs vs consommé aujourd'hui (progress bars). Historique des changements (qui, quand, DC). Bouton « Modifier les limites » → OM-04 (règle DC au-delà du Palier 1).
  4. **Soldes & ledger** — three AmountText tiles (Disponible / En attente / Portefeuille de paiement = `merchant_available`, `merchant_pending`, `merchant_payout_wallet`, docs/03) computed from postings (`GET /internal/v1/merchants/{id}/balances`); journal-entries table (kind, référence, montant, créé par, date) → row opens OM-10 in read-only mode; règlements récents (list, statut, destination).
  5. **Activité** — cross-object recent activity: charges/payouts/refunds table (TxRow anatomy) from `GET /internal/v1/transactions?merchant_id=`, filter chips par type/statut; row → O-08.
  6. **Risque** — `risk_flags` list with raised-on date and rule name, dossiers risque liés (→ O-13), velocity mini-charts (tx/h vs baseline), bouton « Ouvrir un dossier risque » → `POST /internal/v1/risk/cases`.
  7. **Équipe & accès** — memberships table (utilisateur, rôle owner/admin/developer/finance/viewer, 2FA, dernière activité, docs/03 `memberships`), API keys metadata (prefix, mode, last_used_at, revoked_at — **jamais** la clé), sessions actives.
  8. **Webhooks & intégration** — webhook_endpoints (URL, événements, santé), dernières webhook_deliveries en échec, derniers appels API ≥ 400 (`GET /internal/v1/merchants/{id}/api-logs`).
- **Data displayed**: as itemized per tab above; sources: `GET /internal/v1/merchants/{id}` plus per-tab endpoints listed in API additions; ledger fields per docs/03 §3.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Nouvelle note interne | texte long | 3–2 000 caractères | vide | « La note doit contenir au moins 3 caractères. » |

  (notes composer lives in the aside; list of notes below it, auteur + date, non modifiables — append-only.)
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Impersonner (lecture seule) | `eye` | ops-admin | Ouvre OM-06 → `POST /internal/v1/merchants/{id}/impersonate` → ouvre app.ijimpay.com en session lecture seule 30 min, bandeau danger-600 « Session ops — lecture seule », intégralement journalisé |
| Suspendre le marchand | `octagon-pause` | ops-admin (maker), 2e approbateur ops-admin | Ouvre OM-05 (règle DC) — `POST /internal/v1/merchants/{id}/suspension_requests`; à l'approbation (O-17/OM-11) le statut passe à `suspended`, API du marchand renvoie `permission_denied`, e-mail marchand |
| Réactiver le marchand | `circle-check` | ops-admin + DC | Même flux que Suspendre via OM-05 (variante réactivation) |
| Modifier les limites | `pencil` | ops-admin (maker) ; DC si palier > 1 ou hausse > 20 % | Ouvre OM-04 → `POST /internal/v1/merchants/{id}/limit_change_requests` |
| Ajouter une note | `pencil` | tous sauf lecture seule | `POST /internal/v1/merchants/{id}/notes` |
| Copier l'ID marchand | `copy` | tous | Presse-papiers + toast « ID copié » |

- **Modals/drawers/sheets opened**: OM-04, OM-05, OM-06, OM-10 (read-only), OM-14.
- **States**: loading per-tab skeletons; empty per tab (Activité : « Aucune activité pour ce marchand. » · Risque : « Aucun drapeau de risque. » · Notes : « Aucune note interne. »); error; permission-denied: action bar hidden for non-eligible roles; **suspended banner** danger-600 sticky « Marchand suspendu le {date} par {maker}, approuvé par {checker} — portée : {portée} — motif : {motif} »; pending-DC banner warning-600 outlined « Une demande (suspension / limites) attend une seconde approbation » avec lien O-17.
- **Events/notifications triggered**: suspension approve → merchant e-mail + push « Votre compte est suspendu — contactez le support » ; limit change approve → e-mail marchand ; note → aucune ; toutes actions → `audit_logs`.

### O-07 — Recherche transactions / Transactions Search
- **Route**: `/transactions` · **Icon**: lucide `arrow-left-right` · **Access**: all roles · **Purpose**: cross-merchant search of any charge, payout or refund by any field.
- **Layout zones**: header · search form card · results table · footer pagination (cursor).
- **Tabs**: none.
- **Data displayed**: results from `GET /internal/v1/transactions` — type (charge/payout/refund), marchand, StatusBadge, ChannelChip, AmountText, référence, provider_ref, téléphone client/bénéficiaire, failure_code, date. Field semantics mirror docs/02 §2.1/§2.3 objects.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Recherche libre | texte | ≥ 3 caractères si non vide | vide | « Saisissez au moins 3 caractères. » |
| Référence provider | texte | UUID ou vide | vide | « Format de référence provider invalide. » |
| Téléphone | tel | E.164 `2376XXXXXXXX` ou national 9 chiffres | vide | « Numéro invalide — format 6 XX XX XX XX. » |
| Montant min / max | entier FCFA | ≥ 0 ; min ≤ max | vide | « Le montant minimum dépasse le maximum. » |
| Période | plage de dates | ≤ 366 jours | 30 derniers jours | « La période ne peut pas dépasser 366 jours. » |
| Statut | multi-select (8 statuts) | — | tous | — |
| Canal | select mtn_momo / orange_money | — | tous | — |
| Marchand | autocomplete | — | tous | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Rechercher | `search` | tous | `GET /internal/v1/transactions?…` |
| Exporter les résultats | `download` | ops-admin, finance-admin, compliance | CSV (async e-mail > 10 000 lignes) ; export journalisé (PII) |

- **Modals/drawers/sheets opened**: none (row → O-08).
- **States**: loading skeleton; empty « Aucune transaction ne correspond. Élargissez la période ou vérifiez la référence. »; error; permission-denied n/a; too-broad warning « Plus de 50 000 résultats — affinez la recherche. »
- **Events/notifications triggered**: export → `audit_logs` (accès PII en masse).

### O-08 — Détail transaction / Transaction Detail
- **Route**: `/transactions/:id` · **Icon**: lucide `arrow-left-right` · **Access**: all roles read; actions per row · **Purpose**: the full internal truth of one transaction — merchant view plus raw provider payloads, ledger postings and delivery trail.
- **Layout zones**: header (AmountText XL, StatusBadge, ChannelChip, marchand → O-06) · timeline (créé → pending → terminal, avec `last_polled_at`, `poll_count`, docs/03 `provider_transactions`) · panels: Objet (champs docs/02 §2.1), **Payloads provider bruts** (JSON viewer `raw_request`/`raw_response`, copiable, PII masquée par défaut avec bouton « Révéler » journalisé), **Écritures comptables** (postings table: compte, débit/crédit, montant — journal_entries liées, docs/03 §3), **Livraisons webhook** (webhook_deliveries table : tentative, code, prochaine relance), Objets liés (lien, invoice, refund, batch).
- **Tabs**: none.
- **Data displayed**: as per zones; source `GET /internal/v1/transactions/{id}` (embeds charge/payout object per docs/02, `provider_transactions`, postings, `webhook_deliveries`).
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Re-interroger le provider | `rotate-cw` | ops | Ouvre OM-07 → `POST /internal/v1/transactions/{id}/repoll` ; timeline se met à jour (websocket) |
| Renvoyer le webhook | `webhook` | ops | `POST /internal/v1/webhook_deliveries/{deliveryId}/redeliver` ; toast « Livraison relancée » |
| Copier la référence | `copy` | tous | Presse-papiers `id` + `provider_ref` |

- **Modals/drawers/sheets opened**: OM-07, OM-10 (postings row → read-only preview of the parent journal entry).
- **States**: loading; error; permission-denied n/a; **stuck-pending state** (pending > 2× expiry): warning banner « Transaction bloquée en attente depuis {durée} — re-interrogez le provider ou ouvrez un écart. » with link to O-09; test-mode banner `flask-conical` « Mode test — aucun argent réel » (lecture seule, actions masquées).
- **Events/notifications triggered**: repoll résultat → bell à l'initiateur ; reveal PII → `audit_logs`.

### O-09 — Réconciliation / Reconciliation
- **Route**: `/reconciliation` · **Icon**: lucide `scale` · **Access**: finance-admin, ops-admin (ops: read-only) · **Purpose**: import provider reports, run matching, and drive every discrepancy to resolution.
- **Layout zones**: header (dernier import par provider + statut SFTP) · toolbar (provider filter, période, bouton Importer) · tab strip · tab content table · aside stats (taux de correspondance du dernier run, écarts ouverts par âge).
- **Tabs**:
  1. **Imports** — provider_reports table (provider, période, fichier, importé le, lignes, correspondances %, statut du run matched/en cours/échec) ; row expands to run summary.
  2. **Manquants chez nous (missing_ours)** — reconciliation_items table: provider_ref, montant, téléphone, date provider, SLA chip, assigné ; le provider a l'argent, pas nous.
  3. **Manquants chez eux (missing_theirs)** — same columns keyed on our charge: charge id, montant, provider_ref attendu ; nous avons `succeeded`, le rapport ne l'a pas.
  4. **Écarts de montant (amount_mismatch)** — charge id, montant chez nous vs montant provider (delta AmountText signé), SLA chip.
- **Data displayed**: `provider_reports` and `reconciliation_items` per docs/03 §4 via `GET /internal/v1/reconciliation/reports` and `GET /internal/v1/reconciliation/items?status=`.
- **Inputs**: none (import via OM-13).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Importer un rapport | `download` (sens import) | finance-admin | Ouvre OM-13 → `POST /internal/v1/reconciliation/reports` puis run de matching auto |
| Relancer le matching | `rotate-cw` | finance-admin | `POST /internal/v1/reconciliation/reports/{id}/rematch` |
| Enquêter | `search` | finance-admin, ops | Ouvre O-10 pour l'élément |

- **Modals/drawers/sheets opened**: OM-13.
- **States**: loading; empty (tab écarts) « Aucun écart ouvert. Les livres sont à l'équilibre. » `circle-check`; empty (Imports) « Aucun rapport importé ce mois-ci. »; error; permission-denied (ops: read-only, boutons masqués); SFTP-down banner warning « Le dépôt SFTP {provider} n'a pas livré de fichier depuis 26 h. »
- **Events/notifications triggered**: import terminé → bell finance-admin « Matching terminé : {n} écarts » ; écart > 48 h → e-mail quotidien finance-admin.

### O-10 — Enquête d'écart / Discrepancy Investigation
- **Route**: `/reconciliation/items/:id` · **Icon**: lucide `scale` · **Access**: finance-admin (resolve), ops (read + note) · **Purpose**: side-by-side comparison of our record vs the provider report line, ending in a resolution (with adjustment when money moved wrongly).
- **Layout zones**: header (type d'écart badge, montant en jeu AmountText, SLA chip, assigné) · **comparison table** (Champ | Chez nous | Chez le provider | Δ — rows: montant, statut, provider_ref, téléphone, horodatage) · linked-objects panel (charge → O-08, marchand → O-06, `provider_transactions` raw JSON viewer) · resolution panel (form) · notes trail.
- **Tabs**: none.
- **Data displayed**: `GET /internal/v1/reconciliation/items/{id}` — item fields per docs/03 §4 (`provider_ref, amount, matched_charge_id, status, resolved_by, resolution_note`), embedded charge (docs/02 §2.1 shape) and raw provider payloads.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Type de résolution | select : Rapproché manuellement / Erreur provider (ticket ouvert) / Ajustement requis / Doublon — ignorer | requis | vide | « Choisissez un type de résolution. » |
| Charge à rapprocher | autocomplete charge id | requis si « Rapproché manuellement » ; montant identique sinon bloqué | vide | « Le montant de la charge ne correspond pas à la ligne provider. » |
| Note de résolution | texte long | requis, ≥ 20 caractères | vide | « Expliquez la résolution (20 caractères minimum). » |
| N° de ticket provider | texte | requis si « Erreur provider » | vide | « Indiquez la référence du ticket ouvert chez le provider. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Résoudre | `circle-check` | finance-admin | Ouvre OM-08 (récapitulatif) ; si « Ajustement requis », OM-08 enchaîne sur O-16 pré-rempli (règle DC) ; sinon `POST /internal/v1/reconciliation/items/{id}/resolve` |
| M'assigner | `user-round` | finance-admin, ops | `POST /internal/v1/reconciliation/items/{id}/claim` |
| Ajouter une note | `pencil` | finance-admin, ops | Ouvre OM-14 → `POST /internal/v1/reconciliation/items/{id}/notes` |
| Re-interroger le provider | `rotate-cw` | ops | OM-07 sur la charge liée (si existante) |

- **Modals/drawers/sheets opened**: OM-07, OM-08, OM-14.
- **States**: loading; error; permission-denied (ops: resolution panel disabled avec texte « Résolution réservée à finance-admin ») ; resolved state: panels figés + bannière success « Résolu par {nom} le {date} — {type} » ; adjustment-pending banner warning-600 outlined « Ajustement lié en attente de seconde approbation » → O-17.
- **Events/notifications triggered**: résolution → bell à l'importeur du rapport ; ajustement créé → flux DC (O-17) ; tout → `audit_logs`.

### O-11 — Fournisseurs / Providers Health
- **Route**: `/providers` · **Icon**: lucide `activity` · **Access**: all roles read; circuit-breaker ops-admin + DC · **Purpose**: monitor MTN/Orange health, float balances vs ledger, and trip circuit-breakers under dual control.
- **Layout zones**: header · per-provider cards (MTN MoMo, Orange Money): statut (`operational|degraded|down`, source `GET /channels` docs/02 §2.8), success rate 1 h/24 h, latence p95, âge médian des pending, dernier résultat de charge synthétique · **float table** · **synthetic checks table** · circuit-breaker panel per provider (collections / disbursements switches, état + historique).
- **Tabs**: none.
- **Data displayed**: metrics from `GET /internal/v1/providers/health`; float table: provider, solde ledger `provider_float:mtn|orange` (docs/03 chart of accounts, somme des postings), solde déclaré telco (dernier relevé), Δ (AmountText signé, danger si ≠ 0), dernier rapprochement; synthetic checks table: horodatage, provider, direction, durée, résultat StatusBadge (charges synthétiques via numéros magiques docs/02 §3 en environnement test + sonde prod 1 FCFA interne).
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Basculer le disjoncteur | `octagon-pause` | ops-admin (maker) + 2e ops-admin (règle DC) | Ouvre OM-09 → `POST /internal/v1/providers/{provider}/breaker_requests` ; à l'approbation : `GET /channels` renvoie `down` pour le sens coupé, bannières marchands (« Les paiements {provider} sont suspendus temporairement »), création de charges/payouts sur ce canal → `channel_unavailable` (docs/02 §1) |
| Lancer une vérification synthétique | `activity` | ops | `POST /internal/v1/providers/{provider}/synthetic_check` ; résultat en ligne dans la table |

- **Modals/drawers/sheets opened**: OM-09.
- **States**: loading; error; permission-denied (bouton disjoncteur masqué); breaker-pending banner warning-600 outlined « Demande de coupure {provider} ({sens}) en attente de seconde approbation — demandée par {nom} il y a {durée} » avec bouton Approuver (→ OM-11, si rôle et ≠ maker) ; breaker-active banner danger-600 « {provider} {sens} coupé depuis {date} par {maker}/{checker} — motif : {motif} » + bouton Rétablir (même flux DC).
- **Events/notifications triggered**: coupure approuvée → e-mail + bell tout le staff ops, bannière côté marchands (docs/08 A2 provider health strip) ; Δ float ≠ 0 → item automatique dans O-09.

### O-12 — Risque / Risk
- **Route**: `/risk` · **Icon**: lucide `siren` · **Access**: compliance, ops-admin (ops: read-only) · **Purpose**: manage detection rules and review everything they flag.
- **Layout zones**: header · tab strip · tab content · aside (dossiers ouverts par ancienneté, règles les plus déclenchées 7 j).
- **Tabs**:
  1. **Règles** — rules table: nom, description FR, seuil (éditable en ligne pour compliance : ex. « > {n} transactions / {fenêtre} », « montant > {x} FCFA », « ≥ {n} bénéficiaires nouveaux / jour »), sévérité (info/moyenne/haute), active (toggle), déclenchements 7 j. Seuil edit → confirm inline « Appliquer le nouveau seuil ? » ; `PATCH /internal/v1/risk/rules/{id}`.
  2. **Marchands signalés** — flagged merchants table: marchand, règle, valeur observée vs seuil, date, SLA chip, statut (nouveau/en dossier/écarté).
  3. **Transactions signalées** — flagged transactions table: TxRow + règle + score ; row → O-08 ; action rapide « Écarter » / « Joindre à un dossier ».
  4. **Dossiers** — cases table: n° dossier, marchand, ouvert le, sévérité, assigné, SLA chip (72 h), statut (ouvert / en enquête / clos — fondé / clos — non fondé / STR déposé).
- **Data displayed**: `GET /internal/v1/risk/rules`, `/internal/v1/risk/flags?target=merchant|transaction`, `/internal/v1/risk/cases`; merchant `risk_flags` jsonb per docs/03.
- **Inputs**: none at page level (rule edits inline as described; validations: seuil entier > 0, fenêtre ∈ {1h, 24h, 7j}; erreur « Le seuil doit être un entier positif. »).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir un dossier | `siren` | compliance | `POST /internal/v1/risk/cases` depuis un signalement (le signalement y est joint) → O-13 |
| Écarter le signalement | `circle-x` | compliance | `POST /internal/v1/risk/flags/{id}/dismiss` avec motif court requis (« Motif requis pour écarter. ») |
| Activer/désactiver une règle | `settings` | compliance | `PATCH /internal/v1/risk/rules/{id}` `{active}` ; confirm si désactivation « Cette règle ne signalera plus rien. Confirmer ? » |

- **Modals/drawers/sheets opened**: none (dismissal is an inline confirm; cases open O-13).
- **States**: loading; empty (Signalés) « Aucun signalement en attente. » ; empty (Dossiers) « Aucun dossier de risque ouvert. » ; error; permission-denied (ops: tout en lecture seule).
- **Events/notifications triggered**: nouveau signalement sévérité haute → bell + e-mail compliance ; désactivation de règle → `audit_logs` + bell ops-admin.

### O-13 — Dossier de risque / Risk Case Detail
- **Route**: `/risk/cases/:id` · **Icon**: lucide `siren` · **Access**: compliance (full), ops-admin (read + note), ops (none) · **Purpose**: investigate one risk case end-to-end: evidence, notes, decision, escalation to STR.
- **Layout zones**: header (n° dossier, marchand → O-06, sévérité, statut, SLA chip, assigné) · evidence panel: **linked items table** (type: signalement/transaction/marchand/document, référence, ajouté par, date — chaque ligne cliquable vers O-08/O-06) · chronologie des notes (append-only, auteur, date, pièces jointes) · decision panel · aside : profil express du marchand (tier, volume 30 j, drapeaux).
- **Tabs**: none.
- **Data displayed**: `GET /internal/v1/risk/cases/{id}` — case fields, linked flags/transactions, notes; merchant summary from `GET /internal/v1/merchants/{id}`.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Décision de clôture | select : Clos — non fondé / Clos — fondé (mesures internes) / Escalade — dépôt STR | requis pour clôturer | vide | « Choisissez une décision avant de clôturer. » |
| Synthèse de l'enquête | texte long | requis, ≥ 100 caractères pour toute clôture | vide | « La synthèse doit détailler l'enquête (100 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ajouter une note / pièce | `pencil` | compliance, ops-admin | Ouvre OM-14 → `POST /internal/v1/risk/cases/{id}/notes` (pièces : PDF/PNG/JPG ≤ 10 Mo) |
| Joindre une transaction | `arrow-left-right` | compliance | Autocomplete charge/payout id → `POST /internal/v1/risk/cases/{id}/items` |
| Clôturer le dossier | `circle-check` | compliance | Confirm restating décision ; `POST /internal/v1/risk/cases/{id}/close` ; si « Escalade — dépôt STR » → redirige vers O-14 pré-rempli |
| Construire le pack STR | `download` | compliance | Navigue vers O-14 avec le dossier chargé |

- **Modals/drawers/sheets opened**: OM-14.
- **States**: loading; error; permission-denied (ops-admin : decision panel masqué) ; closed state : tout figé + bannière « Dossier clos ({décision}) par {nom} le {date} » ; STR-filed badge info-600 « STR déposé le {date} — pack #{id} ».
- **Events/notifications triggered**: clôture → bell ops-admin ; escalade STR → e-mail responsable conformité ; tout → `audit_logs`.

### O-14 — Pack d'export STR / STR Export Pack Builder
- **Route**: `/risk/cases/:id/str-pack` · **Icon**: lucide `download` · **Access**: compliance only · **Purpose**: assemble the evidence pack for an ANIF suspicious-transaction report (STR) from a risk case.
- **Layout zones**: header (dossier lié → O-13, marchand) · builder form (left) · **selected-items table** (right : élément, type, période couverte, inclus ✓) · footer génération.
- **Tabs**: none.
- **Data displayed**: case items pre-checked from `GET /internal/v1/risk/cases/{id}`; pack contents preview: profil marchand + documents KYB, transactions sélectionnées (export docs/02-shaped objects), écritures comptables liées, notes du dossier, chronologie.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Période couverte | plage de dates | requise ; englobe toutes les transactions cochées | bornes du dossier | « La période doit couvrir toutes les transactions sélectionnées. » |
| Motif de soupçon (résumé ANIF) | texte long | requis, 200–5 000 caractères | pré-rempli avec la synthèse du dossier | « Le résumé doit faire entre 200 et 5 000 caractères. » |
| Inclure les documents KYB | case à cocher | — | coché | — |
| Inclure les payloads provider bruts | case à cocher | — | décoché | — |
| Format | select : PDF + CSV (zip) | requis | PDF + CSV | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Générer le pack | `download` | compliance | `POST /internal/v1/risk/cases/{id}/str_packs` → génération async ; bell + e-mail quand prêt ; téléchargement via URL signée 24 h ; marque le dossier « STR déposé » après confirmation manuelle |
| Marquer comme déposé | `circle-check` | compliance | Confirm avec n° d'accusé ANIF (texte requis) ; `POST /internal/v1/risk/str_packs/{packId}/mark_filed` |
| Aperçu du pack | `book-open` | compliance | Rendu PDF inline (viewer) sans génération finale |

- **Modals/drawers/sheets opened**: none (confirms inline).
- **States**: loading; generating state (progress « Génération du pack… {n}/{total} éléments ») ; error génération « La génération a échoué (req {request_id}). Réessayer » ; permission-denied full-page pour non-compliance « Accès réservé à l'équipe conformité. » ; filed state : lecture seule + badge « Déposé le {date} — accusé {n°} ».
- **Events/notifications triggered**: génération/téléchargement/dépôt → `audit_logs` ; pack prêt → bell + e-mail au demandeur.

### O-15 — Ajustements / Adjustments
- **Route**: `/adjustments` · **Icon**: lucide `book-text` · **Access**: finance-admin, ops-admin (others: read-only) · **Purpose**: list every manual journal entry with its full dual-control audit trail.
- **Layout zones**: header + « Nouvel ajustement » button · toolbar (statut : `pending_approval` / approuvé / rejeté / expiré ; période ; compte ; créé par) · adjustments table.
- **Tabs**: none.
- **Data displayed**: table from `GET /internal/v1/adjustments` — id, montant (AmountText), comptes débités/crédités (résumé « merchant_available → adjustments »), motif, référence (`reference_type/reference_id`, docs/03 journal_entries), créé par, approuvé/rejeté par, StatusBadge (`pending_approval` outlined warning / `succeeded` pour approuvé / `failed` pour rejeté / `expired`).
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Nouvel ajustement | `book-text` | finance-admin | Navigue vers O-16 |
| Voir l'écriture | `book-open` | tous | Ouvre OM-10 en lecture seule (postings + trail maker/checker) |

- **Modals/drawers/sheets opened**: OM-10.
- **States**: loading; empty « Aucun ajustement. C'est une bonne nouvelle. » ; error; permission-denied (bouton créer masqué).
- **Events/notifications triggered**: none (creation/approval flows notify from O-16/O-17).

### O-16 — Nouvel ajustement / Adjustment Creation
- **Route**: `/adjustments/new` · **Icon**: lucide `book-text` · **Access**: finance-admin (maker) · **Purpose**: compose a manual balanced journal entry that will require a second approver before posting.
- **Layout zones**: form card (left) · live journal-entry preview (right — same rendering as OM-10, recalculé à chaque frappe, indicateur d'équilibre « Somme des écritures : 0 FCFA ✓ » ou danger « Écriture non équilibrée ») · footer submit.
- **Tabs**: none.
- **Data displayed**: account picker options from `GET /internal/v1/ledger/accounts` (chart of accounts docs/03 §3 : `provider_float:mtn|orange`, `merchant_available`, `merchant_pending`, `merchant_payout_wallet`, `platform_fees`, `settlement_clearing`, `adjustments`) ; merchant autocomplete for merchant-owned accounts ; current balance of each picked account shown under the picker.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Compte à débiter | select compte (+ marchand si compte marchand) | requis ; ≠ compte crédité | vide | « Choisissez le compte à débiter. » / « Débit et crédit doivent différer. » |
| Compte à créditer | select compte (+ marchand) | requis | vide | « Choisissez le compte à créditer. » |
| Montant | entier FCFA | > 0 ; ≤ 50 000 000 FCFA (au-delà : bloqué, procédure exceptionnelle hors console) | vide | « Montant invalide — entier positif, maximum 50 000 000 FCFA. » |
| Motif | select : Correction d'écart de réconciliation / Geste commercial / Correction d'erreur interne / Reprise sur fraude / Autre | requis | pré-rempli si venu de O-10 | « Choisissez un motif. » |
| Référence liée | texte (charge id, item de réconciliation, ticket) | requis si motif = correction d'écart | pré-rempli depuis O-10 | « Indiquez la référence de l'écart ou du ticket. » |
| Description | texte long | requis, ≥ 20 caractères ; figurera dans journal_entries.description | vide | « Décrivez précisément l'ajustement (20 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Soumettre pour approbation | `check-check` | finance-admin | Confirm OM-10 (aperçu final, restitue montant + comptes) puis `POST /internal/v1/adjustments` → statut `pending_approval`, apparaît dans O-17, bell + e-mail aux finance-admin/ops-admin éligibles (≠ maker) ; expire à 72 h (règle DC) |
| Annuler | `circle-x` | finance-admin | Retour O-15, brouillon abandonné (confirm si champs saisis « Abandonner cet ajustement ? ») |

- **Modals/drawers/sheets opened**: OM-10 (preview/confirm).
- **States**: loading (chart of accounts); error soumission « Échec de la soumission (req {request_id}). Vos saisies sont conservées. » ; permission-denied full-page ; unbalanced state (submit disabled) ; insufficient-balance warning non bloquant « Le compte débité passera négatif ({solde} → {nouveau solde}). Confirmez en connaissance de cause. »
- **Events/notifications triggered**: soumission → bell + e-mail approbateurs ; `audit_logs`.

### O-17 — Approbations / Dual-Control Approvals
- **Route**: `/approvals` · **Icon**: lucide `check-check` · **Access**: any role with approver rights on ≥ 1 request type (ajustements & limites : finance-admin, ops-admin ; suspensions, disjoncteurs & paliers 2 : ops-admin) · **Purpose**: single queue where second approvers action every pending dual-control request.
- **Layout zones**: header (count + SLA 4 h summary) · toolbar (type : ajustement / limites / suspension / disjoncteur / palier 2 ; demandeur ; tri SLA) · requests table · aside « Mes demandes » (as maker : statut de mes propres demandes, non approuvables par moi).
- **Tabs**: none.
- **Data displayed**: `GET /internal/v1/approval_requests?status=pending` — type, résumé FR (« Ajustement 250 000 FCFA — merchant_available → adjustments — Boulangerie Ngong »), demandeur, créé le, SLA chip (4 h), expire à. Rows where maker = me are greyed with tag « Votre demande — un autre approbateur est requis ».
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Examiner & décider | `check-check` | approbateur éligible ≠ demandeur | Ouvre OM-11 (détail complet + OM-10 si monétaire) |
| Voir l'objet lié | `search` | tous approbateurs | Navigue vers O-04 / O-06 / O-11 / O-15 selon le type |
| Annuler ma demande | `circle-x` | demandeur uniquement | Confirm « Annuler cette demande ? » → `POST /internal/v1/approval_requests/{id}/cancel` |

- **Modals/drawers/sheets opened**: OM-10, OM-11.
- **States**: loading; empty « Aucune approbation en attente. » `circle-check` ; error; permission-denied full-page « Vous n'avez de droit d'approbation sur aucun type de demande. » ; expiring-soon rows (SLA > 75 %) surlignées warning.
- **Events/notifications triggered**: approbation/rejet → bell + e-mail au demandeur ; expiration → bell demandeur « Votre demande a expiré sans seconde approbation. » ; tout → `audit_logs`.

### O-18 — Journal d'audit / Audit Log
- **Route**: `/audit` · **Icon**: lucide `scroll-text` · **Access**: ops-admin, compliance (others: own actions only) · **Purpose**: search the append-only log of every staff action.
- **Layout zones**: toolbar (période, staff, type d'action, marchand concerné, texte libre) · log table (horodatage `fr-CM`, staff, action, objet lié cliquable, adresse IP, détail JSON expandable) · footer pagination cursor.
- **Tabs**: none.
- **Data displayed**: `GET /internal/v1/audit_logs` (append-only `audit_logs`, docs/03) — chaque connexion, lecture PII révélée, décision KYB, impersonation, ajustement, approbation, export.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Exporter | `download` | ops-admin, compliance | CSV `GET /internal/v1/audit_logs/export` (borné à 90 jours par export ; l'export lui-même est journalisé) |

- **Modals/drawers/sheets opened**: none.
- **States**: loading; empty « Aucune action sur cette période. » ; error; permission-denied (rôles non habilités : filtre staff verrouillé sur soi-même avec note « Vous ne voyez que vos propres actions. »).
- **Events/notifications triggered**: export → `audit_logs` (méta-journalisation).

### O-19 — Personnel & rôles / Staff & Roles Admin
- **Route**: `/staff` · **Icon**: lucide `users-round` · **Access**: ops-admin (others: read own profile) · **Purpose**: manage staff accounts, ops roles and hardware-key enrollment.
- **Layout zones**: header + « Inviter » button · staff table (nom, e-mail, rôle `shield` ops/ops-admin/compliance/finance-admin, clés de sécurité enrôlées (n + modèle), dernière connexion, statut actif/désactivé) · aside : matrice des rôles (récapitulatif lecture seule des droits par rôle, celle de ce document).
- **Tabs**: none.
- **Data displayed**: `GET /internal/v1/staff` — users, roles, WebAuthn credentials metadata (nickname, enrôlée le, dernière utilisation), statut.
- **Inputs**: none (invite/edit via OM-12).
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Inviter un membre | `users-round` | ops-admin | Ouvre OM-12 → `POST /internal/v1/staff/invites` (e-mail @ijimpay.com uniquement ; le nouvel arrivant doit enrôler une clé à la première connexion) |
| Changer le rôle | `shield` | ops-admin | Ouvre OM-12 (mode édition) ; passage vers/depuis ops-admin exige la règle DC (`POST /internal/v1/staff/{id}/role_change_requests`) |
| Désactiver le compte | `octagon-pause` | ops-admin | Confirm restating nom + rôle ; `POST /internal/v1/staff/{id}/deactivate` ; sessions révoquées immédiatement ; impossible de désactiver le dernier ops-admin (« Impossible : dernier ops-admin actif. ») |

- **Modals/drawers/sheets opened**: OM-11 (pour les changements de rôle DC), OM-12.
- **States**: loading; empty n/a (self toujours listé) ; error; permission-denied (non ops-admin : page réduite à « Mon profil » + gestion de ses propres clés — l'enrôlement d'une nouvelle clé requiert une clé existante ou un ops-admin) ; invite-pending rows en warning outlined « Invitation envoyée — en attente de première connexion ».
- **Events/notifications triggered**: invitation → e-mail ; changement de rôle / désactivation → e-mail à l'intéressé + `audit_logs` ; enrôlement/révocation de clé → e-mail sécurité.

### O-20 — Paramètres ops / Ops Settings
- **Route**: `/settings` · **Icon**: lucide `settings` · **Access**: ops-admin (compliance : lecture + SLA conformité) · **Purpose**: configure queue SLAs, security policy and staff notification routing.
- **Layout zones**: header · tab strip · tab content form · footer save bar (sticky, « Enregistrer » disabled tant qu'aucun changement).
- **Tabs**:
  1. **SLA des files** — inputs ci-dessous ; chaque changement historisé (qui/quand/ancienne valeur).
  2. **Sécurité** — durée de session (heures), exigence de ré-authentification WebAuthn pour approbations (toujours actif, lecture seule), liste d'IP autorisées (CIDR, une par ligne), délai d'expiration DC (24 h fixe sauf ajustements 72 h — lecture seule avec note explicative).
  3. **Notifications staff** — matrice événement ops × canal (bell / e-mail) par rôle : signalement risque haute sévérité, écart > seuil montant, demande DC créée, DC expirée, disjoncteur activé, pack STR prêt.
- **Data displayed**: `GET /internal/v1/settings` ; change history table (paramètre, ancienne → nouvelle valeur, par, le).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| SLA — File KYB (heures ouvrées) | entier | 4–72 | 24 | « Entre 4 et 72 heures. » |
| SLA — Écarts de réconciliation (heures) | entier | 8–168 | 48 | « Entre 8 et 168 heures. » |
| SLA — Dossiers risque (heures) | entier | 24–336 | 72 | « Entre 24 et 336 heures. » |
| SLA — Approbations DC (heures) | entier | 1–24 | 4 | « Entre 1 et 24 heures. » |
| Durée de session (heures) | entier | 1–12 | 8 | « Entre 1 et 12 heures. » |
| IP autorisées | liste CIDR | CIDR valides ; ≥ 1 ligne | bureau + VPN | « CIDR invalide à la ligne {n}. » |
| Seuil d'alerte écart (FCFA) | entier | ≥ 100 000 | 1 000 000 | « Minimum 100 000 FCFA. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer | `circle-check` | ops-admin | Confirm listant chaque changement (ancienne → nouvelle valeur) ; `PATCH /internal/v1/settings` ; toast « Paramètres enregistrés » |

- **Modals/drawers/sheets opened**: none (confirm inline).
- **States**: loading; error save « Échec de l'enregistrement (req {request_id}). Vos modifications sont conservées à l'écran. » ; permission-denied (compliance : seuls les SLA sont éditables, reste en lecture seule ; autres rôles : page entière lecture seule avec bannière « Lecture seule — réservé à ops-admin ») ; unsaved-changes guard « Modifications non enregistrées — quitter quand même ? ».
- **Events/notifications triggered**: tout changement → `audit_logs` + bell ops-admin.

### O-21 — Page introuvable / 404
- **Route**: toute route inconnue de `ops.ijimpay.com` · **Icon**: lucide `search` (illustration de la banque d'illustrations, pas une icône agrandie) · **Access**: all roles (staff authentifié ; sinon redirection O-01) · **Purpose**: dead-end recovery (miroir de D-45, docs/12).
- **Layout zones**: illustration centrée + titre + actions ; sans sidebar si la session est invalide.
- **Tabs**: none.
- **Data displayed**: aucune. Copy : « Page introuvable. Cette page n'existe pas ou a été déplacée. »
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Retour à l'accueil | `layout-dashboard` (primary) | tous | → `/` (O-02) |
| Recherche globale | `search` | tous | Ouvre le ⌘K (marchand, ref charge, téléphone, provider_ref) |

- **Modals/drawers/sheets opened**: none.
- **States**: statique.
- **Events/notifications triggered**: none.

### O-22 — Erreur serveur / 500
- **Route**: rendue sur erreur applicative irrécupérable · **Icon**: lucide `triangle-alert` · **Access**: all roles · **Purpose**: fail gracefully with a support handle (miroir de D-46, docs/12).
- **Layout zones**: illustration + titre + `request_id` monospace copiable.
- **Tabs**: none.
- **Data displayed**: `request_id` de la réponse en erreur. Copy : « Une erreur est survenue de notre côté. Réessayez ; si le problème persiste, transmettez ce code à l'équipe plateforme : req_8fK2… »
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Réessayer | `rotate-cw` (primary) | tous | Recharge la route courante |
| Copier le code | `copy` | tous | Presse-papiers + toast « Code copié » |

- **Modals/drawers/sheets opened**: none.
- **States**: statique.
- **Events/notifications triggered**: erreur remontée au monitoring interne (invisible) ; jamais dans `audit_logs` (pas une action staff).

### O-23 — Maintenance / Maintenance
- **Route**: servie par l'edge pendant une maintenance planifiée de la console · **Icon**: lucide `wrench` · **Access**: all roles · **Purpose**: planned downtime message (miroir de D-47, docs/12).
- **Layout zones**: illustration + titre + fenêtre horaire.
- **Tabs**: none.
- **Data displayed**: fenêtre de maintenance (config edge). Copy : « Maintenance en cours. La console ops revient vers {heure}. Les paiements marchands ne sont pas affectés. » + lien statut status.ijimpay.com.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Voir la page de statut | `activity` | tous | → status.ijimpay.com (nouvel onglet) |

- **Modals/drawers/sheets opened**: none.
- **States**: auto-refresh toutes les 60 s.
- **Events/notifications triggered**: none.

### O-24 — Session expirée / Forced logout
- **Route**: interception globale sur 401 (toast + redirection `/login?reason=expired`) · **Icon**: lucide `log-in` · **Access**: all roles · **Purpose**: expel an expired/revoked staff session (durée 8 h, O-01) without losing work context (miroir de D-48, docs/12) — remplace le simple toast session-expired de O-01, qui reste l'écran d'atterrissage.
- **Layout zones**: toast danger persistant « Session expirée — reconnectez-vous. » puis page O-01 avec bandeau info ; le deep-link d'origine est mémorisé et restauré après reconnexion (SSO + clé de sécurité complets exigés — pas de reprise de session).
- **Tabs**: none.
- **Data displayed**: aucune.
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Se reconnecter | `log-in` (primary, dans le toast) | tous | → `/login` (O-01) ; retour à la page d'origine après succès |

- **Modals/drawers/sheets opened**: none.
- **States**: brouillons en cours (O-16 ajustement, formulaires de résolution O-10) conservés en localStorage 15 min et restaurés avec toast « Brouillon restauré. » — jamais pour les champs sensibles (motifs de suspension, décisions DC).
- **Events/notifications triggered**: `audit_logs` session_expired (expulsion serveur).

---

## Modals & Drawers/Sheets

### OM-01 — Rejeter le dossier KYB / KYB Rejection (reason-template library)
- **Opened from**: O-04 · **Type**: modal (560px) · **Roles**: ops, compliance (per O-04 access).
- **Contents**: merchant name + palier demandé restated; template picker; message preview exactly as the merchant will receive it (e-mail + in-dashboard, FR shown, EN toggle `globe`).
- **FR reason-template library** (code → texte envoyé au marchand ; {placeholders} remplis avant envoi) :
  - `doc_illisible` — « Le document « {document} » est illisible ou incomplet. Merci de le re-photographier à plat, en pleine lumière, puis de le renvoyer depuis Paramètres → Vérification. »
  - `rccm_expire` — « Votre document RCCM est expiré ou ne correspond pas au nom légal déclaré ({nom_légal}). Merci de fournir un extrait RCCM en cours de validité. »
  - `identite_non_concordante` — « La pièce d'identité fournie ne correspond pas au dirigeant déclaré. Merci de fournir la pièce d'identité de {nom_dirigeant}, ou de mettre à jour le dirigeant déclaré. »
  - `compte_non_prouve` — « La preuve de propriété du compte de règlement ({canal} {numéro_masqué}) est insuffisante. Merci de fournir une capture du compte affichant le nom du titulaire. »
  - `activite_non_eligible` — « L'activité déclarée n'est pas éligible à nos services à ce jour. Votre compte reste utilisable en mode test. »
  - `adresse_invalide` — « L'adresse de l'entreprise n'a pas pu être vérifiée. Merci de fournir un justificatif (facture ENEO/Camwater ou contrat de bail) de moins de 3 mois. »
  - `document_manquant` — « Il manque le document « {document} » pour valider le palier {palier}. Merci de l'ajouter depuis Paramètres → Vérification. »
  - `autre` — texte libre (validation ci-dessous).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Motif(s) de rejet | multi-select modèles ci-dessus | ≥ 1 requis | vide | « Sélectionnez au moins un motif. » |
| Complément (envoyé au marchand) | texte long | requis si `autre` ; ≤ 1 000 caractères ; relu dans l'aperçu | vide | « Précisez le motif (obligatoire pour « Autre »). » |
| Documents concernés | multi-select docs du dossier | requis si motif documentaire | pré-coché selon motifs | « Indiquez le(s) document(s) concerné(s). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer le rejet | `circle-x` | ops, compliance | `POST /internal/v1/kyb/reviews/{id}/reject` `{template_codes[], custom_text, document_ids[]}` ; dossier clos ; marchand notifié (push+e-mail+SMS) ; les documents rejetés passent « à renvoyer » côté marchand (docs/08 A10 Verification) |
| Annuler | `arrow-left` | ops, compliance | Ferme sans effet |

- **Confirm rules**: bouton d'envoi jamais focus par défaut ; l'aperçu du message doit avoir été affiché (scroll) avant activation du bouton.

### OM-02 — Approuver le palier / Approve Tier
- **Opened from**: O-04 · **Type**: modal · **Roles**: ops, compliance (maker) ; **palier 2 : règle DC** — approbateur ops-admin ≠ maker (F-047, docs/08 §B). Une seule règle : Palier 1 = décision simple ; tout octroi de Palier 2 (limites standard ou personnalisées) = dual-control via O-17.
- **Contents**: restates merchant + checklist state (toutes concordances ✓ requises sinon modal refuse de s'ouvrir : toast « Complétez la liste de vérification. »); limit preview for chosen tier.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Palier accordé | select 1 / 2 | ≤ palier demandé | palier demandé | « Le palier accordé ne peut dépasser le palier demandé. » |
| Limites appliquées | select : Limites standard du palier / Personnalisées | personnalisées → ouvre les champs d'OM-04 ; incluses dans la même demande DC que le palier (jamais une seconde demande séparée) | standard | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer l'approbation | `circle-check` | ops, compliance | **Palier 1** : `POST /internal/v1/kyb/reviews/{id}/approve` `{tier, limits}` — merchants.tier mis à jour immédiatement, notifications marchand (N-04). **Palier 2** : le même appel crée une demande d'approbation `pending_approval` (O-17, approbateur ops-admin ≠ maker, règle DC — F-047) couvrant palier **et** limites ; rien n'est appliqué avant la seconde approbation, le marchand reste à son palier actuel ; à l'approbation : tier 2 + limites appliqués atomiquement, notifications marchand (N-06) |
| Annuler | `arrow-left` | ops, compliance | Ferme |

- **Confirm rules**: restates « Palier {n} — plafond encaissement {x} FCFA/jour, paiements sortants {y} FCFA/jour » ; pour le palier 2, le bouton est libellé « Soumettre pour approbation » et le récapitulatif ajoute « Un ops-admin devra approuver (règle DC). » ; bouton non focus par défaut.

### OM-03 — Demander des informations / Request More Info (KYB)
- **Opened from**: O-04 · **Type**: modal · **Roles**: ops, compliance (per O-04 access).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Éléments demandés | multi-select (nouveau document RCCM / pièce d'identité / preuve de compte / justificatif d'adresse / autre) | ≥ 1 | vide | « Sélectionnez au moins un élément. » |
| Message au marchand | texte long | requis, 20–1 000 caractères, FR | modèle pré-rempli selon éléments | « Rédigez le message (20 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer la demande | `message-square` | ops, compliance | `POST /internal/v1/kyb/reviews/{id}/request_info` ; dossier → « infos demandées », SLA en pause jusqu'à nouvel upload marchand (reprend automatiquement) ; notifications marchand |
| Annuler | `arrow-left` | ops, compliance | Ferme |

- **Confirm rules**: aperçu du message obligatoire avant envoi.

### OM-04 — Modifier les limites / Limit Change (dual-control)
- **Opened from**: O-06 (tab Limites), OM-02 · **Type**: modal · **Roles**: ops-admin maker ; approbation DC (finance-admin ou ops-admin ≠ maker) requise si palier > 1 ou toute hausse > 20 %.
- **Contents**: current vs proposed table (limite, valeur actuelle, nouvelle valeur, Δ %).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Plafond encaissement / jour | entier FCFA | > 0 ; ≤ plafond réglementaire du palier ×2 | valeur actuelle | « Dépasse le plafond autorisé pour ce palier. » |
| Plafond paiements sortants / jour | entier FCFA | idem | valeur actuelle | idem |
| Plafond par transaction | entier FCFA | > 0 ; ≤ plafond journalier | valeur actuelle | « Ne peut dépasser le plafond journalier. » |
| Motif du changement | texte long | requis, ≥ 20 caractères | vide | « Justifiez le changement (20 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Soumettre | `check-check` | ops-admin | Si DC requis : `POST /internal/v1/merchants/{id}/limit_change_requests` → O-17, bannière pending sur O-06 ; sinon application immédiate `PATCH /internal/v1/merchants/{id}/limits` |
| Annuler | `arrow-left` | ops-admin | Ferme |

- **Confirm rules**: le récapitulatif Δ % est toujours affiché ; toute baisse est immédiate (protection), toute hausse suit la règle DC.

### OM-05 — Suspendre / Réactiver le marchand / Suspend–Reactivate (dual-control)
- **Opened from**: O-06 · **Type**: modal danger · **Roles**: ops-admin maker + ops-admin checker (règle DC, jamais le même).
- **Contents**: restates merchant name, solde disponible, volume 30 j ; texte d'impact dynamique selon la portée choisie (F-048) — **Encaissements uniquement** : « Les créations de charges renvoient `permission_denied` ; paiements sortants et règlements continuent. » · **Tout (encaissements + paiements sortants)** : « Encaissements et paiements sortants bloqués ; les paiements sortants sont masqués côté marchand avec bannière ; les règlements continuent. » · **Gel total (règlements inclus)** : « API entièrement bloquée, dashboard en lecture seule, règlements gelés. » Dans tous les cas : soldes gelés, jamais saisis (aucune écriture ledger).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Motif | select : Fraude suspectée / Demande d'autorité / Impayés / Violation des CGU / Autre | requis | vide | « Choisissez un motif. » |
| Détail du motif | texte long | requis, ≥ 30 caractères | vide | « Détaillez le motif (30 caractères minimum). » |
| Portée de la suspension | select : Encaissements uniquement / Tout (encaissements + paiements sortants) / Gel total (règlements inclus) | requise | Gel total (règlements inclus) | « Choisissez la portée de la suspension. » |
| Saisir le nom du marchand pour confirmer | texte | doit égaler le nom exact | vide | « Le nom saisi ne correspond pas. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Demander la suspension | `octagon-pause` | ops-admin | `POST /internal/v1/merchants/{id}/suspension_requests` `{scope}` → `pending_approval`, O-17 ; rien ne change tant que non approuvé (sauf urgence : cocher « Gel immédiat 4 h » qui coupe l'API à titre conservatoire en attendant l'approbation, auto-levé si rejet/expiration) ; à l'approbation, seule la portée demandée est bloquée (suspension partielle : les encaissements continuent — F-048) ; N-26 au Owner mentionne la portée `{scope}` |
| Annuler | `arrow-left` | ops-admin | Ferme |

- **Confirm rules**: name-typing confirmation obligatoire ; bouton danger, jamais focus par défaut ; la variante Réactiver réutilise ce modal (motif select réduit à « Résolution du motif initial / Décision interne », sans name-typing).

### OM-06 — Impersonation lecture seule / Read-only Impersonation
- **Opened from**: O-06 · **Type**: modal · **Roles**: ops-admin.
- **Contents**: « Vous allez ouvrir le dashboard de {marchand} en lecture seule pendant 30 minutes. Chaque page consultée est journalisée et visible par le marchand sur demande. »
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Motif de la consultation | texte | requis, ≥ 15 caractères | vide | « Indiquez le motif (15 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Ouvrir la session lecture seule | `eye` | ops-admin | `POST /internal/v1/merchants/{id}/impersonate` `{reason}` → URL app.ijimpay.com à jeton unique 30 min ; nouvelle fenêtre ; toutes les mutations y sont désactivées côté serveur |
| Annuler | `arrow-left` | ops-admin | Ferme |

- **Confirm rules**: re-authentification WebAuthn exigée avant ouverture.

### OM-07 — Re-interroger le provider / Manual Re-poll
- **Opened from**: O-08, O-10 · **Type**: confirm modal · **Roles**: ops.
- **Contents**: restates transaction id, provider, `poll_count`, `last_polled_at` ; note « Limité à 1 re-interrogation / minute / transaction. »
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Re-interroger maintenant | `rotate-cw` | ops | `POST /internal/v1/transactions/{id}/repoll` ; spinner inline ; résultat affiché (« Statut provider : {statut} ») ; si le statut change, la machine d'états normale s'applique (webhooks marchands compris) |
| Annuler | `arrow-left` | ops | Ferme |

- **Confirm rules**: bouton désactivé si dernière interrogation < 60 s (« Réessayez dans {n} s »).

### OM-08 — Confirmer la résolution d'écart / Discrepancy Resolution Confirm
- **Opened from**: O-10 · **Type**: confirm modal · **Roles**: finance-admin.
- **Contents**: récapitulatif : type d'écart, montant, type de résolution choisi, note ; si « Ajustement requis » : encart « Un ajustement sera préparé — vous serez redirigé vers sa création ; il exigera une seconde approbation avant toute écriture. » ; si « Rapproché manuellement » : mini journal-entry preview (OM-10 embarqué) montrant l'écriture de rattrapage éventuelle (ex. charge succeeded tardive : `debit provider_float:{p} / credit merchant_pending + credit platform_fees`, docs/03 entrées canoniques).
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Case « J'ai vérifié les payloads provider » | case à cocher | requise | décochée | « Confirmez avoir vérifié les données provider. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer la résolution | `circle-check` | finance-admin | `POST /internal/v1/reconciliation/items/{id}/resolve` ; si ajustement requis → navigation O-16 pré-rempli (l'item reste « en résolution » jusqu'à approbation de l'ajustement) |
| Retour | `arrow-left` | finance-admin | Ferme, conserve le formulaire O-10 |

- **Confirm rules**: montant toujours restitué en toutes lettres FCFA ; bouton non focus par défaut.

### OM-09 — Disjoncteur provider / Circuit-breaker (dual-control)
- **Opened from**: O-11 · **Type**: modal danger · **Roles**: ops-admin maker + ops-admin checker (règle DC).
- **Contents**: impact statement dynamique : « Couper {provider} — {sens} bloquera immédiatement ~{n} transactions/heure (moyenne 7 j). Les marchands verront une bannière et recevront `channel_unavailable`. »
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Sens à couper | cases : Encaissements / Paiements sortants | ≥ 1 | vide | « Choisissez au moins un sens. » |
| Motif | select : Incident provider confirmé / Taux d'échec anormal / Maintenance annoncée / Demande du provider / Sécurité | requis | vide | « Choisissez un motif. » |
| Détail | texte long | requis, ≥ 20 caractères | vide | « Détaillez (20 caractères minimum). » |
| Rétablissement automatique après | select : jamais / 1 h / 4 h / 12 h | — | jamais | — |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Demander la coupure | `octagon-pause` | ops-admin | `POST /internal/v1/providers/{provider}/breaker_requests` → `pending_approval` (O-17, SLA 4 h mais notifié en urgence : bell + e-mail immédiats à tous les ops-admin) ; à l'approbation : effet immédiat (voir O-11) |
| Annuler | `arrow-left` | ops-admin | Ferme |

- **Confirm rules**: re-auth WebAuthn du maker à la soumission ET du checker à l'approbation ; le rétablissement suit le même flux DC (sauf rétablissement automatique programmé).

### OM-10 — Aperçu d'écriture comptable / Journal-Entry Preview
- **Opened from**: O-06 (tab Soldes), O-08, O-15, O-16, O-17/OM-11, OM-08 · **Type**: modal (also embedded as a live panel in O-16) · **Roles**: all (read); confirm variant per calling flow.
- **Contents**: entry header (kind — `charge_succeeded|payout_sent|fee|settlement|adjustment|refund|topup`, docs/03 —, description, reference_type/reference_id, created_by) ; **postings table** : Compte (nom + propriétaire : « merchant_available — Boulangerie Ngong ») | Débit | Crédit — amounts AmountText, débits négatifs à gauche, crédits à droite ; footer ligne « Somme : 0 FCFA ✓ » (success-600) ou « Somme : {Δ} FCFA — écriture non équilibrée » (danger-600) ; balances après écriture (solde avant → après par compte).
- **Inputs**: none.
- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Confirmer (variante confirm uniquement) | `check-check` | rôle du flux appelant | Retourne le contrôle au flux appelant (soumission O-16 ou approbation OM-11) ; désactivé tant que la somme ≠ 0 |
| Fermer | `arrow-left` | tous | Ferme |

- **Confirm rules**: this preview is MANDATORY before every money action in the console (règle DC point 6) — no ops flow may post to the ledger without having rendered OM-10.

### OM-11 — Décision de seconde approbation / Second-Approver Decision
- **Opened from**: O-17, O-11 (banner), O-19 (role changes) · **Type**: modal · **Roles**: eligible approver per request type, ≠ maker (server-enforced).
- **Contents**: full request detail restated (type, objet lié cliquable, demandeur, date, motif du maker in extenso) ; pour tout type monétaire : OM-10 embarqué ; pour suspension/disjoncteur : impact statement ; bandeau « En approuvant, vous engagez votre responsabilité au même titre que le demandeur. »
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Décision | radio : Approuver / Rejeter | requise | aucune (rien de présélectionné) | « Choisissez une décision. » |
| Motif du rejet | texte long | requis si Rejeter, ≥ 10 caractères | vide | « Motivez le rejet (10 caractères minimum). » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Valider ma décision | `check-check` | approbateur ≠ maker | WebAuthn re-auth obligatoire → `POST /internal/v1/approval_requests/{id}/approve` ou `/reject` ; approve exécute l'effet (écriture ledger / limites / suspension / disjoncteur) atomiquement ; bell + e-mail au demandeur |
| Fermer sans décider | `arrow-left` | approbateur | Ferme, la demande reste en file |

- **Confirm rules**: bouton jamais focus par défaut ; si la demande a expiré entre l'ouverture et la validation → erreur « Cette demande a expiré — plus rien à approuver. » ; si maker = approbateur (session partagée, etc.) → 403 « Le demandeur ne peut pas approuver sa propre demande. »

### OM-12 — Inviter / modifier un membre du personnel / Staff Invite–Edit
- **Opened from**: O-19 · **Type**: modal · **Roles**: ops-admin.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Adresse e-mail | email | domaine `@ijimpay.com` obligatoire ; unique | vide | « Adresse @ijimpay.com requise. » / « Ce membre existe déjà. » |
| Nom complet | texte | requis, 2–80 caractères | vide | « Indiquez le nom complet. » |
| Rôle | select : ops / ops-admin / compliance / finance-admin (avec descriptif d'un phrase par rôle) | requis | ops | « Choisissez un rôle. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Envoyer l'invitation / Enregistrer | `users-round` | ops-admin | Create : `POST /internal/v1/staff/invites` ; edit rôle simple : `PATCH /internal/v1/staff/{id}` ; edit vers/depuis ops-admin : crée une demande DC (`POST /internal/v1/staff/{id}/role_change_requests`) → O-17 |
| Annuler | `arrow-left` | ops-admin | Ferme |

- **Confirm rules**: attribution du rôle ops-admin restituée explicitement (« {nom} pourra approuver suspensions et disjoncteurs »).

### OM-13 — Importer un rapport provider / Provider Report Import
- **Opened from**: O-09 · **Type**: modal · **Roles**: finance-admin.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Provider | select mtn_momo / orange_money | requis | vide | « Choisissez le provider. » |
| Période couverte | plage de dates | requise ; pas de chevauchement avec un rapport déjà importé (avertissement non bloquant) | vide | « Période requise. » / avert. « Un rapport couvre déjà {dates} — le matching dédupliquera. » |
| Fichier | upload CSV/XLSX | ≤ 100 Mo ; colonnes obligatoires provider_ref, amount, phone, timestamp (mapping assisté si en-têtes différents) | vide | « Colonne obligatoire absente : {colonne}. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Importer et lancer le matching | `download` | finance-admin | `POST /internal/v1/reconciliation/reports` (multipart) ; barre de progression ; à la fin : résumé « {n} lignes — {m} correspondances — {k} écarts créés » avec lien vers les tabs d'écarts |
| Annuler | `arrow-left` | finance-admin | Ferme (annule l'upload en cours) |

- **Confirm rules**: le mapping de colonnes doit être validé écran par écran avant lancement.

### OM-14 — Ajouter une note / Add Note (with attachment)
- **Opened from**: O-06, O-10, O-13 · **Type**: drawer (right, 420px) · **Roles**: per opening page.
- **Inputs**:

| Champ (FR) | Type | Validation | Défaut | Message d'erreur (FR) |
|---|---|---|---|---|
| Note | texte long | requis, 3–5 000 caractères | vide | « La note doit contenir entre 3 et 5 000 caractères. » |
| Pièces jointes | upload multiple | PDF/PNG/JPG, ≤ 10 Mo chacune, ≤ 5 fichiers | aucune | « Fichier trop volumineux ({nom}) — 10 Mo maximum. » |

- **Actions**:

| Action (FR) | Icône | Rôle requis | Comportement |
|---|---|---|---|
| Enregistrer la note | `pencil` | rôle de la page appelante | `POST /internal/v1/{contexte}/{id}/notes` ; la note apparaît immédiatement dans la chronologie ; append-only, jamais modifiable ni supprimable |
| Annuler | `arrow-left` | idem | Ferme (confirm si texte saisi) |

- **Confirm rules**: none beyond the unsaved-text confirm.

---

## API additions needed

None of the following exists in docs/02 (which covers the merchant API only). All are internal, served at `https://api.ijimpay.com/internal/v1` (staff session auth + WebAuthn step-up where noted), to be specified in a future `docs/17-internal-api.md` (14–16 are already taken by the checkout pages, mobile screens and flows catalog):

- `GET /internal/v1/auth/sso/start` · `POST /internal/v1/auth/webauthn/verify` · `GET /internal/v1/staff/me`
- `GET /internal/v1/queues/summary`
- `GET /internal/v1/kyb/reviews` · `GET /internal/v1/kyb/reviews/{id}` · `POST /internal/v1/kyb/reviews/{id}/claim` · `POST /internal/v1/kyb/reviews/{id}/assign` · `POST /internal/v1/kyb/reviews/{id}/approve` · `POST /internal/v1/kyb/reviews/{id}/reject` · `POST /internal/v1/kyb/reviews/{id}/request_info` · `GET /internal/v1/kyb/reviews/export` · `GET /internal/v1/kyb/documents/{id}/download`
- `GET /internal/v1/merchants` · `GET /internal/v1/merchants/{id}` · `GET /internal/v1/merchants/export` · `GET /internal/v1/merchants/{id}/balances` · `GET /internal/v1/merchants/{id}/api-logs` · `POST /internal/v1/merchants/{id}/notes` · `POST /internal/v1/merchants/{id}/impersonate` · `POST /internal/v1/merchants/{id}/suspension_requests` · `POST /internal/v1/merchants/{id}/limit_change_requests` · `PATCH /internal/v1/merchants/{id}/limits`
- `GET /internal/v1/transactions` · `GET /internal/v1/transactions/{id}` · `POST /internal/v1/transactions/{id}/repoll` · `POST /internal/v1/webhook_deliveries/{id}/redeliver`
- `GET /internal/v1/reconciliation/reports` · `POST /internal/v1/reconciliation/reports` · `POST /internal/v1/reconciliation/reports/{id}/rematch` · `GET /internal/v1/reconciliation/items` · `GET /internal/v1/reconciliation/items/{id}` · `POST /internal/v1/reconciliation/items/{id}/claim` · `POST /internal/v1/reconciliation/items/{id}/resolve` · `POST /internal/v1/reconciliation/items/{id}/notes`
- `GET /internal/v1/providers/health` · `POST /internal/v1/providers/{provider}/synthetic_check` · `POST /internal/v1/providers/{provider}/breaker_requests`
- `GET /internal/v1/risk/rules` · `PATCH /internal/v1/risk/rules/{id}` · `GET /internal/v1/risk/flags` · `POST /internal/v1/risk/flags/{id}/dismiss` · `GET /internal/v1/risk/cases` · `POST /internal/v1/risk/cases` · `GET /internal/v1/risk/cases/{id}` · `POST /internal/v1/risk/cases/{id}/items` · `POST /internal/v1/risk/cases/{id}/notes` · `POST /internal/v1/risk/cases/{id}/close` · `POST /internal/v1/risk/cases/{id}/str_packs` · `POST /internal/v1/risk/str_packs/{id}/mark_filed`
- `GET /internal/v1/ledger/accounts` · `GET /internal/v1/adjustments` · `POST /internal/v1/adjustments`
- `GET /internal/v1/approval_requests` · `POST /internal/v1/approval_requests/{id}/approve` · `POST /internal/v1/approval_requests/{id}/reject` · `POST /internal/v1/approval_requests/{id}/cancel`
- `GET /internal/v1/audit_logs` · `GET /internal/v1/audit_logs/export`
- `GET /internal/v1/staff` · `POST /internal/v1/staff/invites` · `PATCH /internal/v1/staff/{id}` · `POST /internal/v1/staff/{id}/deactivate` · `POST /internal/v1/staff/{id}/role_change_requests`
- `GET /internal/v1/settings` · `PATCH /internal/v1/settings`

New error codes needed (internal API): `approver_is_maker`, `approval_request_expired`, `unbalanced_journal_entry`, `webauthn_required`, `last_ops_admin`.

Data-model additions needed (docs/03): `approval_requests` table (type, payload jsonb, maker_id, checker_id, status pending_approval|approved|rejected|expired|canceled, expires_at); `str_packs`; `risk_rules` / `risk_flags` / `risk_cases` (+ case items & notes); `staff_users` / `staff_webauthn_credentials`; `ops_settings`; `merchant_limits` (+ history); notes tables (merchant/recon/case, append-only).

## Icon additions needed

To be added to the canonical map in docs/07 §4 (one icon = one meaning):

- `scale` — reconciliation & discrepancies (already named in docs/08 §B, not yet in the docs/07 map)
- `eye` — read-only impersonation
- `siren` — risk (queue, cases, flags)
- `octagon-pause` — suspend / circuit-breaker (destructive pause, distinct from `clock` pending)
- `book-text` — ledger adjustments / manual journal entries
- `rotate-cw` — retry / re-poll / redeliver (used in docs/08 prose, absent from the docs/07 map)
- `wrench` — maintenance page (already used by docs/12 D-47; O-23 here)
- `log-in` — sign back in / reconnect (already used by docs/12 D-48; O-24 here — distinct from `log-out` sign out)

Also note: a 9th badge style `draft` (ink-500 outline) is used by O-15 listings only if adjustments ever expose drafts; v1 keeps the canonical 8 statuses and maps adjustment states onto them as specified in O-15.

---

```
INVENTORY: pages=24 tabs=19 modals=14 forms=21 tables=31 actions=90
```
