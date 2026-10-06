# Implementation Plan — Zoho Books Chat Assistant

| | |
|---|---|
| **Owner** | Implementer |
| **Version** | 0.1 (Draft) |
| **Date** | 2026-10-06 |
| **Based on** | `PRD.md` 0.1 · `Architecture.md` 0.1 · `milestones.md` 0.1 · `Rules.md` 0.2 · `Database.md` 0.1 · `Workflows.md` 0.1 · `design/M1-infrastructure.md` 0.1 (Draft) · `gaps.md` (GAP-001…015) |
| **Status** | Planning only. **Nothing is built.** The project gate (`Rules.md` §3.1) is still closed |

> This is the Implementer's execution plan. It says **how and in what order** the Implementer builds what the Architect documents. It does not change any design. If this plan and a design document disagree, the design document wins and the Implementer raises a GAP (`Rules.md` §4.4).

---

## 1. How every milestone runs

Each milestone goes through the same ten steps. We never start step 3 of milestone *n+1* until step 10 of milestone *n* is done.

| # | Step | Who | Output |
|---|---|---|---|
| 1 | Architect publishes the Milestone Design Note (`docs/design/M<n>-*.md`) | Architect | Design note in Draft |
| 2 | Implementer reviews it against the Definition of Ready (`Rules.md` §3.3) and replies **Understood**, or sends Gap Queries | Implementer | `GAP-RESPONSE` / confirmation box |
| 3 | Gaps closed, design note **Approved** | Architect (+ Client if scope changes) | Approved note |
| 4 | Pre-flight: credentials, test data, and Zoho config ready in **DEV** | Implementer | Pre-flight checklist ticked |
| 5 | Build in DEV, in the build order of §4 below. Small commits, each referencing the FR/WF ID | Implementer | Git commits |
| 6 | Test every acceptance test (T1…Tn) in the design note | Implementer | `docs/test-evidence/M<n>/*.md` |
| 7 | Export workflows → `n8n/workflows/WF-xx_<name>.json` + paste into `Workflows.md` §7; bump register status to **Tested** | Implementer | Clean JSON (no creds IDs, no `pinData`) |
| 8 | Send the **VERIFICATION** box (`Memory.md` M-002) | Implementer | Handoff box |
| 9 | Architect review (read-only on n8n) → Accepted / GAPs | Architect | REVIEW box |
| 10 | Client sign-off, retro in `SelfImprovement.md` §5, `CHANGELOG.md` entry | Client · everyone | Milestone ✅ |

**Daily:** a short status update (done / next / blockers), per `Rules.md` §8. Blockers are raised within 1 working day.

## 2. Repository layout (Implementer territory)

Created at the start of M1, not before.

```
n8n/workflows/          WF-xx_<name>.json (exported, sanitised)
supabase/migrations/    <YYYYMMDDHHMM>_<description>.sql (from Database.md, never edited once applied)
supabase/seed/          seed templates with placeholders only (real values filled outside Git)
prompts/                intent-v<n>.md, receipt-v<n>.md, schemas/<intent>.json
tests/intent-set/       test set v1 (from the Architect's M0 deliverable) + runner workflow output
tests/receipts/         anonymised receipt samples (M7)
scripts/                export/sanitise + secrets-scan helpers
docs/test-evidence/M<n>/ evidence per acceptance test
CHANGELOG.md            releases
```

## 3. Cross-cutting rules the Implementer applies in every milestone

| Topic | How it is applied |
|---|---|
| Environments | DEV first, always. PROD only at M9 go-live. Topology waits on **GAP-011** |
| Starter template | Every sub-workflow starts from `Workflows.md` §4 (input mode per **GAP-013** outcome) |
| Error workflow | Every workflow sets *Error workflow = WF-90*. WF-90 stub exists from M1 |
| Zoho calls | Only through WF-92. No raw HTTP node to Zoho anywhere else |
| Writes | Only after `claim_pending_action()` returns a row. Every write → `audit_log` with Zoho record ID |
| Secrets | Credentials/env only. Before each commit: sanitise export (remove credential IDs, `pinData`) and run the secrets scan |
| Execution data | Success = none, error = all (except workflows that handle PINs/tokens — see GAP-001/012 outcome) |
| Permissions | Always `check_permission()`; never a hard-coded role check |
| Tenant | Every query filters by `tenant_id` |
| Test data | Anonymised only. No real customer names, TRNs, amounts, or Telegram IDs in Git |
| Prompts | Versioned in `prompts/`; any change re-runs the full test set (≥ 90%) |

---

## 4. Milestone execution plans

### M0 — Documentation & sign-off (current) · Implementer's part

The Implementer builds nothing in M0. Its tasks are:

| # | Task | Done when |
|---|---|---|
| 0.1 | Read all core docs and the M1 design note | ✅ 2026-10-06 |
| 0.2 | Answer the 6 Gap Queries that block M1: **GAP-005, 010, 011, 012, 013, 015** | GAP-RESPONSE sent (see §6) |
| 0.3 | Answer the other 9 Gap Queries (GAP-001…004, 006…009, 014) before the milestone each one blocks | GAP-RESPONSE sent |
| 0.4 | Review the four core docs and sign the Implementer column in `Rules.md` §9 | Signed by the Owner on the Implementer's behalf |
| 0.5 | Confirm the M1 design note (☐ Understood) once the gaps are closed | Box ticked in the note |
| 0.6 | Help the Owner collect M1 inputs from the client checklist: Zoho plan + DC, domain, hosting choice | Inputs available |

### M1 — Infrastructure & Zoho Books setup

**Entry gate:** core docs Approved · M1 design note Approved · GAP-005/010/011/012/013/015 closed · client chose the Zoho plan and data centre.

**Build order**

| # | Step | Detail | Test |
|---|---|---|---|
| 1.1 | Verify external facts | Fill design note §5 table with official doc URLs: Zoho DC domains, OAuth flow, scopes, rate limits, refresh-token minting limit, report endpoints/exports (Arch T2), Telegram `secret_token` | Table complete |
| 1.2 | Server | Per GAP-011 outcome: Docker, Caddy, HTTPS, firewall 22/80/443, SSH keys only, editor IP allow-list + 2FA, `/webhook/*` public | T9 |
| 1.3 | n8n config | Own Postgres; env vars from `Workflows.md` §2; `DB_ENCRYPTION_KEY` generated on the server; execution pruning 7 days | — |
| 1.4 | Supabase DEV | Create project; record Postgres version (GAP-015) | — |
| 1.5 | Migrations | Convert `Database.md` §4–§7 into migrations 1–9 **unchanged**, plus approved CR changes; apply to empty DEV | T1 |
| 1.6 | RLS check | `anon` key cannot read tables or call functions | T2 |
| 1.7 | Seed | Seed template in Git with placeholders; real values applied in a one-off session; Zoho secrets inserted encrypted, never committed | T1 |
| 1.8 | Zoho DEV org | Checklist §6.4: UAE edition, AED, Asia/Dubai, VAT 5% + zero/exempt tax IDs, tax treatments, emirates, currencies, numbering, Net 30, bilingual templates (draft), API user/role | Screenshots |
| 1.9 | Zoho OAuth client | Server-based client, least-privilege scopes, refresh token generated once and stored encrypted | — |
| 1.10 | WF-90 stub | Error Trigger → `error_log` → admin alert (no personal data) | Forced error |
| 1.11 | WF-91 Zoho Auth | Cached token; refresh < 5 min; single-flight per GAP-005 outcome | T3 |
| 1.12 | WF-92 Zoho Client | `{tenant_id, method, path, query, body, accept}`; backoff 2/4/8 s; one refresh on 401; error mapping §5.4; `increment_zoho_usage()`; 80% alert | T4, T5, T6 |
| 1.13 | Telegram DEV bot | BotFather; privacy mode; commands. Webhook registered in M2 (no WF-00 yet) | — |
| 1.14 | Backups | Daily `pg_dump` of n8n DB + Supabase, off-server, 14 days; **restore drill** to a scratch DB | T8 |
| 1.15 | Secrets scan | grep exports, Git, `error_log`, saved executions | T7 |
| 1.16 | Export + report | WF-90/91/92 JSON to Git and `Workflows.md` §7; VERIFICATION box | — |

**Exit:** E1 (create + read a contact in DEV via WF-92) and E2 (restore tested). **Needs from the Architect:** approved M1 note.

### M2 — Bot core

**Entry gate:** M1 signed off · M2 design note Approved · GAP-001, 003, 004, 008, 009, 014 closed · Client confirmed the Staff permission matrix (Q2).

| # | Step | Detail |
|---|---|---|
| 2.1 | Migrations for approved CRs (e.g. inbound update de-dup table, PIN-setup changes) | New migration files only |
| 2.2 | WF-01 Channel Adapter | Inbound Telegram → normalised message; outbound text/keys in EN/AR/UR/ur-Latn, buttons, documents, 4096-char split, plain text for AR/UR |
| 2.3 | WF-02 Auth & Session | Allow-list, status, lockout, session load/extend, `message_log` (with PIN masking per GAP-001) |
| 2.4 | WF-00 Telegram Inbound | Secret header check → 401; 200 at once; de-dup (GAP-004); route. Register webhook `/webhook/zb/telegram/dev` |
| 2.5 | WF-40 User Admin | `user.add/remove/set_role/reset_pin`, `audit.summary`; Owner + PIN; last-owner guard; first-PIN setup flow (GAP-003) |
| 2.6 | Interim command router | Until WF-04 exists (M3), WF-40 is reached through fixed commands (e.g. `/adduser`). **To confirm in the M2 design note** — raise a GAP if the note doesn't cover it |
| 2.7 | WF-90 full | User-facing `error.generic` in their language + admin alert |
| 2.8 | WF-93 Housekeeping | Daily 03:00 Asia/Dubai; `run_housekeeping()`; health summary |

**Exit tests:** unknown user rejected + logged · Owner adds Staff, sets role, resets PIN · lockout after 5 wrong PINs + Owner alert · errors reach admin chat · every message in `message_log`.

### M3 — AI understanding & confirm flow

**Entry gate:** M2 signed off · M3 design note Approved · intent catalogue + JSON schemas (M0 D3) and test set v1 (M0 D4) delivered by the Architect · client's **written OpenAI consent** (Q3) · GAP-002, 006, 007 closed.

| # | Step | Detail |
|---|---|---|
| 3.1 | Prompts + schemas | `prompts/intent-v1.md`, `prompts/schemas/*.json` exactly from the Architect's catalogue |
| 3.2 | Test runner | A DEV-only workflow that runs the test set through WF-04 and writes accuracy per language to evidence. Built first so every prompt change is measured |
| 3.3 | WF-04 Intent Parser | Structured output → schema validation → confidence < 0.7 → `AI_UNCLEAR` → `check_permission()` → `ai_usage` |
| 3.4 | WF-03 Media Normaliser | Telegram `getFile` → STT for voice; vision for photo/PDF; `ai_usage` |
| 3.5 | WF-05 Entity Resolver | `entity_cache` trigram → WF-92 search → 0 / 1 / 2–5 (buttons) / >5 |
| 3.6 | WF-06 Draft & Confirm | Missing-field loop; recompute totals/VAT; `pending_actions`; preview; Confirm/Edit/Cancel; PIN path (GAP-002); `claim_pending_action()` → WF-07 → `complete_pending_action()` → `audit_log` |
| 3.7 | WF-07 Router | Switch on intent domain; re-check permission; standard result |
| 3.8 | Stub domain | A DEV-only echo domain workflow so Confirm can be tested end-to-end before M4 |
| 3.9 | Tune | Iterate prompt to ≥ 90% on all 4 languages; each failure logged in `SelfImprovement.md` §7 |

**Exit tests:** ≥ 90% intent accuracy · voice notes work · ambiguity buttons · 30-min expiry · double Confirm → one record.

### M4 — Customers & services

| # | Step |
|---|---|
| 4.1 | Verify Zoho `/contacts` and `/items` fields for the UAE edition (tax treatment, TRN, place of supply, currency) and cite them |
| 4.2 | WF-10 Customers: create/update/get/search/balance; refresh `entity_cache` after writes |
| 4.3 | WF-11 Items: create/update/get/search; default VAT 5% tax ID |
| 4.4 | Router wiring + reply keys in 4 languages |

**Exit:** every customer and item intent end-to-end in DEV, ≥ 2 languages each, incl. UAE tax treatment and place of supply.

### M5 — Quotations & invoices

| # | Step |
|---|---|
| 5.1 | Totals/VAT engine (one Code node, shared by WF-12/13) checked against Zoho's own totals on a fixed sample set, incl. discount and rounding |
| 5.2 | WF-12 Quotations: create/update (editable status only)/get/list/accept/decline/convert (own preview + Confirm)/PDF |
| 5.3 | WF-13 Invoices: create/update/get/list (unpaid/overdue/paid/customer/date)/void (Owner + PIN)/PDF |
| 5.4 | PDF delivery via WF-01 with file name `INV-000123.pdf` |
| 5.5 | Bilingual EN/AR templates finalised in Zoho; **accountant approval** (Q4) |

**Exit:** AED + one foreign currency · VAT/totals match Zoho exactly · convert works · void Owner + PIN only · accountant approves the template.

### M6 — Payments received

WF-14: full, partial, and multi-invoice payments; PIN; receipt PDF; remaining balance after save.

### M7 — Expenses with receipt photos

| # | Step |
|---|---|
| 7.1 | Collect 20 anonymised receipts (mixed languages and quality) into `tests/receipts/` with expected fields |
| 7.2 | `prompts/receipt-v1.md`; measure field accuracy (≥ 85%) before wiring the write |
| 7.3 | WF-15: mapped expense accounts only; `POST /expenses` then attach the receipt file |

### M8 — Reports on request

Path decided by the M1 Arch T2 finding: either Zoho report endpoints + export, or JSON → our own PDF/XLSX rendering. WF-20 for the 4 reports, any date range, access rules enforced. Early spike in M1 (step 1.1) so M8 isn't a surprise.

### M9 — Hardening, UAT & go-live

| # | Step |
|---|---|
| 9.1 | Security review against `Architecture.md` §11 (evidence per line) |
| 9.2 | Load sanity test at 2× expected volume in DEV |
| 9.3 | One-page user guides EN/AR/UR + training |
| 9.4 | 1-week UAT in DEV with Owner + 1 Staff; defects triaged |
| 9.5 | PROD: client Zoho org (same §6.4 checklist), PROD bot, PROD Supabase, migrations, seed, credentials, webhook |
| 9.6 | Restore drill on PROD backups; monitoring + uptime check on |
| 9.7 | Go-live checklist; `CHANGELOG.md` v1.0.0 |

### Phase 2 and 3 (planned at high level only)

| M | Implementer focus | Hard dependency |
|---|---|---|
| M10 | New domain workflows (vendors, bills, vendor payments, POs, credit notes) through WF-92; no pipeline change | Client confirms PO use (L-002) |
| M11 | WF-30 + `report.schedule`; quick questions; AI cost report | `scheduled_reports` table (already in schema) |
| M12 | WF-00b + WhatsApp branch in WF-01; template submission | `marketing_opt_in` CR to `Database.md`; Meta business verification |
| M13 | Zoho accredited e-invoicing provider; new fields; status in bot | Revenue band (Q1) |
| M14–M15 | Admin panel; tenant onboarding/billing | Design notes not yet written |

---

## 5. Dependencies and risks (Implementer view)

| Item | Impact | Mitigation |
|---|---|---|
| GAP-011 topology unresolved | Can't start M1 server work | Recommendation in §6; client decides |
| Zoho plan/DC not chosen (Q5) | Blocks OAuth, domains, rate limits | Ask client early; DEV org can start on a trial |
| Report API availability (Arch T2) | M8 effort could double | Verify in M1 step 1.1 |
| OpenAI consent (Q3) | Blocks any real message to OpenAI in M3 | Only synthetic test set until consent |
| Intent catalogue + test set (M0 D3/D4) | Blocks M3 | Architect deliverable; request before M2 ends |
| Shared Contabo instance runs an unrelated active workflow | Resource sharing, accidental edits | Don't touch non-`WF-*` workflows; GAP-011 |
| n8n version not exposed via API | Node typeVersions may differ | Check in the n8n UI (*Settings → About*) at M1 start and record it |

## 6. Implementer positions on open Gap Queries (draft for GAP-RESPONSE)

These are the Implementer's recommendations. **They change nothing.** The Architect decides through a CR (`Rules.md` §4).

**Architect review 2026-10-07:** GAP-005 → CR-002, GAP-010 → CR-003, GAP-013 → CR-006, GAP-015 → CR-007 approved (docs updated, verified by the Implementer). GAP-011 → CR-004 awaits the Client. GAP-012 → CR-005 on hold. Clarifications C1 (GAP-005) and C2 (GAP-012) answered on 2026-10-07. The Implementer raised a new gap on CR-003 migration order (the revoke block sits in 0007 but must run after 0008).

**Later on 2026-10-07:** GAP-016 → CR-009 approved; C3 → CR-002 rev 3 approved (`last_alert_at`); GAP-005 closed. GAP-RESPONSEs sent for GAP-001, 002, 003, 004, 006, 007, 008, 009, 014 (`implementation/handoffs/`). They refine the rows above, and the handoff files take precedence. Main refinements:
- GAP-001: a separate PIN sub-workflow, and PROD execution saving off for WF-00/01/02.
- GAP-003: PIN setup is driven by `pin_hash is null` and confirmed with `pin_pending_hash`; new users stay `active`.
- GAP-006: on conflict the user is asked to resend (no merge).
- GAP-014: `check_permission()` honours `intents.active`.

| GAP | Recommendation | Impact |
|---|---|---|
| 001 | Change: when `sessions.state.awaiting = 'pin'`, WF-02 logs text as `[REDACTED-PIN]`; PIN check runs in a sub-workflow with execution saving **off** (success and error); WF-00 error saving off in PROD, with WF-90 logging a redacted payload; delete the PIN message via Telegram `deleteMessage` | WF-00/02/06; no schema change |
| 002 | Change: one atomic function `verify_pin_for_action(user, action, pin)` that verifies (with lockout) and sets `pin_verified_at` in the same transaction. `pin_hash is null` returns `pin_not_set` and doesn't count as a failure | `Database.md` §5.1; WF-06 |
| 003 | Change: new users start as `invited`; WF-02 lets `invited` users into the PIN-setup flow only (enter + repeat) → `active`. Seed Owner as `invited` too | WF-02, WF-40, seed |
| 004 | Change: table `inbound_updates (tenant_id, channel, update_id, received_at)` with a unique key; WF-00 inserts with `on conflict do nothing` and stops if 0 rows. Purged after 7 days by housekeeping | New migration; WF-00, WF-93 |
| 005 | Change: claim-based single flight. Add `refresh_lock_until timestamptz` to `zoho_connections`; a function claims it with a conditional update (`… where refresh_lock_until is null or < now() returning`). Winner refreshes; others wait ~1 s and re-read the token (max 5 tries). Advisory locks are not suitable because each n8n Postgres node call is its own transaction | 1 column + 1 function; WF-91 |
| 006 | Change: add `sessions.version int`; every state write is `update … where version = $old` (optimistic). On conflict, re-read and re-apply, or reply "one moment, still working on your last message" | 1 column; WF-02/06 |
| 007 | Change: notify on next interaction **and** a small schedule (every 5 min, in WF-93 or a new ops workflow) that expires drafts and edits the preview to "expired" through WF-01 | WF-93 or new WF; FR-3.3 |
| 008 | Change: redact instead of delete. After `draft_retention_days`, clear `payload` (keep `meta` only) and `preview_text`; keep the row for the audit link | `run_housekeeping()` |
| 009 | Change: purge null-tenant `error_log` rows with a fixed default (180 days). Phase 1 tenant comes from env `TENANT_ID`; a bot→tenant mapping is a SaaS item (M15) | `run_housekeeping()` |
| 010 | Change (agree): `revoke execute on all functions in schema public from public, anon, authenticated` + matching `alter default privileges`; `grant execute … to service_role` | Migration 0007/0008 |
| 011 | **Client decision.** Recommend: a **separate PROD n8n instance** for the client bot (own container + domain, ideally own VPS); use the existing Contabo instance as **DEV**. Reason: `ENV`/`TENANT_ID` env vars, credentials, and the error workflow are instance-wide, and the instance runs an unrelated active workflow | Infra cost of one more instance |
| 012 | Change: store `client_secret` and `refresh_token` in **Supabase Vault** (key managed by Supabase, never passed in SQL); one `security definer` function readable only by `service_role`; WF-91 execution saving off. Verify Vault is available on the chosen Supabase hosting | `Database.md` §4.2/§5.3; WF-91; `DB_ENCRYPTION_KEY` may no longer be needed |
| 013 | Change: use typed inputs ("Define using JSON example") on the Execute Workflow Trigger. The instance is n8n ≥ 1.119 (it hides its version from the API, which started in 1.119), so typed inputs are available. Keep the `Validate input` node for business rules | `Workflows.md` §4 template |
| 014 | Keep seeding future intents but with `active = false` until their milestone. Change `report.schedule` to `needs_pin = true` to match `report.run` | Seed §7 |
| 015 | Use the Postgres major version that Supabase provisions for the new project; record it in `M1/migrations.md`; test migrations against that exact version. Docs should state the tested version | Docs only |

## 7. Change history

| Version | Date | Change | By |
|---|---|---|---|
| 0.1 | 2026-10-06 | First plan from the M0 document set | Implementer |
| 0.2 | 2026-10-07 | §6: Architect review outcome (CR-002…007), C1/C2 answered, new gap on CR-003 order | Implementer |
