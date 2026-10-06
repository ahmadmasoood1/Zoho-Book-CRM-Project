# Change Requests — Zoho Books Chat Assistant

| | |
|---|---|
| **Owner** | Architect |
| **Process** | `Rules.md` §5: a CR is created only after a GAP has an Implementer justification, or after a new client requirement. Docs are updated (and versions bumped) **before** the build. |

## Register

| ID | Title | Linked gap | MoSCoW impact | Approved by | Status |
|---|---|---|---|---|---|
| CR-001 | Gap Protocol instruction added to Rules.md §4.1 | New requirement (Project Owner) | None | Project Owner | Done |
| CR-002 | Single-flight Zoho token refresh (rev 3: failure paths + alert suppression) | GAP-005 | None | Architect | Approved · docs updated · awaiting build (M1) |
| CR-003 | Revoke function EXECUTE from public/anon/authenticated | GAP-010 | None | Architect | Approved · docs updated · awaiting build (M1) |
| CR-004 | Two n8n instances (DEV = existing Contabo, PROD = new) + separate Supabase projects | GAP-011 | None (adds infra cost) | **Client** (pending) | Draft · awaiting Client approval |
| CR-005 | Zoho secrets in Supabase Vault instead of pgcrypto + app key | GAP-012 | None | Architect (pending) | Draft · design agreed · on hold: Supabase hosting decision |
| CR-006 | Typed inputs on the Execute Workflow Trigger | GAP-013 | None | Architect | Approved · docs updated · awaiting build (M1) |
| CR-007 | State the tested Postgres version | GAP-015 | None | Architect | Approved · docs updated |
| CR-008 | Keep the Zoho access token out of saved executions and logs | GAP-012 (C2a) | None | Architect | Approved · docs updated · awaiting build (M1) |
| CR-009 | Function privileges in their own migration after the functions | GAP-016 | None | Architect | Approved · docs updated · awaiting build (M1) |
| CR-010 | PIN isolation: WF-08 PIN Handler, redaction, PROD no-save (rev 2: late-PIN rule) | GAP-001 | None | Architect | Approved · docs updated · build M2 |
| CR-011 | `verify_pin_for_action()`; `verify_user_pin()` fix | GAP-002 | None | Architect | Approved · docs updated · build M2/M3 |
| CR-012 | First-time PIN setup driven by `pin_hash` | GAP-003 | None | Architect | Approved · docs updated · build M2 |
| CR-013 | Inbound update de-duplication | GAP-004 | None | Architect | Approved · docs updated · build M2 |
| CR-014 | Proactive draft-expiry notice (WF-94) | GAP-007 | None | Architect | Approved · docs updated · build M3 |
| CR-015 | Redact audited drafts instead of keeping them | GAP-008 | None (NFR-5 wording) | Architect | Approved · docs updated · build M2 |
| CR-016 | Null-tenant error purge; tenant from `TENANT_ID`; M15 mapping note | GAP-009 | None | Architect | Approved · docs updated · build M2 |
| CR-017 | Future intents seeded inactive; `report.schedule` needs PIN | GAP-014 | None | Architect | Approved · docs updated · build M1/M2 |
| CR-018 | Optimistic locking on sessions (TOUCH vs STATE writes) | GAP-006 | None | Architect | Approved · docs updated · build M2/M3 |
| CR-019 | Sole-Owner PIN recovery; remove `set_user_pin()` | GAP-017 | None | Architect | Approved · docs updated · build M2 |
| CR-020 | SQL hardening: PIN required in `check_permission()`, fail-loud settings, extensions schema | GAP-018 | None | Architect | Approved · docs updated · build M1 |
| CR-021 | n8n DB identity via PostgREST only; WF-08 input checks; search_path; minor SQL fixes | GAP-020 | None | Architect | Approved · docs updated · build M1/M2 |
| CR-022 | Operator account move for a lost Telegram account; user.remove cancels drafts | GAP-019 | None | Architect | Approved · docs updated · build M2 |
| CR-023 | New objects start closed (views, later tables); search_path on Zoho token functions; Cancel per PIN state; M3 trigram RPC noted | GAP-021 | None | Architect | Approved · docs updated · build M1 |
| CR-024 | Revoke existing sequences from anon/authenticated | GAP-022 | None | Architect | Approved · docs updated · build M1 |

<!-- Copy the CR template from Rules.md §5 below this line for each new CR. -->

### CR-001: Gap Protocol instruction added to Rules.md §4.1
- **Linked gap:** new requirement (Project Owner instruction, 2026-10-06)
- **Reason:** The Project Owner wants the Gap Protocol rule kept in `Rules.md`, not in `Memory.md`.
- **Documents updated:** `docs/Rules.md` §4.1, header, §9, §10 → v0.2; `Memory.md` M-001 replaced with a pointer
- **Build impact:** none
- **MoSCoW impact:** none
- **Approved by:** Project Owner · **Date:** 2026-10-06
- **Implemented by:** Architect (documentation only) · **Verified by:** Project Owner · **Date:** 2026-10-06

### CR-002: Single-flight Zoho token refresh
- **Linked gap:** GAP-005
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: concurrent refreshes can exceed Zoho's token-minting limit; advisory locks don't work because each n8n Postgres call is its own transaction.
- **Documents updated:** `Database.md` §4.2 (`refresh_lock_until`), §5.3 (`zoho_claim_refresh()`, claim released in `zoho_save_tokens()`) → v0.2; `Workflows.md` §6 WF-91 → v0.2
- **Build impact:** migrations 0002/0008; WF-91. Test T3 unchanged.
- **Condition:** the Zoho refresh-token minting limit is verified and cited in M1 note §5 before build. ✅ Met 2026-10-07: 10 tokens / 10 min (Zoho Accounts OAuth docs), recorded in M1 note §5.
- **Rev 2 (2026-10-07, clarification C1):** losers never refresh or alert, and return `ZOHO_AUTH` / `refresh_wait_timeout` after 5 tries; winner's token call times out after 10 s with no node retry; new `zoho_refresh_failed()` (permanent = `invalid_code` → status `error`; transient → 5 s cool-down); `zoho_claim_refresh()` claims only active connections; tests T3b/T3c. Docs: `Database.md` v0.3 §5.3; `Workflows.md` v0.3 WF-91. Alert-suppression storage is pending clarification C3.
- **Rev 3 (2026-10-07, clarification C3):** new column `zoho_connections.last_alert_at`; `zoho_refresh_failed()` returns `{alert, status}` and decides atomically (permanent → always; transient → first failure, then at most one reminder every 10 min; a successful refresh ends the incident). WF-91 alerts only when `alert = true`; null → `INTERNAL` + alert. Test T3d. Option (b), an `error_log` lookup, was rejected: no code column, nullable tenant, not atomic, purged by housekeeping. Docs: `Database.md` v0.4; `Workflows.md` v0.4; M1 note v0.3.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-003: Revoke function EXECUTE from public/anon/authenticated
- **Linked gap:** GAP-010
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: oversight; Postgres grants function EXECUTE to PUBLIC by default.
- **Documents updated:** `Database.md` §4.7 → v0.2
- **Build impact:** migration 0007 (run after 0008, as the function owner). Test T2 extended: `anon` cannot call any `/rpc` function.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-004: Two n8n instances and separate Supabase projects (DRAFT, needs Client approval)
- **Linked gap:** GAP-011
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: `ENV`, `TENANT_ID`, credentials, execution data, and the error workflow are instance-wide in n8n; the existing instance runs an unrelated active workflow.
- **Proposed change:** existing Contabo instance = **DEV**; new dedicated instance (own container + domain, preferably own VPS) = **PROD**; separate Supabase projects for DEV and PROD.
- **Documents to update after approval:** `Architecture.md` §12 (Environments), §15 (T3/T4); `Workflows.md` §2; M1 note §6.1
- **Build impact:** infrastructure only
- **Cost impact:** one more VPS/instance (+ a second Supabase project if hosted) → **Client approval required** (`Rules.md` §5.3)
- **Approved by:** — · **Date:** —

### CR-005: Zoho secrets in Supabase Vault (DRAFT, on hold)
- **Linked gap:** GAP-012
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: the app-held key passed in SQL can reach Postgres logs and saved executions.
- **Proposed change:** `client_secret`, `refresh_token`, and the cached access token stored in Supabase Vault, read through one `security definer` function (EXECUTE granted to `service_role` only); WF-91 saves no execution data. Fallback if Vault is unavailable: pgcrypto with statement logging off for the n8n role and the key passed only as a bound parameter.
- **Agreed design (clarification C2b, 2026-10-07):** `zoho_connections` keeps the non-secret columns (`organization_id`, `api_domain`, `accounts_domain`, `client_id`, `access_token_expires_at`, `refresh_lock_until`, `status`, `last_refreshed_at`, `last_error`). The three `*_enc` bytea columns become uuid references to Vault secrets: `client_secret_id`, `refresh_token_id`, `access_token_id`. `zoho_get_credentials(p_tenant)` reads `vault.decrypted_secrets` and takes no key argument. `zoho_save_tokens(p_tenant, p_access, p_expires_at)` updates (or creates) the access-token secret and the expiry, releasing the claim, in one function. All three functions are `security definer` with `set search_path = ''`, executable only by n8n's role. The seed uses no `pgp_sym_encrypt`, and `DB_ENCRYPTION_KEY` is removed. Residual risk: the 1-hour access token is a bound parameter of `zoho_save_tokens()`; statement logging is turned off for n8n's role where the hosting allows. Source: https://supabase.com/docs/guides/database/vault (hosted platform).
- **Token exposure in executions:** split out as **CR-008** (approved), so M1 isn't blocked by hosting.
- **On hold until:** Supabase hosted vs self-hosted is decided (open question #5). If self-hosted, Vault availability must be verified first; if Vault is unavailable, the fallback applies.
- **Documents to update after approval:** `Database.md` §4.2, §5.3; `Workflows.md` §2 (`DB_ENCRYPTION_KEY`), WF-91
- **Approved by:** — · **Date:** —

### CR-006: Typed inputs on the Execute Workflow Trigger
- **Linked gap:** GAP-013
- **Reason:** Implementer GAP-RESPONSE 2026-10-07. Architect verified (n8n-mcp node catalogue, 2026-10-07): `inputSource` options `workflowInputs` / `jsonExample` / `passthrough` exist from typeVersion 1.1; the current typeVersion is 1.2.
- **Documents updated:** `Workflows.md` §2 (Sub-workflows), §4 (template: `jsonExample`, typeVersion 1.2) → v0.2
- **Condition:** the exact n8n version is recorded at M1 start; if the instance doesn't offer typeVersion 1.2, use 1.1 with the same parameters and note it in the change history.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-007: State the tested Postgres version
- **Linked gap:** GAP-015
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: the version isn't known until the Supabase project exists.
- **Documents updated:** `Database.md` header → v0.2; `PROJECT_OVERVIEW.md` §7
- **Build impact:** the Implementer records the provisioned version in M1 evidence and tests the migrations against that exact major version before T1.
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-008: Keep the Zoho access token out of saved executions and logs
- **Linked gap:** GAP-012 (clarification C2a, split out of CR-005)
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: the token must be in WF-92 item data to build the Authorization header, so it can't be removed, only stopped from being saved. This applies to both the Vault and the pgcrypto designs.
- **Documents updated:** `Workflows.md` v0.3: §2 (settings exception for WF-91/92), WF-92 (token protection, items 1–4), WF-90 (redaction); M1 note v0.2 (T5, T7)
- **Build impact:** WF-90, WF-91, WF-92 settings and error branches. No schema change.
- **Trade-off accepted:** no saved n8n executions to inspect for WF-91/92; the sanitised `error_log` row is the debugging evidence.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-009: Function privileges in their own migration after the functions
- **Linked gap:** GAP-016 (raised by the Implementer with reason and recommendation, `Rules.md` §4.4)
- **Reason:** CR-003 put the privilege block in 0007, which runs before the functions in 0008. As filed, n8n would get "permission denied" on every 0008 function.
- **Documents updated:** `Database.md` v0.3: §4.7 (block removed), new §5.6 `0008b_function_privileges.sql` (adds the default grant to `service_role`; any other n8n DB role is granted too and named in `Workflows.md` §2); M1 note v0.2 §3 (migration order 1–11)
- **Build impact:** migration order only. No workflow impact.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-010: PIN isolation (WF-08 PIN Handler, redaction, PROD no-save)
- **Linked gap:** GAP-001
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: the PIN sits in WF-00/01/02 item data and is logged by WF-02 before WF-06 knows it's a PIN.
- **Documents updated:** `Workflows.md` v0.5: §2 (PROD no-save exception for WF-00/01/02/08), register (WF-08), WF-00 step 6, WF-02 redaction, WF-06, new WF-08 spec, WF-90 (never item data); `Architecture.md` v0.2 §4; `milestones.md` v0.2 M2
- **Build impact:** WF-00, WF-02, WF-06, WF-08 (new), WF-90. No schema change.
- **Tests:** PIN `[REDACTED]` in `message_log` and deleted in chat; PROD settings + forced failure → test PIN 0 hits in the n8n execution DB.
- **Rev 2 (2026-10-07, clarification C4):** point 4 replaced by a narrow late-PIN rule: digits only AND `awaiting` null AND `last_pin_prompt_at` within `draft_ttl_minutes` → WF-08 `late_pin` mode. `awaiting` values defined in `Workflows.md` §5.2. Residual risk accepted: a PIN typed with no recent prompt is handled as normal text. Docs: `Workflows.md` v0.6 §5.2, WF-00, WF-02, WF-06, WF-08. Tests: field answer "1500" not deleted; late PIN 10 min after expiry caught; "1234" with no recent prompt → normal flow; after a successful PIN, a number → normal flow.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-011: `verify_pin_for_action()` and `verify_user_pin()` fix
- **Linked gap:** GAP-002
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: linking a correct PIN to the draft was never specified; counting a missing PIN as a failure was a bug.
- **Documents updated:** `Database.md` v0.5 §5.1 (`verify_user_pin()` with `pin_not_set` and `just_locked`; new `verify_pin_for_action()` that reuses it in one transaction and sets `pin_verified_at` + `status='confirmed'`); `Workflows.md` v0.5 WF-06, WF-08
- **Build impact:** migration 0008; WF-06, WF-08.
- **Tests:** correct PIN → claim succeeds once; 5 wrong → lockout + exactly 1 Owner alert; `pin_hash` null → `pin_not_set`, counter unchanged; someone else's draft → `action_invalid`.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-012: First-time PIN setup driven by `pin_hash`
- **Linked gap:** GAP-003
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: onboarding was never specified; status `invited` would conflict with the WF-02 lookup and M1 T1.
- **Documents updated:** `Database.md` v0.5 §4.3 (`pin_pending_hash`), §5.1 (`pin_setup_first()`, `pin_setup_confirm()`), seed comment; `Workflows.md` v0.5 WF-02 gate + unknown-sender reply with own ID, WF-08 setup states, WF-40 user.add / reset_pin, error code `PIN_NOT_SET`
- **Build impact:** migrations 0003/0008; WF-02, WF-08, WF-40; message keys in 4 languages.
- **Tests:** 2-step setup; mismatch restarts; no PIN → can't create anything; after reset a new PIN is required; seed Owner → T1 still 0 rows.
- **Architect decisions:** recent-unknown-sender buttons are not in Phase 1. The fate of `set_user_pin()` and sole-Owner PIN recovery are raised as **GAP-017**.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-013: Inbound update de-duplication
- **Linked gap:** GAP-004
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: Telegram repeats unsuccessful deliveries; `update_id` identifies repeats (Telegram Bot API `setWebhook`).
- **Documents updated:** `Database.md` v0.5 §2, §4.5 (`inbound_updates`, in 0005 so RLS in 0007 and privileges in 0008b apply in order, L-005), §4.7 RLS list, §5.4 `register_inbound()`, §5.5 2-day purge; `Workflows.md` v0.5 WF-00 step 3; `milestones.md` M15 note
- **Build impact:** migrations 0005/0007/0008; WF-00; WF-93 via housekeeping.
- **Tests:** the same update posted twice → 1 `message_log` row, 1 reply, 1 draft.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-014: Proactive draft-expiry notice (WF-94 Draft Expiry)
- **Linked gap:** GAP-007
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: FR-3.3 "the user is told" needs a proactive message; WF-93 runs only daily.
- **Documents updated:** `Database.md` v0.5 §5.2 `expire_drafts()`; `Workflows.md` v0.5 register + new WF-94 spec; `Architecture.md` v0.2 §4; `milestones.md` v0.2 M3
- **Build impact:** migration 0008; WF-94 (new); message key `draft.expired_notice` in 4 languages.
- **Tests:** buttons removed and notice within 5 min of expiry; old button → `draft.expired`; 2 drafts → 1 message.
- **Architect decision:** a notice within 30–35 minutes of the preview meets FR-3.3.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-015: Redact audited drafts instead of keeping them
- **Linked gap:** GAP-008
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: keeping audited drafts kept customer and amount payloads for 5 years, against NFR-5.
- **Documents updated:** `Database.md` v0.5 §2, §4.4 (`redacted_at`), §5.5 (redaction step); `PRD.md` v0.2 NFR-5
- **Build impact:** migrations 0004/0008.
- **Tests:** an executed draft older than 7 days keeps meta + Zoho IDs only, audit link intact; a cancelled unaudited draft is deleted; a second run changes nothing.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-016: Null-tenant error purge; tenant from `TENANT_ID`; M15 mapping note
- **Linked gap:** GAP-009
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: null-tenant rows never matched the purge; the bot→tenant mapping was postponed without being written down.
- **Documents updated:** `Database.md` v0.5 §5.5; `Workflows.md` v0.5 WF-00 step 3, WF-90; `milestones.md` v0.2 M15
- **Build impact:** migration 0008; WF-00, WF-90.
- **Tests:** a null-tenant `error_log` row older than 180 days is purged; an unknown-sender `message_log` row has the tenant set.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-017: Future intents seeded inactive; `report.schedule` needs PIN
- **Linked gap:** GAP-014
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: `check_permission()` ignored `intents.active`, so seeded future intents were allowed; `report.schedule` without PIN was a mistake.
- **Documents updated:** `Database.md` v0.5 §5.4 `check_permission()`, §7.1 (all intents inactive; each milestone migration activates its own), §7.2 needs_pin list; `Architecture.md` v0.2 §8
- **Build impact:** migrations 0008/seed; one activation line per milestone migration; WF-04 uses active intents only.
- **Tests:** inactive intent → `PERMISSION_DENIED` for the Owner too; `report.schedule` needs PIN; M2 intents active after the M2 migration.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-018: Optimistic locking on sessions (TOUCH vs STATE writes)
- **Linked gap:** GAP-006
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: parallel executions can overwrite each other's session state. Clarification C6: separate the expiry touch from state writes, and create the row safely.
- **Documents updated:** `Database.md` v0.6 §4.4 (`sessions.version`), §5.2 (`touch_session()`, `save_session()`); `Workflows.md` v0.6 §5.2 (write rules), §5.4 (`SESSION_BUSY`), WF-02, WF-06, WF-08
- **Architect refinement:** `touch_session()` also clears `last_intent` when an expired session is reset.
- **Build impact:** migrations 0004/0008; WF-02 uses `touch_session()`; WF-06/08 use `save_session()`; key `session.busy` in 4 languages.
- **Tests (M3):** slow voice note + quick read-only text → no busy; two state-changing messages → exactly one busy reply for the later writer, state = first writer's; two parallel first messages → one row, no error; message after expiry → empty state.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-019: Sole-Owner PIN recovery; remove `set_user_pin()`
- **Linked gap:** GAP-017
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: with one Owner, a forgotten Owner PIN blocks every PIN action; `set_user_pin()` had no purpose after CR-012.
- **Documents updated:** `Database.md` v0.6 §5.1 (`set_user_pin()` removed; `recover_user_pin()` added), §5.6 (revoke from `service_role` after the grants), §9 (operations row); `Workflows.md` v0.6 WF-40; `client-checklist.md` (second Owner recommended; identity-check method); M1 note v0.4 T2
- **Runbook (Implementer writes in M2):** ~~`implementation/runbooks/pin-recovery.md`~~ → renamed by CR-022 to `implementation/runbooks/account-recovery.md` (Procedure A: PIN recovery). Required steps: request → identity check outside the bot (call-back to the phone on file + the agreed company detail) → `recover_user_pin()` on PROD as the DB owner → check the `audit_log` row → tell the Owner to message the bot and set a new PIN → note it in the daily update.
- **Build impact:** migrations 0008/0008b; runbook.
- **Tests (M2):** clears PIN + lockout and writes 1 `audit_log` row; `service_role` via `/rpc` → permission denied; the Owner then sets a new PIN in chat.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-020: SQL hardening from the Implementer's v0.5 review
- **Linked gap:** GAP-018 (raised by the Implementer with reason and recommendation)
- **Reason:** (a) `check_permission()` allowed an active user without a PIN, so only the WF-02 gate stopped them; (b) a missing `tenant_settings` row silently disabled PIN lockout; (c) on Supabase, extensions belong in the `extensions` schema, otherwise `crypt()`/`gen_salt()` depend on the caller's search_path.
- **Documents updated:** `Database.md` v0.6 §4.1 (`create schema if not exists extensions`; both extensions `with schema extensions`; search_path note), §5.1 (`verify_user_pin()` raises when settings are missing), §5.4 (`check_permission()` requires `pin_hash is not null`); M1 note v0.4 T1 (verify the layout)
- **Build impact:** migrations 0001/0008.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-021: n8n database identity, WF-08 input checks, and SQL fixes
- **Linked gap:** GAP-020 (raised by the Implementer with reason and recommendation)
- **Reason:** (a) a Postgres-node login as the owner role would ignore RLS and the CR-003/009/019 revokes; (b) any input in a PIN state was treated as a PIN attempt (lockout without entering a PIN; voice/photo could reach STT/vision; stale state after expiry); (c) plain Postgres search_path breaks `crypt()` and `gin_trgm_ops`; (d) WF-02 writes state but wasn't listed; (e) a null language erased the stored one; (f) a misleading error for an unknown user; (g) Workflows.md layout.
- **Decision on (a):** option (i), **PostgREST with the `service_role` key only**, decided now. It holds for both hosted and self-hosted Supabase, so it doesn't wait for CR-005. Option (ii), a dedicated login role, is used only if the project ever leaves Supabase (raise a GAP).
- **Documents updated:** `Database.md` v0.7 (§4.6 qualified `gin_trgm_ops`; §5.1 `set search_path = public, extensions` on the 4 PIN functions, `verify_user_pin()` `user_not_found`; §5.2 `touch_session()` coalesce; §5.6 identity note); `Workflows.md` v0.7 (§2 database access row; §5.2 writer list; WF-00 step 6; WF-02; WF-08 input checks, Cancel button, expired handling; layout fix); M1 note v0.5 (§6.2, T2)
- **Build impact:** migrations 0006/0008; n8n credentials; WF-00, WF-02, WF-08.
- **Tests:** M1 T2 (n8n identity can't run operator functions; no owner login in n8n). M2: "cancel" in a PIN state is not counted; a voice note in a PIN state is deleted, not transcribed; a PIN after expiry → `draft.expired`, state cleared.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-022: Operator account move; user.remove cancels drafts
- **Linked gap:** GAP-019
- **Reason:** Implementer GAP-RESPONSE 2026-10-07: a sole Owner who loses their Telegram account can't get back in; accounts were assumed fixed for life.
- **Documents updated:** `Database.md` v0.7 §5.1 `move_user_channel()` (PIN kept by default; `p_clear_pin` optional; cancels open drafts; deletes the session; audit row), §5.6 revoke from `service_role`, §9 operations row; `Workflows.md` v0.7 WF-02 (fill `chat_id`), WF-40 (remove cancels drafts; account moves; `audit.summary` lists disabled users)
- **Runbook:** `implementation/runbooks/account-recovery.md`: Procedure A `recover_user_pin()`, Procedure B `move_user_channel()`; same identity check, audit check, and "tell the Owner" step. The new Telegram ID comes from the unknown-sender reply (CR-012), read to the operator during the identity call.
- **Tests (M2):** move → old ID rejected, new works, PIN kept; with `p_clear_pin` → new PIN required; open drafts cancelled and session deleted; used ID → `id_in_use`; `service_role` can't execute it; 1 `audit_log` row.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-023: New database objects start closed; small follow-ups
- **Linked gap:** GAP-021 (raised by the Implementer with reason and recommendation)
- **Reason:** (a) on Supabase, new tables in `public` are granted to `anon`/`authenticated` by default, and views bypass RLS (they run as their owner); our 0007 revoked only the named tables, so both views and any later table would be readable with the public anon key. Verified by the Architect on the Supabase docs page "Row Level Security" (2026-10-07). (b) The Zoho token functions lacked the CR-021 search_path fix. (c) WF-05 trigram search needs an RPC under CR-021. (d) Cancel was defined only for a draft.
- **Documents updated:** `Database.md` v0.8 (§3 convention; §4.7 default-privilege revokes; §5.3 search_path on `zoho_save_tokens()`/`zoho_get_credentials()`; §6 now `0010_views.sql` with `security_invoker = true` + revoke); `Workflows.md` v0.8 (WF-08 Cancel per state; WF-05 RPC note for M3); M1 note v0.6 (§3 rows 7/10, T2)
- **Build impact:** migrations 0007/0008/0010.
- **Tests (M1 T2):** anon can't read either view; a table created after 0007 is closed to anon; service_role still reads both views.
- **Deferred to the M3 note:** `search_entity_cache()` RPC specification.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —

### CR-024: Revoke existing sequences from anon/authenticated
- **Linked gap:** GAP-022 (raised by the Implementer with reason and recommendation)
- **Reason:** CR-023's default-privilege revoke covers only sequences created later. The identity sequences of `audit_log`, `message_log`, `ai_usage`, and `error_log` (created in 0005) kept Supabase's default grants, breaking the "everything starts closed" rule. Exposure is minimal (PostgREST doesn't expose sequences), but T2 wouldn't detect it.
- **Documents updated:** `Database.md` v0.9 §4.7 (`revoke all on all sequences in schema public from anon, authenticated;` after the table loop); M1 note v0.7 T2
- **Build impact:** migration 0007 (one line).
- **Tests (M1 T2):** no sequence in `public` is granted to `anon`/`authenticated`.
- **MoSCoW impact:** none
- **Approved by:** Architect · **Date:** 2026-10-07
- **Implemented by:** — · **Verified by:** — · **Date:** —
