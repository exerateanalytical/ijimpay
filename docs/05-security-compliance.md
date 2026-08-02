# IjimPay — Security & Compliance Plan

---

## 1. Regulatory Path (Cameroon / CEMAC)

Payment aggregation in Cameroon is regulated under the CEMAC framework (Règlement n°04/18/CEMAC/UMAC/COBAC on payment services), supervised by **BEAC/COBAC**, with the national telecom regulator (ART) relevant to telco partnerships.

Options, in order of speed-to-market:

1. **Operate under a licensed partner** (bank or licensed PSP/telco umbrella) — fastest launch; revenue share; our brand on the API. Recommended for year 1.
2. **Own Payment Service Provider (établissement de paiement) license** — capital requirements, governance, reporting to COBAC; target from month 6 in parallel with operations under option 1.

Actions (start immediately, parallel to engineering):
- Retain a Cameroonian fintech lawyer; confirm licensing category and capital requirements.
- Open commercial/aggregator discussions with **MTN Cameroon** and **Orange Cameroon** (production API access requires contracts, KYC of our company, and often revenue commitments; sandbox ≠ production).
- Define fund-flow model with counsel: whose name holds the telco float, settlement accounts, safeguarding of merchant funds (segregated account requirement).
- Data protection: Cameroon Law No. 2010/012 (cybersecurity/cybercrime) and forthcoming data-protection rules; keep personal data processing documented; check whether ledgers must be hosted in-region.

## 2. KYC / KYB & AML Program

- **Merchant KYB tiers**: Tier 0 sandbox (email only) → Tier 1 (ID + business info, capped volumes) → Tier 2 (full RCCM/registration docs, proof of account ownership, beneficial owners) — limits enforced in code, visible to merchant.
- **Screening**: sanctions/PEP screening of merchants and beneficial owners at onboarding + periodic re-screen.
- **Transaction monitoring** (day one, simple rules → tune later): velocity per payer phone, structuring detection (many charges just under thresholds), sudden volume spikes, payout-only accounts, mismatch between declared activity and pattern. Flags → ops review queue; case notes retained.
- **Record keeping**: 5+ years for transactions and KYB files; STR (suspicious transaction report) process to ANIF (Cameroon's FIU) documented with counsel.
- Payer-side KYC is carried by the telcos (wallet holders are KYC'd by MTN/Orange) — we still monitor patterns.

## 3. Application Security

| Layer | Controls |
|---|---|
| API | TLS 1.2+ only; HSTS; secret-key auth with hashed storage; per-key rate limits; strict schema validation on every input; idempotency keys; no sequential IDs (ULIDs) |
| Webhooks (outbound) | HMAC-SHA256 + timestamp; per-endpoint secrets; retry with backoff |
| Provider callbacks (inbound) | IP allowlisting where telcos publish ranges + signature verification where offered + **never trust callbacks alone**: every callback triggers a status re-query to the provider before posting to ledger |
| Dashboard | 2FA (TOTP) mandatory for owner/admin/finance; session management; RBAC enforced server-side |
| Mobile app | User-session JWTs (never API secret keys), cert pinning, device binding for payout approval, FLAG_SECURE, root detection |
| Secrets | Cloud KMS/Vault; no secrets in env files in repos; telco credentials rotated per contract |
| Data | PII envelope-encrypted at rest; backups encrypted; least-privilege DB roles; ledger tables append-only at the grant level |
| Infra | Private subnets for DB/Redis; WAF on public edge; image scanning; dependency audit in CI (npm audit/osv-scanner); no SSH to prod (SSM/teleport) |

## 4. Internal Controls (fraud & insider risk)

- **Maker–checker** on: merchant payouts above threshold, manual ledger adjustments, KYB tier changes, refund overrides.
- Append-only `audit_logs` for every admin/ops action (who, what, before/after, why).
- Dual control on production deploys touching money paths; ledger invariants (balanced entries, non-negative merchant balances) enforced by DB constraints + nightly assertion job that pages on violation.
- Separation: ops console is a separate app with separate auth (SSO + hardware keys for staff), never the merchant dashboard with a flag.

## 5. Availability & Incident Response

- Provider health monitoring: synthetic test charges per provider every N minutes in production (small self-charges); status surfaced on `/channels`, dashboard banner, and app banner.
- Alerting: pending-charge age p95, provider error rate, webhook backlog, reconciliation mismatches, balance drift.
- Runbooks: telco outage (degrade gracefully, message merchants), stuck-pending storm, webhook flood, key compromise (rotate + revoke), data-breach notification steps (legal timeline with counsel).
- RTO 4h / RPO 5 min: PITR backups, cross-AZ Postgres, restore drill quarterly.
- Status page (status.ijimpay.com) from day one — trust is the product.

## 6. Pre-Launch Security Checklist

- [ ] External penetration test of API + dashboard + hosted checkout
- [ ] Mobile app security review (static analysis + API abuse attempt)
- [ ] Threat model workshop on money paths (charge, payout, adjustment)
- [ ] Load test: 50 rps sustained, provider-timeout storm simulation
- [ ] Chaos drill: kill worker fleet mid-payout-batch, verify no double-send (idempotency proof)
- [ ] Reconciliation dry run with real telco statement format
- [ ] Legal sign-off: PSP arrangement, merchant ToS, privacy policy (FR+EN)
