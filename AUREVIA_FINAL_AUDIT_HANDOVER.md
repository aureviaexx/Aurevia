# Aurevia Exchange — Final Audit and Release Recovery Handover

**Recovery date:** 2026-11-13

The recovery and validation sections below record the state known on 2026-11-13. The final production verification section at the end updates the health, migration, security, and release-readiness conclusions as of 2026-11-14.

**Repository:** `/workspaces/Aurevia`

**Application:** `Aurevia-Exchange-main/aurevia`

**Branch:** `main`

**Starting HEAD:** `4dcbd6b52f01b7edfa6b7db1029dbcd9f505aa7b`

## Recovery and preservation

- The working tree was clean at recovery start (`main...origin/main`) with no staged, modified, deleted, or untracked files.
- The recorded starting HEAD matched the expected HEAD. The existing E2E harness fixes are already committed in `b3b06fb`; no earlier recovery changes were present to overwrite.
- No `AUREVIA_FINAL_AUDIT_HANDOVER.md`, earlier Chromium logs, PID/exit record, or Playwright report was found in the repository or searched runtime locations. The recent recovery sessions contain only the initial task prompts, so prior checklist completion cannot be verified.
- Before starting a new run, no E2E, Chromium, Next.js, or Prisma process or project lock was active. The only existing Docker container was the development container. Disk availability was 26 GiB, available memory 13 GiB, and the machine had 4 CPUs.
- The standalone production build generated a change to `next-env.d.ts`; that generated change was restored to its checked-in contents. Build and Playwright output directories are ignored and were not staged.

## E2E harness and Chromium results

The existing isolated runner clears inherited database URLs, starts a disposable PostgreSQL 16 container on a dynamically assigned loopback-only port, applies migrations to that container, disables external integrations, and removes the container after the run. Playwright additionally rejects non-local database hosts and database names without the `_e2e` suffix.

`npm run test:e2e:chromium` completed successfully:

- **363 passed, 0 failed** in 18.8 minutes; process exit code 0.
- Observed run processes: isolated runner PID 2295, Playwright PID 2577, Chromium PID 3405. They have exited.
- Full console log: `/tmp/aurevia-chromium-recovery-20261113.log` (temporary runtime file).
- HTML report: `Aurevia-Exchange-main/aurevia/playwright-report/index.html` (ignored generated output).
- Playwright last-run metadata: `Aurevia-Exchange-main/aurevia/test-results/.last-run.json` (ignored generated output).
- No isolated E2E container or runner lock remained after completion.

The run included the financial-flow checks for REAL withdrawal reservations and settlement, uncredited pending REAL deposits, internal deposit approval and fee allocation, idempotent DEMO orders, and account-mode ledger isolation. Only expected “not-configured” email-delivery warnings appeared; the tests passed.

## Validation and build

| Check | Result |
|---|---|
| `npm run typecheck` | Passed |
| `npm run lint` | Passed |
| `npm test` | 138 files, 735 tests passed |
| `npx prisma validate` | Passed using non-production placeholder database URLs |
| `npm run build` | Passed; Prisma Client generated, Next.js compiled, and 154 static pages generated |

The standalone build explicitly overrode database URLs with a local placeholder and disabled external integrations. It did not apply migrations. The E2E production-mode web server also completed its production build against the disposable test database.

## Financial-integrity review

The reviewed paths include `lib/ledger.ts`, `lib/orders.ts`, `lib/real-withdrawals.ts`, `lib/alpaca-paper-orders.ts`, and the wallet/funding and order routes.

- Customer balances are derived from ledger entries; posting uses decimal amounts, rejects non-positive transfers and account currency/mode mismatches, and creates balanced debit/credit entries.
- Order and funding mutations use database transactions; key financial mutations use serializable isolation and idempotency protections.
- REAL withdrawal requests reserve funds while pending/approved/processing. User cancellation releases the reservation without a debit. At the recovery snapshot, automated REAL settlement was unconfigured; approval alone did not claim that money was paid.
- Deposits remain pending until administrator approval; the E2E suite verified that pending deposits are not credited and that approved gross amount, fee, and net credit are reflected in the ledger.
- The scoped code review found no verified financial-integrity defect. This is not an independent accounting audit, and no production ledger data or reconciliation was inspected.

## Prisma migrations and production divergence

- The repository contains **59 PostgreSQL migrations**, latest `20261111010000_didit_kyc_sessions`.
- The isolated E2E setup successfully applied the repository migrations to a fresh disposable database.
- Production `_prisma_migrations` status could not be established: no production database query or migration was run. The read-only request to `https://aureviaexchange.onrender.com/api/health` timed out after 15 seconds without an HTTP response. Production migration divergence and current service health therefore remain **unverified**.

## Deployment readiness

No push or deployment was performed. The repository’s Render workflow deploys `main` after the Test workflow succeeds for the current main SHA. Given the unverified production migration status and the health-check timeout, **do not push to `main` or deploy yet**. First have the production operator confirm service health and migration status through the authorized production controls; do not apply migrations as part of this recovery.

## Change inventory

The only intended project change from this recovery is this handover document. No application source or migration was changed. Generated build and Playwright artifacts remain untracked-by-Git/ignored and are not part of the commit.

## Final production verification and release decision

**Audit date:** 2026-11-14

**Application source reviewed:** unchanged since `4dcbd6b52f01b7edfa6b7db1029dbcd9f505aa7b`

### Git and repository identity

- At audit start, the Git root was `/workspaces/Aurevia`, branch `main`, and HEAD was `b090b31f27b6806ebc034dc0b2f0a81a4a2d8848`. Its parent is the expected `4dcbd6b52f01b7edfa6b7db1029dbcd9f505aa7b`.
- The prior commit exists and adds only this handover document. At audit start the working tree was clean. `main` is one commit ahead of configured `origin/main`, with no uncommitted or untracked project files.
- Configured `origin` is `https://github.com/aurevia2/Aurevia`, not the requested `aurevia37/Aurevia-Exchange`. The requested repository resolves to `aurevia16/Aurevia-Exchange`; its `main` is `eb8aafec82f1a7169f465d72963d68e147b6f8a5`, and the audited commit is not present there. Do not change remotes or push until the repository owner and intended history are confirmed.
- A successful GitHub Test workflow on the configured repository tested the parent source commit `4dcbd6b`; the audited HEAD differs only by documentation. Its isolated Chromium run reported **363 passed, 0 failed**. The production build compiled successfully and generated all 154 static pages. This is CI evidence for the unchanged application source, not a run on the user-specified repository.

### Live production health

Checks against `https://aureviaexchange.onrender.com` on 2026-11-14:

| Request | Result |
|---|---|
| HTTPS homepage | HTTP/2 200; page title “Aurevia Exchange” |
| `/api/health` | HTTP/2 200; `ok: true`, `database: ok` |
| `/api/auth/session` | HTTP/2 200; unauthenticated session `{}` |
| `/api/profile`, `/api/orders`, `/api/wallet` | HTTP/2 401, expected without authentication |

The first homepage/health requests had roughly 4.5-second time-to-first-byte; subsequent requests completed in roughly 0.15–0.69 seconds. This is consistent with a slow wake-up/cold start, but the prior timeout's cause cannot be conclusively separated from network delay without Render logs. No persistent application or database outage was observed. No Render dashboard/API credentials or service logs were available, so the deployed SHA, rollout status, and startup-error logs could not be independently confirmed. The successful Render deploy workflow run only confirms that its deploy hook request succeeded, not that a particular deployment completed.

### Read-only migration and schema review

The migration-history and catalog queries used a read-only transaction against the configured Supabase database. The connection was not independently matched to Render's private runtime configuration; treat it as the configured production candidate until an operator verifies that identity. `DIRECT_URL` in the local environment was syntactically invalid and was not used. No production migration or SQL write was run.

- The repository contains 59 migration directories. The configured database history contains 62 rows: 60 successful, zero unfinished, and two rolled-back attempts.
- All 59 repository migrations have successful history records; none are pending in that database. There is one additional successful database-only migration, `20261013110000_payment_method_network`, whose migration SQL is absent from the checked-out source/history.
- Catalog inspection confirmed a `PaymentMethod.network` text column in the database, absent from the current Prisma model. Its exact origin, intended data, and migration SQL remain unknown; do not reconcile or drop it without the database owner.
- The applied history is not fully canonical: `20261104130000_user_gender_selection` was recorded after the later-timestamped `20261104100000_account_data_deletion`, and its history row has `applied_steps_count=0`. The `User.gender` column and all `GenderSelection` enum labels do exist in the inspected catalog, but the mechanism by which they were created is unverified.
- The two rolled-back attempts are `20261006211000_private_storage_bucket_policies` and `20261013103000_client_story_private_storage`; later successful source-migration records exist. A partial unique index for REAL Bitcoin funding transaction references is present in the inspected database.
- Review of checked-in migration SQL found no `DROP TABLE`, `DROP COLUMN`, `TRUNCATE`, or `DELETE FROM` statements. The already-recorded `20261105090000_security_center` migration does set existing `User.twoFactorEnabled` values to false; this is a historical user-security-setting change, not a pending migration. The account-data migration changes foreign keys, including ledger-entry cascade behavior on ledger-account deletion. The application account-closure path reviewed below anonymizes accounts and retains ledger/audit records rather than deleting ledger accounts.
- No customer, account, wallet, or financial-history rows were queried. The migration divergence and inability to prove the queried database is exactly Render's attached instance remain operator blockers.

### Financial-integrity review

Reviewed ledger, funding, withdrawal, demo-reset, paper-order, and account-closure code and regression tests. No verified financial-integrity defect was found in the reviewed paths; this was a code/test review, not a production-ledger reconciliation or independent accounting audit.

- Ledger balances are derived from decimal debit/credit entries. Posting rejects non-positive values, currency/mode mismatches, and same-account transfers; financial mutations use database transactions, idempotency, and serializable isolation where required.
- REAL account initialization does not credit a starting balance; DEMO initialization records exactly **$5,000** as a balanced opening allocation. DEMO reset is ledger-based and audited; REAL funding and withdrawal paths are mode-gated.
- Deposits remain pending until authorized administrator review/crediting. Receipt uploads are private, and the admin UI labels them as user-submitted and not independently verified. No automatic Bitcoin-chain confirmation was found; approval therefore relies on an administrator's independent evidence review, and the approval route does not itself require a receipt/reference. The inspected database has a partial unique REAL Bitcoin transaction-reference index. This is a manual operational control, not automated proof verification.
- Withdrawals reserve funds while pending/approved/processing. Approval and processing do not debit the ledger. Rejection/failure release reservations without a debit; completion requires explicit administrator confirmation of external processing, checks the remaining reserve, and posts one idempotent debit. This is a manual confirmation flow; no automated external payout provider is configured. Lifecycle tests cover these transitions and repeat completion.
- REAL orders use the disclosed Alpaca paper/simulated execution model. Aurevia records confirmed provider fills into its own portfolio/ledger; the reviewed paths do not claim live-market settlement.
- Account closure verifies password/admin authorization, blocks nonzero balances and unresolved financial obligations, anonymizes personal fields, revokes sessions, and retains financial/regulatory/audit records, including KYC/tax documents. Avatar cleanup is attempted after the database transaction; the admin path reports cleanup pending on failure. No customer data or private files were opened. Retention of KYC/tax documents should remain aligned with the applicable retention policy.

### Security finding

One high-confidence, actionable issue was found:

| # | Severity | File | Lines | Vulnerability | Confidence |
|---|----------|------|-------|---------------|------------|
| 1 | 🟠 HIGH | `app/api/webauthn/register/start/route.ts`; `app/api/webauthn/register/finish/route.ts`; `app/api/password-reset/confirm/route.ts` | 11; 11, 48–57; 30–32 | A session-only account can enroll a new passkey without fresh reauthentication. A session thief can register an attacker-controlled passkey; password reset revokes active sessions but does not revoke existing passkeys, allowing persistence after reset. | 9/10 |

The remaining reviewed controls did not yield another verified finding in the scoped review. This does not substitute for a full independent penetration test. The passkey persistence issue is a release blocker; remediate with fresh authentication for enrollment and a recovery/reset policy that invalidates attacker-added credentials, then add focused regression tests.

### Quality checks and available evidence

- Re-ran locally: TypeScript passed; ESLint passed; Vitest passed (**735 tests across 138 files**); `npx prisma validate` passed with non-production placeholder URLs.
- The existing Playwright `test-results/.last-run.json` reports `status: passed` and no failed tests. The successful GitHub CI run on the unchanged source parent independently reports 363 Chromium tests passed and 0 failed using the isolated E2E workflow.
- The successful CI production build compiled and generated 154 static pages. No second E2E run or production build was started during this audit; no application source changed, and rerunning would overwrite ignored generated test/build artifacts.
- These checks do not resolve production migration divergence, prove the Render deployment SHA, or remedy the passkey finding.

### Push/deployment decisions and operator actions

**SAFE TO PUSH: NO.** The configured remote is not the GitHub repository specified for this audit, the audited commit is not present on that target's `main`, and pushing to the configured `main` runs a successful-Test-triggered Render deploy workflow. Do not push until the target repository/history is reconciled and the high-severity passkey issue is fixed and verified.

**SAFE TO DEPLOY: NO.** The passkey persistence issue is unresolved; production migration history contains an undocumented source-divergent migration and noncanonical history; the configured database could not be proven to be Render's attached database; and the active Render deployment SHA/logs were unavailable. Live homepage and health checks are currently successful, but that is not sufficient for deployment approval.

Required before reconsidering release:

1. Fix and regression-test passkey enrollment reauthentication and passkey recovery/reset behavior.
2. Have the database owner identify `20261013110000_payment_method_network`, verify the queried database is the Render runtime database, and reconcile migration ownership/history without destructive SQL or automatic migration deployment.
3. Have the Render operator confirm active deployment SHA, rollout status, and runtime logs; investigate the earlier timeout if it recurs.
4. Confirm the intended GitHub repository and branch lineage; run required CI on the exact intended commit.
5. Reassess financial, security, migration, and deployment readiness after those actions. No push, deployment, production configuration change, migration, or financial-record modification was performed by this audit.
