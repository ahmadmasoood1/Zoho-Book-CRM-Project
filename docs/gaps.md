# Gaps Log — Zoho Books Chat Assistant

| | |
|---|---|
| **Owner** | Architect |
| **Process** | `Rules.md` §4 Gap Protocol: detect → ask the Implementer *why* → justify → CR → docs first → build |
| **Related** | `changes.md`, `SelfImprovement.md` |

> **No gap below has changed any document or build.** Each one waits for the Implementer's response (`Rules.md` §4.1). Gaps marked **Client** need a client decision; the Implementer still gives feedback first.

## Register

| ID | Title | Where | Severity | Blocks | Status |
|---|---|---|---|---|---|
| GAP-001 | PIN text can be stored in `message_log` and n8n execution data | WF-02, WF-06, Workflows.md §2 | High | M2 | Justified → CR-010 rev 2 approved · Closed |
| GAP-002 | Nothing sets `pending_actions.pin_verified_at` | Database.md §5.2, WF-06 | High | M3 | Justified → CR-011 approved · Closed |
| GAP-003 | First-time PIN setup isn't reachable/specified | WF-02, WF-40, `users.status` | Medium | M2 | Justified → CR-012 approved · Closed (follow-up GAP-017) |
| GAP-004 | No de-duplication of inbound Telegram updates | WF-00, Database.md | Medium | M2 | Justified → CR-013 approved · Closed |
| GAP-005 | Concurrent Zoho token refresh has no lock | WF-91, Database.md §5.3 | Medium | M1 | Justified → CR-002 rev 3 approved · Closed |
| GAP-006 | Parallel messages from one user race on `sessions.state` | WF-02, WF-06 | Medium | M3 | Justified → CR-018 approved · Closed |
| GAP-007 | "User is told" when a draft expires has no mechanism | FR-3.3, WF-93 | Medium | M3 | Justified → CR-014 approved · Closed |
| GAP-008 | 7-day draft deletion conflicts with the audit-reference rule | NFR-5, `run_housekeeping()` | Low | M2 | Justified → CR-015 approved · Closed |
| GAP-009 | Logs without `tenant_id` are never purged; unknown senders need a tenant | Database.md §4.5, §5.5 | Low | M2 | Justified → CR-016 approved · Closed |
| GAP-010 | EXECUTE on functions not revoked from `anon`/`authenticated` | Database.md §4.7 | Low | M1 | Justified → CR-003 approved · Closed |
| GAP-011 | DEV/PROD environment topology on the n8n instance undefined | Architecture §12, Workflows.md §2 | High · **Client** | M1 | Justified → CR-004 awaiting Client |
| GAP-012 | `DB_ENCRYPTION_KEY` passed as a SQL argument can leak to logs | Database.md §5.3, WF-91 | Medium | M1 | Justified → CR-008 approved; CR-005 on hold (hosting) |
| GAP-013 | Sub-workflow inputs use "passthrough" instead of typed inputs | Workflows.md §4 starter template | Low | M1 | Justified → CR-006 approved · Closed |
| GAP-014 | `report.schedule` seeded but not in the intent catalogue; PIN rule differs | Database.md §7, Architecture §8 | Low | M2 | Justified → CR-017 approved · Closed |
| GAP-015 | Postgres version stated inconsistently | Database.md header vs PROJECT_OVERVIEW §7 | Low | M1 | Justified → CR-007 approved · Closed |
| GAP-016 | CR-003 function revokes would run before the functions exist | Database.md §4.7, M1 note §3 | Medium | M1 | Raised by Implementer → CR-009 approved · Closed |
| GAP-017 | No PIN recovery when the only Owner forgets their PIN | WF-40, `set_user_pin()`, Rules §4.4 | Medium | M2 | Justified → CR-019 approved · Closed |
| GAP-018 | SQL review findings: PIN not required in check_permission; silent lockout failure; extensions schema | Database.md v0.5 §4.1, §5.1, §5.4 | Low | M1 | Raised by Implementer → CR-020 approved · Closed |
| GAP-019 | No procedure when an Owner loses their Telegram account | WF-40, `users.channel_user_id/chat_id` | Low | M2 | Justified → CR-022 approved · Closed |
| GAP-020 | v0.6 review: n8n DB identity undefined; non-PIN input counted as PIN; search_path; minor | Database.md v0.6, Workflows.md v0.6 | Medium | M1/M2 | Raised by Implementer → CR-021 approved · Closed |
| GAP-021 | v0.7 review: views and later tables readable with the anon key; search_path; Cancel per state | Database.md §4.7, §6; Workflows.md WF-08 | Medium | M1 | Raised by Implementer → CR-023 approved · Closed |
| GAP-022 | Existing sequences keep anon/authenticated grants | Database.md §4.7 | Low | M1 | Raised by Implementer → CR-024 approved · Closed |

---

### GAP-001: PIN text can be stored in `message_log` and n8n execution data
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** WF-02 steps (`Workflows.md` §6), WF-06 Confirm steps, conventions table (`Workflows.md` §2)
- **What I observed:** WF-02 writes every inbound message to `message_log` *before* WF-06 knows the message is a PIN reply. Separately, the convention "save failed executions = on" means a failed run of WF-06 keeps the PIN in n8n's execution data.
- **Expected (per docs):** "Logs never contain PINs or tokens" (Architecture §11); FR-1.5.
- **Question to Implementer:** Why isn't there a step that masks the text when the session is in an `awaiting_pin` state? How do you plan to keep the PIN out of saved executions?
- **Severity:** High · **Status:** Justified → CR-010 rev 2 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-001.md`.

#### Decision
- **Outcome:** CR-010 approved for points 1–3, 5, 6 (WF-08 PIN Handler, redaction, PROD no-save). Point 4 (bare digits treated as a PIN) pending **C4**.
- **Decided by:** Architect · **Date:** 2026-10-07

#### Follow-up (2026-10-07)
- **Outcome:** C4 answered: narrow late-PIN rule → **CR-010 rev 2**. Closed.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-002: Nothing sets `pending_actions.pin_verified_at`
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `Database.md` §5.1–§5.2, WF-06
- **What I observed:** `claim_pending_action()` requires `pin_verified_at is not null` when `needs_pin`. `verify_user_pin()` checks the PIN per *user* and never touches `pending_actions`, and no documented step or function sets `pin_verified_at`. Also, `verify_user_pin()` counts a user with `pin_hash is null` as a failed attempt.
- **Expected (per docs):** FR-1.3, FR-3.4: a PIN-protected draft can be executed exactly once after a correct PIN.
- **Question to Implementer:** Why is setting `pin_verified_at` left out? Should it be part of a single function (verify + mark draft) so it is atomic? How should a user with no PIN be handled?
- **Severity:** High · **Status:** Justified → CR-011 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-002.md`.

#### Decision
- **Outcome:** CR-011 approved. `verify_pin_for_action()` reuses `verify_user_pin()` inside one transaction.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-003: First-time PIN setup isn't reachable/specified
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** WF-02 (`status='active'` lookup), WF-40 ("new user sets their own PIN on first message"), `users.status` enum (`invited` exists)
- **What I observed:** If WF-40 creates users as `invited`, WF-02 rejects them as unknown, so they never reach PIN setup. If they're created as `active`, the setup flow (prompt, confirm PIN, when `invited` becomes `active`) isn't specified anywhere. The first Owner from the seed also has no PIN.
- **Expected (per docs):** FR-1.4, FR-1.5; M2 exit criteria.
- **Question to Implementer:** Why is the onboarding flow not in the WF-02/WF-40 specs, and which status should a new user start with?
- **Severity:** Medium · **Status:** Justified → CR-012 approved · Closed (follow-up GAP-017)

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-003.md`.

#### Decision
- **Outcome:** CR-012 approved. Unknown-sender buttons: not in Phase 1. `set_user_pin()` and sole-Owner recovery → **GAP-017**.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-004: No de-duplication of inbound Telegram updates
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** WF-00, `Database.md` (no table stores `update_id`)
- **What I observed:** WF-00 replies 200 straight away, but Telegram still redelivers updates if the webhook times out or n8n restarts mid-request. Nothing stores the processed `update_id`. A redelivered message would create a second draft or preview. Writes stay safe because of `claim_pending_action`, but the user would see duplicates.
- **Expected (per docs):** FR-3.4 spirit (no duplicates); NFR-1.
- **Question to Implementer:** Why is inbound idempotency left out? Is a unique `(tenant_id, channel, update_id)` record needed?
- **Severity:** Medium · **Status:** Justified → CR-013 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-004.md`.

#### Decision
- **Outcome:** CR-013 approved. Table placed in 0005 so RLS (0007) and privileges (0008b) apply in order (L-005).
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-005: Concurrent Zoho token refresh has no lock
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** WF-91, `zoho_get_credentials()` / `zoho_save_tokens()`
- **What I observed:** Two executions that start together with an expired token will both refresh. Zoho limits how many access tokens one refresh token can mint in a short window (verify the current limit in Zoho Accounts OAuth docs during M1). Bursts can therefore produce `ZOHO_AUTH` errors.
- **Expected (per docs):** Architecture §9 (cache until expiry), §12 reliability.
- **Question to Implementer:** Why is there no single-flight mechanism, such as `pg_advisory_xact_lock` or a `refreshing_until` column?
- **Severity:** Medium · **Status:** Justified → CR-002 approved

#### Implementer response (2026-10-07)
- **Reason:** Not specified in the original design; WF-91 was a simple read-check-refresh.
- **Feedback / recommendation:** Claim-based single flight (`refresh_lock_until` + conditional-update claim); losers wait ~1 s and re-read (max 5). Advisory locks unsuitable.

#### Decision
- **Outcome:** Change via **CR-002** (approved). Open clarification: behaviour after 5 failed tries and when the winner's refresh fails (see box).

#### Clarification C1 (Implementer, 2026-10-07)
- Minting limit verified: 10 tokens / 10 min (Zoho Accounts OAuth docs).
- C1a: losers return `ZOHO_AUTH` / `refresh_wait_timeout` and never alert. C1b: new `zoho_refresh_failed()` with permanent/transient handling and a 5 s cool-down; claim only active connections.
- **Decision:** accepted → **CR-002 rev 2**. Still open, **C3**: where the last-alert time is stored for the "no repeat alert within 10 min" rule.

#### Clarification C3 (Implementer, 2026-10-07)
- **Reason:** oversight; C1b described the behaviour but not where its state lives.
- **Recommendation:** option (a): `last_alert_at` column; `zoho_refresh_failed()` returns `{alert, status}` with the decision made in the same UPDATE. Option (b) rejected.
- **Decision:** accepted → **CR-002 rev 3**. GAP-005 **closed** (design complete; build verification at M1).
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-006: Parallel messages from one user race on `sessions.state`
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** WF-02 / WF-06 session read-modify-write
- **What I observed:** n8n runs each webhook call in its own execution. A user who sends a voice note and then a quick text (or taps a button twice) gets parallel executions that read and overwrite the same `sessions.state`.
- **Expected (per docs):** FR-2.5 (one question at a time), FR-3.2.
- **Question to Implementer:** Why isn't per-user serialisation or optimistic locking (a `version` column) specified?
- **Severity:** Medium · **Status:** Justified → CR-018 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-006.md`.

#### Decision
- **Outcome:** Justified. **CR-018** drafted, on hold for clarification **C6** (which session writes bump the version).
- **Decided by:** Architect · **Date:** 2026-10-07

#### Follow-up (2026-10-07)
- **Outcome:** C6 answered: TOUCH vs STATE writes, safe row creation → **CR-018** approved. Closed.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-007: "User is told" when a draft expires has no mechanism
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** FR-3.3, M3 exit criteria, WF-93 (runs daily at 03:00)
- **What I observed:** Drafts expire after 30 minutes, but the only expiry job runs once a day, and no workflow sends an expiry notice. At present the user finds out only when they press Confirm (`DRAFT_EXPIRED`).
- **Expected (per docs):** FR-3.3: "the user is told".
- **Question to Implementer:** Was a "tell on next interaction" approach intended? If not, which workflow and schedule should send the notice?
- **Severity:** Medium · **Status:** Justified → CR-014 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-007.md`.

#### Decision
- **Outcome:** CR-014 approved, new WF-94 Draft Expiry. A 30–35 min notice meets FR-3.3.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-008: 7-day draft deletion conflicts with the audit-reference rule
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** NFR-5 ("drafts are deleted after 7 days"), `run_housekeeping()`
- **What I observed:** Housekeeping skips any draft referenced by `audit_log`. Every executed draft is audited, so executed drafts and their full payloads (customer and amount data) are effectively kept for 5 years.
- **Expected (per docs):** NFR-5, plus "no accounting data outside Zoho except logs and drafts".
- **Question to Implementer:** Why is this acceptable, or should drafts be redacted (payload cleared) instead of deleted?
- **Severity:** Low · **Status:** Justified → CR-015 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-008.md`.

#### Decision
- **Outcome:** CR-015 approved; PRD NFR-5 reworded.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-009: Logs without `tenant_id` are never purged; unknown senders need a tenant
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `error_log.tenant_id` (nullable) and the purge join in `run_housekeeping()`; `message_log.tenant_id not null` for unknown senders
- **What I observed:** `error_log` rows with a null tenant never match the join, so they are never deleted. For unknown senders, the tenant must come from the bot. That works with the `TENANT_ID` env in Phase 1, but no bot→tenant mapping is documented for SaaS.
- **Question to Implementer:** Why are these cases not covered? Is a default retention for null-tenant rows enough?
- **Severity:** Low · **Status:** Justified → CR-016 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-009.md`.

#### Decision
- **Outcome:** CR-016 approved; M15 note added to milestones.md.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-010: EXECUTE on functions not revoked from `anon`/`authenticated`
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `Database.md` §4.7
- **What I observed:** Table privileges are revoked, but Postgres grants EXECUTE on new functions to PUBLIC by default, and Supabase exposes them through `/rpc`. Today they are SECURITY INVOKER, so table privileges still block misuse. A later `security definer` change would silently open `set_user_pin` and `zoho_get_credentials`.
- **Question to Implementer:** Why not add `revoke execute on all functions in schema public from public, anon, authenticated` (plus default privileges) as defence in depth?
- **Severity:** Low · **Status:** Justified → CR-003 approved

#### Implementer response (2026-10-07)
- **Reason:** Oversight: function defaults were missed.
- **Feedback / recommendation:** Revoke EXECUTE from public/anon/authenticated + default privileges; grant to service_role.

#### Decision
- **Outcome:** Change via **CR-003** (approved).
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-011: DEV/PROD environment topology on the n8n instance undefined — **Client**
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** Architecture §12 (Environments), Workflows.md §2 (credentials `… (DEV)` / `… (PROD)`, webhook `/webhook/zb/telegram/<env>`), Arch T3/T4
- **What I observed:** The docs require separate DEV and PROD but don't say whether that means two n8n instances or one. The connected instance (`vmi3618607.contaboserver.net`, Contabo) already runs unrelated workflows, including one active ("CRM Entry Inventory Management v3.7"). With one instance, workflow IDs, the `ENV` variable, and the error workflow are shared between DEV and PROD, and an unrelated workflow failure shares resources with the bot.
- **Expected (per docs):** "Never test against the client's live org"; NFR-2, NFR-6.
- **Question to Implementer:** Why isn't the topology stated? Recommend one: (a) two n8n instances (DEV and PROD), or (b) one instance with duplicated `WF-xx [DEV]` workflows. Should the client bot share a VPS with unrelated workloads?
- **Severity:** High · **Status:** Justified → CR-004 awaiting Client → then Client decision

#### Implementer response (2026-10-07)
- **Reason:** Hosting was an open question when the docs were written.
- **Feedback / recommendation:** Two instances: Contabo = DEV, new PROD instance; separate Supabase projects.

#### Decision
- **Outcome:** **CR-004** drafted; needs Client approval (cost).
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-012: `DB_ENCRYPTION_KEY` passed as a SQL argument can leak to logs
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `zoho_get_credentials(p_tenant, p_key)`, `zoho_save_tokens(…, p_key)`, WF-91
- **What I observed:** The key travels as a query parameter. It can appear in Postgres statement logs (`log_statement`, `pg_stat_statements`, Supabase logs explorer) and in saved failed n8n executions. The decrypted refresh token also appears in node output.
- **Expected (per docs):** Architecture §11 ("Logs never contain PINs or tokens"); Rules §7.6.
- **Question to Implementer:** Why is this acceptable, or is an alternative preferred, such as Supabase Vault, n8n-side encryption, or redacting the node output and disabling statement logging for that role?
- **Severity:** Medium · **Status:** Justified → CR-005 on hold (clarification + hosting)

#### Implementer response (2026-10-07)
- **Reason:** pgcrypto + app key chosen for portability; logging paths not considered.
- **Feedback / recommendation:** Supabase Vault + security definer function; WF-91 saves no executions; pgcrypto fallback.

#### Decision
- **Outcome:** **CR-005** on hold: Supabase hosting decision + clarification on token exposure via WF-92.

#### Clarification C2 (Implementer, 2026-10-07)
- C2a: WF-91/92 save no executions; WF-92 handles its own errors; sanitised `error_log`; whitelisted return; WF-90 redaction.
- C2b: Vault design (uuid secret references, keyless `security definer` functions, single-transaction token save).
- **Decision:** C2a accepted → **CR-008** (approved, independent of hosting). C2b accepted as the CR-005 design; CR-005 stays on hold until open question #5 is decided.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-013: Sub-workflow inputs use "passthrough" instead of typed inputs
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `Workflows.md` §4 starter template (`executeWorkflowTrigger`, `inputSource: passthrough`, typeVersion 1.1)
- **What I observed:** Current n8n versions support typed inputs ("Define using fields below" or a JSON example) on the Execute Workflow Trigger. Typed inputs document the contract inside n8n and show it in the caller's mapping UI. Passthrough leaves all checking to the `Validate input` Code node.
- **Question to Implementer:** Why passthrough? Does the installed n8n version support typed inputs?
- **Severity:** Low · **Status:** Justified → CR-006 approved

#### Implementer response (2026-10-07)
- **Reason:** Template written for the older trigger behaviour.
- **Feedback / recommendation:** Typed inputs (JSON example); keep `Validate input` for business rules.

#### Decision
- **Outcome:** Change via **CR-006** (approved); Architect verified the node options.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-014: `report.schedule` seeded but not in the intent catalogue; PIN rule differs
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `Database.md` §7.1/§7.2 vs `Architecture.md` §8
- **What I observed:** `report.schedule` (M11) is seeded but missing from the Architecture catalogue. The seed gives `report.run` `needs_pin = true` but `report.schedule` `needs_pin = false`. Scheduling then delivers financial reports later without a PIN.
- **Question to Implementer:** Is seeding M11 intents in M1 intended? Was the PIN difference deliberate?
- **Severity:** Low · **Status:** Justified → CR-017 approved · Closed

#### Implementer response (2026-10-07)
- See `implementation/handoffs/HANDOFF_2026-10-07_Implementer-to-Architect_GAP-RESPONSE_GAP-014.md`.

#### Decision
- **Outcome:** CR-017 approved; Architecture §8 lists `report.schedule`.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-015: Postgres version stated inconsistently
- **Found by:** Architect · **Date:** 2026-10-06
- **Where:** `Database.md` header ("PostgreSQL 15+") vs `PROJECT_OVERVIEW.md` §7 ("tested on Postgres 16")
- **Question to Implementer:** Which version will DEV/PROD Supabase run, and was the SQL tested on it?
- **Severity:** Low · **Status:** Justified → CR-007 approved

#### Implementer response (2026-10-07)
- **Reason:** SQL tested on 16; Supabase version unknown until provisioned.
- **Feedback / recommendation:** State the tested version; record the provisioned version at M1.

#### Decision
- **Outcome:** Change via **CR-007** (approved).
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-016: CR-003 function revokes would run before the functions exist
- **Found by:** Implementer · **Date:** 2026-10-07 (raised under `Rules.md` §4.4, reason and recommendation included)
- **Where:** `Database.md` §4.7 (0007_rls.sql); M1 note §3 migration order
- **What was observed:** the privilege block sat in 0007, which runs before 0008. `revoke`/`grant … on all functions` would only cover the two trigger functions, and the default revoke would leave every 0008 function without a grant to n8n's role ("permission denied").
- **Expected (per docs):** CR-003 intent: `anon`/`authenticated` cannot execute functions; n8n's role can.
- **Severity:** Medium (caught before build) · **Status:** Closed via CR-009

#### Implementer response
- **Reason:** the block was added to the existing security section without rechecking the file order.
- **Feedback / recommendation:** move it to its own migration after the functions; add a default grant to `service_role`; grant to any other n8n DB role too and name it in `Workflows.md` §2.

#### Decision
- **Outcome:** Change via **CR-009** (approved). Architect accepts the root cause: the error was in the Architect's CR-003 text.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-017: No PIN recovery when the only Owner forgets their PIN
- **Found by:** Architect · **Date:** 2026-10-07 (while reviewing the GAP-003 response)
- **Where:** WF-40 `user.reset_pin` (Owner + PIN), `set_user_pin()` (no longer called after CR-012), `Rules.md` §4.4
- **What I observed:** `user.reset_pin` needs the Owner's own PIN. Phase 1 has one Owner. If that Owner forgets their PIN, or is locked out repeatedly, nobody can reset it in chat, and every PIN-protected action (void, payments, reports, user admin) is blocked. No operator recovery procedure is documented, and `set_user_pin()` is left with no stated purpose.
- **Expected (per docs):** FR-1.4, NFR-2 availability; the Owner can always regain control.
- **Question to Implementer:** Why is there no recovery path? Recommend one, e.g. an operator runbook that clears `pin_hash` with an `audit_log` row, versus a second Owner. And should `set_user_pin()` be removed or kept for that runbook?
- **Severity:** Medium · **Status:** Justified → CR-019 approved · Closed

#### Follow-up (2026-10-07)
- **Outcome:** Option (a) operator recovery with `recover_user_pin()` (revoked from service_role); `set_user_pin()` removed; second Owner recommended in the client checklist → **CR-019**. Lost Telegram account → **GAP-019**. Closed.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-018: SQL review findings in Database.md v0.5
- **Found by:** Implementer · **Date:** 2026-10-07 (raised under `Rules.md` §4.4 with reason and recommendation)
- **Where:** `Database.md` v0.5 §4.1, §5.1 `verify_user_pin()`, §5.4 `check_permission()`
- **What was observed:** (a) an active user without a PIN passes `check_permission()`; (b) with no `tenant_settings` row, lockout silently never happens; (c) extensions may install into `public`, so `crypt()` depends on the search_path.
- **Severity:** Low · **Status:** Closed via CR-020

#### Architect answer to "why"
- (a) and (b) were oversights in the Architect's v0.5 SQL; (c) was not checked against Supabase's schema layout.

#### Decision
- **Outcome:** all three recommendations accepted → **CR-020**.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-019: No procedure when an Owner loses their Telegram account
- **Found by:** Implementer (raised in the GAP-017 response, no proposal) · logged by Architect · **Date:** 2026-10-07
- **Where:** WF-40; `users.channel_user_id`, `users.chat_id`; PIN-recovery runbook (CR-019)
- **What was observed:** if an Owner loses their Telegram account (new phone number, deleted account), their new account is an unknown sender. Only an Owner can add users, so a sole Owner can't get back in.
- **Expected (per docs):** FR-1.4; NFR-2.
- **Question to Implementer:** Please recommend the procedure: for example, extend the operator runbook with a function that moves `channel_user_id`/`chat_id` to the new account (identity check, `audit_log` row, PIN cleared so a new PIN is set), and say whether the old account's sessions and pending drafts are cancelled.
- **Severity:** Low · **Status:** Justified → CR-022 approved · Closed

#### Implementer response (2026-10-07)
- **Reason:** accounts were treated as fixed for life; a sole Owner can't add themselves.
- **Recommendation:** operator-only `move_user_channel()` (PIN kept by default), combined account-recovery runbook, Staff moves via user.remove + user.add.

#### Decision
- **Outcome:** accepted → **CR-022**.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-020: Review findings on Database.md / Workflows.md v0.6
- **Found by:** Implementer · **Date:** 2026-10-07 (raised under `Rules.md` §4.4 with reason and recommendation)
- **Where:** `Workflows.md` §2 credentials, WF-00/02/08, §5.2; `Database.md` §4.6, §5.1, §5.2
- **What was observed:** (a) MEDIUM: the n8n database identity was undefined; an owner login would bypass RLS and the revokes. (b) MEDIUM: every input in a PIN state was treated as a PIN attempt. (c)–(f) LOW: search_path, WF-02 state write not listed, language overwrite, misleading error. (g) cosmetic layout.
- **Severity:** Medium · **Status:** Closed via CR-021

#### Architect answer to "why"
- (a) The credential was named but its database role was never pinned down. I didn't connect the privilege model (CR-003/009/019) to how n8n logs in.
- (b) WF-08 was designed around the happy path; non-PIN input in a PIN state wasn't considered.

#### Decision
- **Outcome:** all recommendations accepted → **CR-021**; (a) decided now as option (i), PostgREST with the `service_role` key.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-021: Views and later objects are not covered by the 0007 revokes
- **Found by:** Implementer · **Date:** 2026-10-07 (raised under `Rules.md` §4.4 with reason and recommendation)
- **Where:** `Database.md` v0.7 §4.7, §6, §5.3; `Workflows.md` WF-05, WF-08
- **What was observed:** (a) MEDIUM: `v_daily_health` and `v_tenants_without_owner` (tenant names, daily counts) readable through `/rest/v1` with the anon key; any later table open the same way. (b)–(d) LOW: search_path on the Zoho token functions; WF-05 RPC for M3; Cancel per PIN state.
- **Severity:** Medium · **Status:** Closed via CR-023

#### Architect answer to "why"
- I designed the revokes as a fixed list of named tables, assuming objects start closed. On Supabase they start open, and views run as their owner. I didn't check Supabase's default grants (same root cause as L-007: privileges designed without checking the platform's runtime behaviour).

#### Decision
- **Outcome:** all recommendations accepted → **CR-023**; verified on the Supabase docs.
- **Decided by:** Architect · **Date:** 2026-10-07

### GAP-022: Existing sequences keep anon/authenticated grants
- **Found by:** Implementer · **Date:** 2026-10-07 (raised under `Rules.md` §4.4 with reason and recommendation)
- **Where:** `Database.md` v0.8 §4.7
- **What was observed:** the 0007 loop revokes tables only, and the default-privilege revoke covers only later sequences. The 4 identity sequences from 0005 stay granted.
- **Severity:** Low · **Status:** Closed via CR-024

#### Architect answer to "why"
- Not intended. When adding CR-023 I didn't separate existing objects from future ones; the table loop handles existing tables, but nothing handled existing sequences.

#### Decision
- **Outcome:** one-line fix accepted → **CR-024**; T2 extended.
- **Decided by:** Architect · **Date:** 2026-10-07
