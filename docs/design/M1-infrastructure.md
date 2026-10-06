# M1 — Infrastructure & Zoho Books setup · Milestone Design Note

| | |
|---|---|
| **Status** | **Draft**: cannot be approved until M0 is signed off and the gaps in §8 are closed |
| **Version** | 0.7 |
| **Author** | Architect · **Date:** 2026-10-07 |
| **Implementer confirmation** | ☐ Understood · ☐ Questions (logged as GAP-nnn) |
| **Gate** | Project docs approved (`Rules.md` §9) ☐ · M0 signed off ☐ |

## 1. Scope
- **Milestone:** M1 (`milestones.md`)
- **Requirements:** Must #1 (Zoho UAE setup), #14 (bot DB), #15 (token refresh, retries, rate limits), #16 (self-hosted, HTTPS, backups, secrets) · NFR-3, NFR-6, NFR-7, NFR-9 (partial)
- **Exit criteria:** (E1) a test call from n8n creates and reads a contact in the DEV org; (E2) a backup restore has been tested once.

## 2. Workflows to build
| WF | Name | Spec | Notes |
|---|---|---|---|
| WF-91 | Zoho Auth | `Workflows.md` §6 | Single-flight refresh + failure branch (CR-002 rev 2); no saved executions (CR-008) |
| WF-92 | Zoho Client | `Workflows.md` §6 | Input `{tenant_id, method, path, query, body, accept}`; retry 2s/4s/8s on 429/5xx; one refresh-and-retry on 401; error mapping per §5.4; token protection (CR-008) |
| WF-90 | Error Handler (stub) | `Workflows.md` §6 | Minimal version (**redact** per CR-008 → log to `error_log` → alert the admin chat) so WF-91/92 have an error workflow. The full version comes in M2 |

All workflows start from the starter template (`Workflows.md` §4), with typed inputs (CR-006).

## 3. Data model
Apply `Database.md` §4–§7 as migrations, **unchanged** apart from any approved CRs:

| Order | File | Source |
|---|---|---|
| 1 | `<ts>_extensions_types.sql` | §4.1 |
| 2 | `<ts>_tenants.sql` | §4.2 |
| 3 | `<ts>_users_permissions.sql` | §4.3 |
| 4 | `<ts>_sessions_drafts.sql` | §4.4 |
| 5 | `<ts>_logs.sql` | §4.5 |
| 6 | `<ts>_reports_cache.sql` | §4.6 |
| 7 | `<ts>_rls.sql` | §4.7 (tables + default privileges for later objects, CR-023) |
| 8 | `<ts>_functions.sql` | §5.1–§5.5 (incl. CR-002 rev 2; CR-005 outcome pending) |
| 9 | `<ts>_function_privileges.sql` | §5.6 (CR-003, CR-009) |
| 10 | `<ts>_views.sql` | §6 (`security_invoker` + revoke, CR-023) |
| 11 | Seed | §7, with placeholders filled **outside Git** (tenant name, owner Telegram ID). Zoho secrets are inserted with `pgp_sym_encrypt` in a one-off session and never committed |

## 4. Intents and schemas
None are executed in M1. The intent seed loads with the migrations.

## 5. External APIs: verify each item and record the source URL here
| Item | Design assumption | Verified? |
|---|---|---|
| Zoho data centre for a UAE org and its `accounts_domain` / `api_domain` | Chosen at signup; recorded in `zoho_connections` | ☐ |
| Zoho Books API base | `{api_domain}/books/v3/…?organization_id=` | ☐ |
| OAuth flow | Server-based client, `refresh_token` grant, `access_type=offline` | ☐ |
| Scopes (least privilege) | `ZohoBooks.contacts.ALL, settings.READ, items.ALL, estimates.ALL, invoices.ALL, customerpayments.ALL, expenses.ALL, reports.READ` (adjust after checking) | ☐ |
| Rate limits | Per-minute limit per org plus a daily limit by plan → store the daily limit in `tenant_settings.zoho_daily_call_limit` | ☐ |
| Access-token minting limit per refresh token | **10 access tokens per 10 minutes per refresh token**; token lifetime 1 hour; invalid/revoked refresh token → `invalid_code`. Source: https://www.zoho.com/accounts/protocol/oauth/web-apps/access-token-expiry.html (Implementer 2026-10-07, re-checked by Architect 2026-10-07) | ☑ |
| Report endpoints and export formats on the chosen plan | Arch T2: **high risk for M8**. If no API exists, M8 builds PDF/XLSX from JSON | ☐ |
| Telegram `setWebhook` `secret_token` | Header `X-Telegram-Bot-Api-Secret-Token` | ☐ |

## 6. Configuration
### 6.1 Infrastructure (topology depends on GAP-011)
- VPS: Docker, Caddy (or Nginx) reverse proxy, HTTPS, firewall allowing only 22/80/443, SSH keys only.
- n8n on its own Postgres. Editor protected by owner login + 2FA, and an IP allow-list or VPN for `/` and `/rest` (keep `/webhook/*` public).
- Execution data: success = none, error = all, prune after 7 days. **WF-91 and WF-92 save nothing** (CR-008). GAP-001 is still open for PIN-handling workflows (M2).
- Supabase project per environment (DEV, PROD); hosted vs self-hosted decided under open question #5.
- Backups: daily `pg_dump` of the n8n DB plus Supabase, copied off-server, kept 14 days. **E2 restore drill** goes into `docs/test-evidence/M1/restore-drill.md`.

### 6.2 n8n credentials (names exactly as in `Workflows.md` §2)
`Supabase Service Role` (Supabase API credential with the `service_role` key; **no** Postgres-node credential with the owner login, CR-021) · `Zoho Books OAuth (DEV)` *(if WF-91 uses the n8n OAuth2 credential instead of the DB tokens, raise a GAP: the docs specify DB-stored tokens)* · `Telegram Bot (DEV)` · `Admin Alert Bot`.

### 6.3 Environment variables
As named in `Workflows.md` §2. `DB_ENCRYPTION_KEY` is generated on the server and never leaves it.

### 6.4 Zoho Books DEV org setup checklist
- [ ] UAE edition, base currency AED, timezone Asia/Dubai, fiscal year
- [ ] VAT registration: TRN, VAT 5% tax, zero-rated and exempt taxes; record their tax IDs
- [ ] Tax treatments and place of supply (emirates) enabled on contacts
- [ ] Multi-currency on; add the currencies the client uses
- [ ] Auto-numbering for estimates, invoices, payments
- [ ] Payment terms (e.g. Net 30)
- [ ] Bilingual EN/AR templates for estimate, invoice, payment receipt (accountant review in M5)
- [ ] Dedicated API user with a least-privilege custom role (if the plan allows)
- [ ] Record org ID, DC domains, and plan limits in `zoho_connections` / `tenant_settings`

### 6.5 Telegram
- DEV bot from BotFather; webhook `https://<host>/webhook/zb/telegram/dev` with `secret_token`; privacy mode on; bot commands set.

## 7. Acceptance tests
| # | Test | Exit criterion | Evidence file |
|---|---|---|---|
| T1 | Migrations apply cleanly on an empty DEV Supabase; `v_tenants_without_owner` returns 0 rows after seeding; record the Postgres version (CR-007) and confirm the `extensions` schema layout and search_path (CR-020) | Must #14 | `M1/migrations.md` |
| T2 | `anon` key cannot read any table **or view** (`v_daily_health`, `v_tenants_without_owner`) or call any `/rpc` function; a table created after 0007 (test migration) is also closed to `anon` (CR-023); no sequence in `public` is granted to `anon`/`authenticated` (`has_sequence_privilege`, CR-024); `service_role` still reads both views; n8n's identity (PostgREST, `service_role` key, CR-021) can call every function **except** `recover_user_pin()` and `move_user_channel()` (permission denied, CR-019/022); no n8n credential uses the owner login | NFR-3 | `M1/rls.md` |
| T3 | WF-91 returns a cached token while it is valid; refreshes when < 5 min remain; 10 parallel calls cause **one** refresh | Must #15 | `M1/wf91.md` |
| T3b | Forced invalid refresh token → connection status `error`, exactly one admin alert, callers fail in under 1 s | Must #15 | `M1/wf91.md` |
| T3c | Forced token-call timeout → 5 s cool-down, then exactly one successful refresh | Must #15 | `M1/wf91.md` |
| T3d | Alert suppression: (a) 3 transient failures within 10 min → exactly 1 alert; (b) after a successful refresh, a new failure alerts at once; (c) still failing 10+ min later → exactly 1 reminder | NFR-9 | `M1/wf91.md` |
| T4 | WF-92 creates and then reads a contact in the DEV org | **E1** | `M1/wf92-contact.md` |
| T5 | WF-92 maps a forced 429 / 5xx / 401 / Zoho validation error to the §5.4 codes; evidence = sanitised `error_log` rows with retry count (no saved executions, CR-008) | Must #15 | `M1/wf92-errors.md` |
| T6 | `zoho_api_usage` increments per call; an alert fires at 80% (test with a low limit) | NFR-9 | `M1/usage-alert.md` |
| T7 | No secret in exported JSON, Git, `error_log`, saved executions, or Postgres logs. After forced 401/429/5xx/validation runs, search for `Zoho-oauthtoken` and the Zoho token pattern → 0 hits. WF-90 still fires for an unhandled WF-92 error with saving off | NFR-3 | `M1/secrets-scan.md` |
| T8 | Backup → restore to a scratch DB → row counts match | **E2**, NFR-6 | `M1/restore-drill.md` |
| T9 | Editor not reachable from a non-allow-listed IP; webhook path still reachable | NFR-3 | `M1/editor-access.md` |

## 8. Risks and open questions
| Item | Link |
|---|---|
| Environment topology and the shared VPS | GAP-011 → **CR-004 awaiting Client** (blocks approval) |
| Token refresh concurrency | GAP-005 → CR-002 rev 3 approved (C3 closed) |
| Token in saved executions | GAP-012 → CR-008 approved |
| Encryption key in SQL args / Vault | GAP-012 → **CR-005 on hold**: Supabase hosting (open question #5) (blocks approval) |
| Function EXECUTE grants | GAP-010 → CR-003 approved; ordering fixed by CR-009 (GAP-016) |
| Trigger input mode | GAP-013 → CR-006 approved |
| Postgres version | GAP-015 → CR-007 approved |
| Zoho plan and DC still unknown | Open question #5 |
| Report API availability | Arch T2: verify in M1 (§5) |

## 9. Out of scope
Telegram inbound/auth (M2), any AI call (M3), any domain workflow (M4+), PROD Zoho org configuration beyond creation (done at M9 go-live, using the same checklist as §6.4).
