# Workflows — Zoho Books Chat Assistant (n8n)

| | |
|---|---|
| **Version** | 0.8 (Draft for sign-off) |
| **Date** | 7 October 2026 |
| **Platform** | n8n (self-hosted, Docker) |
| **Owner** | Architect (specs) · Implementer (builds and exports JSON) |
| **Related docs** | `Architecture.md` §3–§5, `Database.md`, `milestones.md`, `Rules.md` |

> **Purpose of this file:** this is the single register of **every n8n workflow** in the project. It holds the specification of each workflow and its **exported JSON**.
>
> **Status of JSON sections:** under `Rules.md` §3 (no implementation before documentation), each workflow's JSON is added **only after** its milestone design note is approved and the workflow is built and tested in DEV. Until then, the JSON section of a workflow says `Pending — <milestone>`. The **starter template** in §4 is the agreed skeleton every workflow begins from.

---

## 1. How to use this file

1. **Before building:** the Architect fills in the workflow's spec (§6). The Implementer confirms it.
2. **After building and testing in DEV:** the Implementer exports the workflow from n8n (`⋯` → *Download*). They paste the JSON into that workflow's **Exported JSON** block and update the version and status in the register (§3).
3. **Every later change:** follows the Gap Protocol / CR process in `Rules.md`. Replace the JSON, bump the version, and add a line to the workflow's change history.
4. **Git is the backup.** The same JSON is also committed to `n8n/workflows/WF-xx_<name>.json`. If this file and Git disagree, raise a GAP.
5. **Before pasting JSON, remove:** credential IDs specific to one environment (replace them with the credential *name*), `pinData`, and any test data with real customer information.

## 2. Conventions

| Topic | Rule |
|---|---|
| Naming | `WF-<nn> <Name>`, e.g. `WF-13 Invoices`. Same name in n8n, Git file, and this document |
| Tags | `zb-assistant`, plus layer tag: `channel`, `core`, `domain`, `zoho`, `ops` |
| Settings | Execution order `v1`; **Error workflow = WF-90**; save failed executions = on; save successful executions = off in PROD (privacy). **Exception (CR-008): WF-91 and WF-92** save no execution data at all (`saveDataErrorExecution` = none, `saveDataSuccessExecution` = none, `saveManualExecutions` = false, `saveExecutionProgress` = false), because their item data holds the Zoho access token. **Exception (CR-010), PROD only: WF-00, WF-01, WF-02, WF-08** use the same four settings, because their item data can hold a PIN; DEV keeps saving (test PINs only). Failures still reach WF-90 → `error_log` |
| Credentials (names) | `Telegram Bot (DEV)` / `Telegram Bot (PROD)`, `Supabase Service Role`, `OpenAI`, `Zoho Books OAuth (DEV/PROD)`, `Admin Alert Bot` |
| Database access (CR-021) | Only through PostgREST with the `service_role` key (`Supabase Service Role`, a Supabase API credential): the Supabase node for tables, HTTP Request to `/rest/v1/rpc/<function>` for functions. **No Postgres-node credential with the owner login (`postgres`)**: the owner ignores RLS and the function revokes. Multi-step atomicity lives inside database functions, because each call is its own transaction |
| Environment variables | `ENV` (`dev`/`prod`), `TENANT_ID` (Phase 1), `DB_ENCRYPTION_KEY`, `TELEGRAM_WEBHOOK_SECRET`, `ADMIN_CHAT_ID`, `OPENAI_INTENT_MODEL`, `OPENAI_STT_MODEL`, `OPENAI_VISION_MODEL`, `PROMPT_VERSION` |
| Sub-workflows | Called with **Execute Workflow**; input via **Execute Workflow Trigger** with **typed inputs** ("Define using JSON example", replace the template's example with the workflow's own input contract from §6; CR-006); always return the **standard result** (§5.3). The `Validate input` node still enforces business rules (required values, formats) |
| Layer rule | Domain workflows (WF-10…WF-20) never call Telegram/WhatsApp. Channel workflows (WF-00/01) never call Zoho |
| Zoho calls | **Only through WF-92 Zoho Client** (never a raw HTTP node to Zoho in other workflows) |
| Writes | Only after `claim_pending_action()` returns a row (see `Database.md` §5.2) |
| Secrets | Never in Set/Code nodes or sticky notes; only credentials/env |
| Code nodes | Small and pure; validation logic lives in one Code node per workflow named `Validate input` |
| Sticky note | Every workflow starts with a sticky note: ID, purpose, input, output, version |

## 3. Workflow register

| ID | Name | Layer | Trigger | Called by | Calls | Milestone | Version | Status |
|---|---|---|---|---|---|---|---|---|
| WF-00 | Telegram Inbound | channel | Telegram webhook | Telegram | WF-01, WF-02 | M2 | — | Spec |
| WF-01 | Channel Adapter | channel | Execute Workflow | WF-00, all replies | Telegram API | M2 | — | Spec |
| WF-02 | Auth & Session | core | Execute Workflow | WF-00 | Supabase | M2 | — | Spec |
| WF-03 | Media Normaliser | core | Execute Workflow | WF-00 | Telegram file API, OpenAI | M3 | — | Spec |
| WF-04 | Intent Parser | core | Execute Workflow | WF-00 | OpenAI | M3 | — | Spec |
| WF-05 | Entity Resolver | core | Execute Workflow | WF-00 | WF-92, Supabase | M3 | — | Spec |
| WF-06 | Draft & Confirm | core | Execute Workflow | WF-00 | Supabase, WF-07, WF-01 | M3 | — | Spec |
| WF-07 | Router | core | Execute Workflow | WF-00, WF-06 | WF-10…WF-40 | M3 | — | Spec |
| WF-08 | PIN Handler | core | Execute Workflow | WF-00 | Supabase, WF-01 | M2 | — | Spec |
| WF-10 | Customers | domain | Execute Workflow | WF-07 | WF-92 | M4 | — | Spec |
| WF-11 | Items | domain | Execute Workflow | WF-07 | WF-92 | M4 | — | Spec |
| WF-12 | Quotations | domain | Execute Workflow | WF-07 | WF-92 | M5 | — | Spec |
| WF-13 | Invoices | domain | Execute Workflow | WF-07 | WF-92 | M5 | — | Spec |
| WF-14 | Payments | domain | Execute Workflow | WF-07 | WF-92 | M6 | — | Spec |
| WF-15 | Expenses | domain | Execute Workflow | WF-07 | WF-92 | M7 | — | Spec |
| WF-20 | Reports | domain | Execute Workflow | WF-07, WF-30 | WF-92 | M8 | — | Spec |
| WF-30 | Scheduled Reports | ops | Schedule (every 15 min) | — | WF-20, WF-01 | M11 | — | Spec |
| WF-40 | User Admin | core | Execute Workflow | WF-07 | Supabase | M2 | — | Spec |
| WF-90 | Error Handler | ops | Error Trigger | n8n (all) | Supabase, WF-01 | M2 | — | Spec |
| WF-91 | Zoho Auth | zoho | Execute Workflow | WF-92 | Zoho Accounts, Supabase | M1 | — | Spec |
| WF-92 | Zoho Client | zoho | Execute Workflow | WF-05, WF-10…WF-20 | WF-91, Zoho Books API | M1 | — | Spec |
| WF-93 | Housekeeping | ops | Schedule (daily 03:00 GST) | — | Supabase, WF-01 | M2 | — | Spec |
| WF-94 | Draft Expiry | ops | Schedule (every 5 min) | — | Supabase, WF-01 | M3 | — | Spec |
| WF-00b | WhatsApp Inbound | channel | WhatsApp webhook | Meta | WF-01, WF-02 | M12 | — | Planned |

**Status values:** Planned → Spec → Spec approved → Built (DEV) → Tested → Live (PROD) → Deprecated

## 4. Starter template (every sub-workflow begins from this)

Import into n8n (*Workflows → Import from file/clipboard*), rename, replace the trigger's `jsonExample` with the workflow's input contract, then build between `Validate input` and `Return result`. The exact n8n version is recorded at M1 start; typed inputs need Execute Workflow Trigger typeVersion ≥ 1.1 (CR-006). Node type versions may differ on your n8n version. If n8n upgrades a node on import, keep the upgraded version and note it in the change history.

```json
{
  "name": "WF-xx <Name>",
  "nodes": [
    {
      "parameters": {
        "content": "## WF-xx <Name>\n**Purpose:** <one line>\n**Input:** see Workflows.md §6\n**Output:** standard result (§5.3)\n**Version:** 0.1",
        "height": 220,
        "width": 380
      },
      "id": "a1b2c3d4-0000-4000-8000-000000000001",
      "name": "Sticky Note",
      "type": "n8n-nodes-base.stickyNote",
      "typeVersion": 1,
      "position": [-120, -260]
    },
    {
      "parameters": {
        "inputSource": "jsonExample",
        "jsonExample": "{\n  \"tenant_id\": \"uuid\",\n  \"user_id\": \"uuid\",\n  \"intent\": \"domain.action\",\n  \"payload\": null\n}"
      },
      "id": "a1b2c3d4-0000-4000-8000-000000000002",
      "name": "When called by another workflow",
      "type": "n8n-nodes-base.executeWorkflowTrigger",
      "typeVersion": 1.2,
      "position": [0, 0]
    },
    {
      "parameters": {
        "jsCode": "// Validate the input contract for this workflow.\nconst input = $input.first().json;\nconst required = ['tenant_id', 'user_id', 'intent'];\nconst missing = required.filter(k => input[k] === undefined || input[k] === null || input[k] === '');\nif (missing.length) {\n  return [{ json: { ok: false, data: null, error: { code: 'INVALID_INPUT', message_key: 'error.invalid_input', details: { missing } } } }];\n}\nreturn [{ json: { ...input, _valid: true } }];"
      },
      "id": "a1b2c3d4-0000-4000-8000-000000000003",
      "name": "Validate input",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [220, 0]
    },
    {
      "parameters": {
        "conditions": {
          "options": { "caseSensitive": true, "leftValue": "", "typeValidation": "loose" },
          "conditions": [
            {
              "id": "c0000000-0000-4000-8000-000000000001",
              "leftValue": "={{ $json._valid }}",
              "rightValue": true,
              "operator": { "type": "boolean", "operation": "true", "singleValue": true }
            }
          ],
          "combinator": "and"
        },
        "options": {}
      },
      "id": "a1b2c3d4-0000-4000-8000-000000000004",
      "name": "Is valid?",
      "type": "n8n-nodes-base.if",
      "typeVersion": 2,
      "position": [440, 0]
    },
    {
      "parameters": {
        "jsCode": "// TODO (Implementer): replace with the workflow's real steps.\nconst input = $input.first().json;\nreturn [{ json: { ok: true, data: { echo: input.intent }, error: null } }];"
      },
      "id": "a1b2c3d4-0000-4000-8000-000000000005",
      "name": "Do work",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [660, -100]
    },
    {
      "parameters": {
        "jsCode": "// Standard result: { ok, data, error }\nconst r = $input.first().json;\nreturn [{ json: { ok: !!r.ok, data: r.data ?? null, error: r.error ?? null } }];"
      },
      "id": "a1b2c3d4-0000-4000-8000-000000000006",
      "name": "Return result",
      "type": "n8n-nodes-base.code",
      "typeVersion": 2,
      "position": [880, 0]
    }
  ],
  "connections": {
    "When called by another workflow": { "main": [[{ "node": "Validate input", "type": "main", "index": 0 }]] },
    "Validate input": { "main": [[{ "node": "Is valid?", "type": "main", "index": 0 }]] },
    "Is valid?": {
      "main": [
        [{ "node": "Do work", "type": "main", "index": 0 }],
        [{ "node": "Return result", "type": "main", "index": 0 }]
      ]
    },
    "Do work": { "main": [[{ "node": "Return result", "type": "main", "index": 0 }]] }
  },
  "settings": { "executionOrder": "v1", "saveDataErrorExecution": "all", "saveDataSuccessExecution": "none" },
  "tags": [],
  "pinData": {}
}
```

> After import: set **Settings → Error workflow = WF-90 Error Handler**, and add the tags `zb-assistant` + the layer tag.

## 5. Shared contracts

### 5.1 Normalised message (output of WF-01 inbound)
See `Architecture.md` §5. All core workflows receive this object as `message`.

### 5.2 Context object (passed between core workflows)

```json
{
  "tenant_id": "uuid",
  "user": { "id": "uuid", "role": "owner", "language": "en", "display_name": "Ahmed" },
  "message": { "...": "normalised message" },
  "session": { "state": { "awaiting": null }, "last_intent": null, "version": 0 },
  "intent": "invoice.create",
  "fields": { "customer_name": "Al Noor Trading", "line_items": [ { "item_name": "Consulting", "quantity": 3, "rate": 500 } ] },
  "resolved": { "customer_id": "4600000000012345", "currency_code": "AED" },
  "permission": { "allowed": true, "needs_pin": false },
  "prompt_version": "intent-v1"
}
```

**`session.state.awaiting` values (CR-010 rev 2):** `null` · `'field'` (+ `field` name) · `'choice'` (+ `options`) · `'pin'` (+ `pending_action_id` or the waiting read request) · `'pin_setup'` · `'pin_setup_confirm'`. Also `last_pin_prompt_at` (set by WF-06 when it asks for a PIN; cleared by WF-08 on success and by Cancel).

**Session writes (CR-018):** WF-02 calls `touch_session()` once per message (no version bump) and passes `session.version` on. Only WF-06, WF-08, and WF-02 (PIN gate only: `awaiting = 'pin_setup'`; CR-021) write state, through `save_session(user, version, state, last_intent)`. A null result means a conflict → reply `session.busy` naming only the message type ("I was still working on your previous voice note/message. Please send this one again."). Read-only turns write no state.

### 5.3 Standard result (returned by every sub-workflow)

```json
{
  "ok": true,
  "data": { "zoho_entity": "invoice", "zoho_record_id": "4600000000098765", "zoho_record_number": "INV-000123", "pdf_binary_key": "data" },
  "error": null,
  "reply": { "message_key": "invoice.created", "params": { "number": "INV-000123", "total": "AED 3,675.00" } }
}
```

On failure:

```json
{ "ok": false, "data": null, "error": { "code": "ZOHO_VALIDATION", "message_key": "error.zoho_validation", "details": { "zoho_code": 1001 } } }
```

### 5.4 Error codes

| Code | Meaning | User message key |
|---|---|---|
| `INVALID_INPUT` | Contract not met | `error.invalid_input` |
| `NOT_AUTHORISED` | Unknown sender | `auth.not_authorised` |
| `PERMISSION_DENIED` | Role not allowed | `auth.permission_denied` |
| `PIN_REQUIRED` / `PIN_WRONG` / `PIN_LOCKED` | PIN flow | `pin.*` |
| `PIN_NOT_SET` | User has no PIN yet; only PIN setup is allowed (CR-012) | `pin.setup_required` |
| `DRAFT_EXPIRED` | Draft past `expires_at` | `draft.expired` |
| `ALREADY_EXECUTED` | Double confirm | `draft.already_done` |
| `AMBIGUOUS_ENTITY` | Multiple matches | `resolve.choose` |
| `NOT_FOUND` | No match in Zoho | `resolve.not_found` |
| `ZOHO_VALIDATION` | Zoho rejected the data | `error.zoho_validation` |
| `ZOHO_RATE_LIMIT` | 429 after retries | `error.busy_try_later` |
| `ZOHO_AUTH` | Token refresh failed | `error.service_unavailable` (+ admin alert) |
| `AI_UNCLEAR` | Low-confidence intent | `ai.clarify` |
| `SESSION_BUSY` | Session state changed by a parallel message (CR-018) | `session.busy` |
| `INTERNAL` | Anything else | `error.generic` (+ admin alert) |

---

## 6. Workflow specifications

Each spec covers: **Purpose · Trigger · Input · Steps · Output · Errors · Tests · Exported JSON · Change history**.

### WF-00 Telegram Inbound
- **Purpose:** Entry point for every Telegram update.
- **Trigger:** Webhook (POST `/webhook/zb/telegram/<env>`). Telegram is registered with `secret_token = TELEGRAM_WEBHOOK_SECRET`.
- **Steps:**
  1. Check header `X-Telegram-Bot-Api-Secret-Token`. If wrong, return 401 and stop (log to `error_log`).
  2. Respond 200 immediately.
  3. Set `tenant_id` from env `TENANT_ID` (Phase 1; CR-016) → `register_inbound(tenant, 'telegram', update_id)`. If false (a redelivery) → stop silently: no log, no reply, no draft (CR-013). Source: Telegram `setWebhook` docs (repeats on unsuccessful requests; `update_id` identifies repeats).
  4. WF-01 (inbound) → normalised message.
  5. WF-02 → user, session (stop if not authorised; if no PIN set, only PIN setup, CR-012).
  6. If the session awaits a PIN (`awaiting` = `pin` / `pin_setup` / `pin_setup_confirm`), whatever the message type, or the message is a late PIN (rule below) → **WF-08 PIN Handler** and stop this path (CR-010, CR-021). A PIN never reaches WF-03/04/06 or OpenAI.
  7. If button press → WF-06; else if voice/photo → WF-03 → WF-04; else → WF-04.
  8. WF-05 (if entities to resolve) → WF-06 (draft/preview) or WF-07 (reads).
  9. WF-01 (outbound) to send the reply.
  - **Late PIN (CR-010 rev 2):** a message is treated as a PIN typed late **only if** the text matches `^[0-9]{4,6}$` AND `awaiting` is null AND `last_pin_prompt_at` is within the last `draft_ttl_minutes`. Then WF-02 logs `[REDACTED]` → WF-08 in mode `late_pin`. Every other number (field answers, choices, no recent PIN prompt) is normal text. Accepted residual risk: a PIN typed with no PIN prompt in the last 30 min is handled as normal text.
- **Output:** none (reply sent through WF-01).
- **Errors:** any failure → WF-90.
- **Tests:** wrong secret rejected; unknown user rejected; text, voice, photo, and button each routed correctly; the same update posted twice → 1 `message_log` row, 1 reply, 1 draft (CR-013); a PIN reply is routed to WF-08 (CR-010).
- **Exported JSON:** `Pending — M2`

### WF-01 Channel Adapter
- **Purpose:** Translate between Telegram (later WhatsApp) and the normalised format, in both directions.
- **Input (inbound):** raw Telegram update. **Input (outbound):** `{ channel, chat_id, reply: { text | message_key+params, buttons[], document{binary,name}, edit_message_id } }`.
- **Steps (outbound):** render `message_key` in the user's language (EN/AR/UR/Roman Urdu text table); build the inline keyboard; send the message or document; return `channel_message_id`.
- **Rules:** Telegram message ≤ 4096 chars (split if longer); Arabic/Urdu sent as plain text (no Markdown parse mode issues); buttons carry `action:pending_action_id`.
- **Tests:** each language renders; long text splits; PDF sends with the correct file name (`INV-000123.pdf`).
- **Exported JSON:** `Pending — M2`

### WF-02 Auth & Session
- **Purpose:** Allow-list, status, lockout, and session load/save.
- **Input:** normalised message. **Output:** `{ ok, user, session }` or `NOT_AUTHORISED`.
- **Steps:** look up `users` by `(channel, channel_user_id, status='active')` → if none, log and reply `auth.not_authorised`, which includes the sender's own Telegram ID so the Owner can add them (CR-012) → `touch_session()` (creates or extends the row; CR-018) → update `last_seen_at` → write `message_log` (inbound). If `sessions.state.awaiting` is a PIN state, or the message is a late PIN (see WF-00), write `text = '[REDACTED]'` (CR-010).
- **PIN gate (CR-012):** if `pin_hash` is null, the user can only do PIN setup. Every other message gets `pin.setup_required`, and the session is set to `awaiting = 'pin_setup'`.
- **Session writes:** `touch_session()` once per message (a null result → `INTERNAL`). The only state write here is the PIN gate (`awaiting = 'pin_setup'`) through `save_session()` (CR-021). If `users.chat_id` is null, fill it from this message (after an account move, CR-022).
- **Tests:** unknown user (reply shows their ID); disabled user; locked user still can read but PIN actions are blocked; a user without a PIN can't create anything; a PIN reply is `[REDACTED]` in `message_log`; an unknown-sender row carries the tenant.
- **Exported JSON:** `Pending — M2`

### WF-03 Media Normaliser
- **Purpose:** Turn voice into text and receipt images/PDFs into structured fields.
- **Steps:** download the file via the Telegram `getFile` API → voice: OpenAI speech-to-text (`OPENAI_STT_MODEL`) → text + detected language. Photo/document: OpenAI vision (`OPENAI_VISION_MODEL`) with the receipt-extraction prompt → `{ vendor, date, total, tax, currency, trn, confidence }` → log `ai_usage`.
- **Rules:** files are not stored in our DB (only `media_ref`); the image is attached to the Zoho expense later by WF-15.
- **Tests:** 20-receipt set ≥ 85% field accuracy; voice in 4 languages.
- **Exported JSON:** `Pending — M3`

### WF-04 Intent Parser
- **Purpose:** Turn text into `{ intent, fields, language, confidence, missing_fields }`.
- **Steps:** build a prompt from the system prompt (`PROMPT_VERSION`) + intent catalogue (`intents` table) + JSON schema + short session context → OpenAI structured output → validate against the schema → if `confidence < 0.7` → `AI_UNCLEAR` (ask the user) → permission check via `check_permission()` → log `ai_usage`.
- **Rules:** the LLM never calls Zoho; numbers it returns are hints only and are recomputed later; text inside messages or images is treated as data, never as instructions.
- **Tests:** test set ≥ 90% accuracy (all 4 languages); prompt-injection samples are ignored.
- **Exported JSON:** `Pending — M3`

### WF-05 Entity Resolver
- **Purpose:** Turn names into Zoho IDs (customer, item, invoice, quote, tax, currency).
- **Steps:** search `entity_cache` (trigram, via an RPC function: PostgREST filters can't do similarity search under CR-021; e.g. `search_entity_cache(p_tenant, p_entity, p_text, p_limit)` with `similarity()`/`%` and `set search_path = public, extensions`; **to be specified in the M3 design note**, CR-023) → if stale or no hit, search Zoho through WF-92 → 0 matches: `NOT_FOUND` (offer to create); 1 match: resolved; 2–5 matches: `AMBIGUOUS_ENTITY` with buttons; more than 5: ask for more detail.
- **Tests:** exact, fuzzy, Arabic-name, and ambiguous cases.
- **Exported JSON:** `Pending — M3`

### WF-06 Draft & Confirm
- **Purpose:** Build the preview, store the draft, and handle Confirm / Edit / Cancel / PIN.
- **Steps (new write):** ask for missing required fields one at a time (state in `sessions`) → build the Zoho body + recompute totals and VAT → insert `pending_actions` (unique `idempotency_key`, `expires_at = now + draft_ttl`) → send the preview with buttons.
- **Steps (Confirm):** if `needs_pin` and not verified → `save_session()` with `awaiting = 'pin'`, the `pending_action_id`, and `last_pin_prompt_at = now()` → ask for the PIN (the reply goes to WF-08, which calls `verify_pin_for_action()`; CR-010/011) → on `{ok:true}` from WF-08 → `claim_pending_action()` → if no row → `ALREADY_EXECUTED`/`DRAFT_EXPIRED` → WF-07 → `complete_pending_action()` → `audit_log`. WF-06 never receives the PIN itself.
- **Steps (Edit):** take the change by text/voice → rebuild → new preview (old draft cancelled). **Cancel:** status `cancelled`; clear `awaiting` and `last_pin_prompt_at`.
- **State writes:** all through `save_session()`; on conflict reply `session.busy` (CR-018).
- **Tests:** double Confirm → one record; expired draft; PIN wrong ×5 → lockout + exactly one Owner alert; correct PIN → `pin_verified_at` set → claim succeeds once; PIN for someone else's draft → `action_invalid`; no PIN set → `pin_not_set`, counter unchanged.
- **Exported JSON:** `Pending — M3`

### WF-07 Router
- **Purpose:** Send a validated intent and payload to the correct domain workflow; re-check permission just before execution.
- **Steps:** switch on `intent` domain → Execute Workflow (WF-10…WF-40) → return the standard result.
- **Exported JSON:** `Pending — M3`

### WF-08 PIN Handler *(new, CR-010)*
- **Purpose:** The only workflow that receives a PIN. It keeps PINs away from WF-04/WF-06 and OpenAI.
- **Trigger:** Execute Workflow, from WF-00 when `sessions.state.awaiting` is `pin`, `pin_setup`, or `pin_setup_confirm`, or in mode `late_pin` (CR-010 rev 2).
- **Input checks before any PIN function (CR-021):**
  1. Non-text (voice, photo, document) → delete it, never transcribe or send to vision; reply `pin.type_it` ("Please type your PIN").
  2. A Cancel button press or the cancel word, per state (CR-023):
     - `pin` + `pending_action_id` → draft `cancelled`; clear `awaiting` and `last_pin_prompt_at`;
     - `pin` + a waiting read request → drop the request (no draft); clear `awaiting` and `last_pin_prompt_at`;
     - `pin_setup` / `pin_setup_confirm` → setup can't be cancelled (CR-012 gate); reply `pin.setup_required`.
  3. Text not matching `^[0-9]{4,6}$` → **no verification, no count**; reply `pin.format` ("Send your 4–6 digit PIN or tap Cancel"). The same check applies in `pin_setup`.
  4. Every PIN prompt carries a **Cancel** button.
- **Steps:** delete the PIN message (Telegram `deleteMessage`; bots can delete incoming messages in private chats if under 48 hours old; on failure, log and continue, no retry; source: Telegram Bot API `deleteMessage`) → by state:
  - `pin` + `pending_action_id` → `verify_pin_for_action()` (CR-011); reason `expired` / `action_invalid` → clear `awaiting` and `last_pin_prompt_at`, reply `draft.expired` (CR-021);
  - `pin` + a waiting PIN-protected read request → `verify_user_pin()`;
  - `pin_setup` → `pin_setup_first()` → ask to repeat → `awaiting = 'pin_setup_confirm'`;
  - `pin_setup_confirm` → `pin_setup_confirm()` → `pin.set_ok` or restart on `mismatch`/`restart` (CR-012);
  - mode `late_pin` → no verification; reply `pin.not_awaited` ("No action is waiting for a PIN. Your draft may have expired; please send the request again."); clear `last_pin_prompt_at` (CR-010 rev 2).
  On success, clear `awaiting` and `last_pin_prompt_at`. State writes through `save_session()`; the PIN functions' DB results are already committed, so on `session.busy` pressing Confirm again works (CR-018).
  If `just_locked = true` → one Owner alert (FR-1.3). Return only the result (`{pin_ok, locked, reason, …}`), never the PIN.
- **Settings:** PROD saves no execution data (§2 exception).
- **Tests:** PIN deleted in chat; correct, wrong, locked, and not-set paths; "cancel" or a new request in a PIN state is not counted as a wrong PIN; a voice note in a PIN state is deleted and not transcribed; a PIN after WF-94 expired the draft → `draft.expired` and the state is cleared; 2-step setup and mismatch restart; with PROD settings and a forced failure, the test PIN appears 0 times in the n8n execution DB.
- **Exported JSON:** `Pending — M2`

### WF-10 Customers
- **Intents:** `customer.create`, `customer.update`, `customer.get`, `customer.search`, `customer.balance`.
- **Zoho:** `POST/PUT/GET /contacts`, `GET /contacts?search_text=`. UAE fields: tax treatment, TRN, place of supply (emirate), currency.
- **After write:** refresh `entity_cache` for that contact.
- **Exported JSON:** `Pending — M4`

### WF-11 Items
- **Intents:** `item.create`, `item.update`, `item.get`, `item.search`.
- **Zoho:** `/items` (service type, rate, tax_id default VAT 5%).
- **Exported JSON:** `Pending — M4`

### WF-12 Quotations
- **Intents:** `quote.create`, `quote.update`, `quote.get`, `quote.list`, `quote.mark_accepted`, `quote.mark_declined`, `quote.convert_to_invoice`, `quote.pdf`.
- **Zoho:** `/estimates`, status endpoints, PDF via `accept=pdf`; conversion creates an invoice from the estimate.
- **Rules:** update only in an editable status; conversion goes through its own preview + Confirm.
- **Exported JSON:** `Pending — M5`

### WF-13 Invoices
- **Intents:** `invoice.create`, `invoice.update`, `invoice.get`, `invoice.list`, `invoice.void`, `invoice.pdf`.
- **Zoho:** `/invoices`, `/invoices/{id}/status/void`, PDF via `accept=pdf`.
- **Rules:** void = Owner + PIN; list filters: unpaid, overdue, paid, customer, date range.
- **Exported JSON:** `Pending — M5`

### WF-14 Payments
- **Intents:** `payment.create`, `payment.get`, `payment.receipt_pdf`.
- **Zoho:** `/customerpayments` (apply to one or more invoices; amount, date, mode, reference).
- **Rules:** PIN required; show the remaining balance after saving.
- **Exported JSON:** `Pending — M6`

### WF-15 Expenses
- **Intents:** `expense.create` (from text/voice or receipt).
- **Zoho:** `POST /expenses`, then attach the receipt file to the expense.
- **Rules:** the expense account is chosen from a mapped list (no free-text account names).
- **Exported JSON:** `Pending — M7`

### WF-20 Reports
- **Intents:** `report.run` (`profit_and_loss`, `receivables_aging`, `sales_by_customer`, `expense_summary`).
- **Steps:** call the Zoho report endpoint through WF-92 (exact endpoints confirmed in M1, Arch T2) → render PDF/XLSX (if Zoho export isn't available on the plan, generate from JSON) → return the binary.
- **Rules:** Owner + PIN unless the permission matrix allows Staff.
- **Exported JSON:** `Pending — M8`

### WF-30 Scheduled Reports *(Should — M11)*
- **Trigger:** Schedule, every 15 minutes → select due `scheduled_reports` → WF-20 → WF-01 → set `next_run_at`.
- **Exported JSON:** `Pending — M11`

### WF-40 User Admin
- **Intents:** `user.add`, `user.remove`, `user.set_role`, `user.reset_pin`, `audit.summary`.
- **Rules:** Owner + PIN; the last active owner cannot be removed or demoted.
- **user.add (CR-012):** the Owner sends the new user's Telegram ID (shown to them in the unknown-sender reply), name, and role → the user is created `active` with `pin_hash` null → they set their own PIN on their first message (WF-02 gate → WF-08). Offering recent unknown senders as buttons: not in Phase 1.
- **user.remove:** sets the user `disabled` and cancels their open drafts (`pending`/`confirmed`) (CR-022). `audit.summary` can list disabled users, so history split across an old and a new account stays visible.
- **Moving a Staff account** (or an Owner when another active Owner exists): `user.remove` the old account + `user.add` the new ID, in chat. A **sole Owner** who lost their account: operator Procedure B, `move_user_channel()` (CR-022).
- **user.reset_pin (CR-012):** clears `pin_hash` and `pin_pending_hash`; the user sets a new PIN on their next message. The Owner never knows anyone's PIN. Recovery when the only Owner forgets their own PIN: operator Procedure A with `recover_user_pin()`, not available in chat (CR-019). Runbook: `implementation/runbooks/account-recovery.md`.
- **Tests:** a new user sets a PIN in 2 steps; after a reset the user must set a new PIN.
- **Exported JSON:** `Pending — M2`

### WF-90 Error Handler
- **Trigger:** Error Trigger (set as the error workflow on every workflow).
- **Steps:** **redact** (CR-008: patterns `Zoho-oauthtoken …` and `1000\.[0-9a-f]+\.[0-9a-f]+`; truncate long messages; never log item data, CR-010) → insert `error_log` with `tenant_id` falling back to env `TENANT_ID` (CR-016) → if a user is known, send `error.generic` in their language → alert `ADMIN_CHAT_ID` with workflow, node, execution ID (no personal data).
- **Exported JSON:** `Pending — M2`

### WF-91 Zoho Auth
- **Steps:** `zoho_get_credentials(tenant, DB_ENCRYPTION_KEY)` → if the access token is valid for more than 5 minutes, return it → else `zoho_claim_refresh(tenant)` (CR-002):
  - **true (winner):** POST `{accounts_domain}/oauth/v2/token` (refresh_token grant) → `zoho_save_tokens()` (also releases the claim) → return the token.
  - **false (another execution is refreshing):** wait ~1 s → re-read credentials → return the token once it is valid; max 5 tries. Losers never refresh and never alert. If `zoho_get_credentials()` returns no row (connection not active), stop at once. After 5 tries: `{ ok:false, error:{ code:"ZOHO_AUTH", message_key:"error.service_unavailable", details:{ reason:"refresh_wait_timeout" } } }` (CR-002 rev 2).
  - **Winner's token call:** timeout 10 s, no node-level retry (always ends inside the 30 s claim).
  - **Winner's failure branch:** `zoho_refresh_failed(tenant, permanent, sanitised_error)`: permanent = Zoho `invalid_code` (refresh token invalid or revoked; source: Zoho Accounts OAuth docs, access-token-expiry page), otherwise transient. The function returns `{alert, status}` (CR-002 rev 3): send the admin alert **only when `alert = true`** (permanent = "reconnect Zoho"; transient = first failure of an incident, then at most one reminder every 10 min). If it returns null (no connection row), treat it as `INTERNAL` and alert.
- **Tests:** T3: 10 parallel calls with an expiring token cause exactly **one** refresh. T3b: forced invalid refresh token → status `error`, one alert, callers fail in under 1 s. T3c: forced timeout → 5 s cool-down, then one successful refresh. T3d: (a) 3 transient failures within 10 min → exactly 1 alert; (b) after a successful refresh, a new failure alerts at once; (c) still failing 10+ min later → 1 reminder.
- **Exported JSON:** `Pending — M1`

### WF-92 Zoho Client
- **Input:** `{ tenant_id, method, path, query, body, accept }`.
- **Steps:** WF-91 token → HTTP request to `{api_domain}/books/v3{path}?organization_id=…` → `increment_zoho_usage()` → on 429/5xx retry with backoff 2s/4s/8s (max 3) → on 401 refresh the token once and retry → map Zoho errors to §5.4 codes → alert when daily calls exceed 80% of `zoho_daily_call_limit`.
- **Token protection (CR-008):**
  1. Saves no execution data (see §2 exception).
  2. The HTTP node uses `onError = continueErrorOutput`. Every failure (401/429/5xx/validation/timeout) is mapped to the §5.4 standard result inside WF-92, so a run never fails with the token in context.
  3. One sanitised `error_log` row per failed call: method, path (no query values), HTTP status, Zoho code/message, retry count, execution ID. Never headers or request body. WF-92 sends the admin alert itself for `ZOHO_AUTH`/`INTERNAL` (handled errors don't trigger WF-90).
  4. The return node outputs only `{ok, data, error}`. The token never leaves WF-92, so callers (which keep "error = all") never hold it.
- **Tests:** T5 evidence = the `error_log` rows. T7: after forced 401/429/5xx/validation runs, search the n8n DB (execution data), `error_log`, and Postgres logs for `Zoho-oauthtoken` and the Zoho token pattern → 0 hits. Also confirm WF-90 still fires for an unhandled WF-92 error while saving is off.
- **Exported JSON:** `Pending — M1`

### WF-93 Housekeeping
- **Trigger:** Schedule, daily at 03:00 Asia/Dubai.
- **Steps:** `run_housekeeping()` (expires drafts as a safety net, deletes or redacts old drafts, purges logs and `inbound_updates`) → read `v_daily_health` and `v_tenants_without_owner` → send the daily health summary to the admin chat.

### WF-94 Draft Expiry *(new, CR-014)*
- **Purpose:** Tell the user when a draft expires (FR-3.3) and remove stale buttons.
- **Trigger:** Schedule, every 5 minutes. The notice arrives within 30–35 minutes of the preview (accepted tolerance for FR-3.3).
- **Steps:** `expire_drafts()` → group rows by user → for each draft, WF-01 edits the preview (`edit_message_id = channel_message_id`) to remove its buttons → one message per user, `draft.expired_notice` ("Your draft <intent label> expired. Send it again if you still need it."), combining several drafts. The Confirm-time `DRAFT_EXPIRED` reply stays.
- **Tests:** a draft left 30 min → within 5 min the buttons are gone and the notice arrives; tapping an old button → `draft.expired`; 2 drafts expiring together → 1 message.
- **Exported JSON:** `Pending — M3`
- **Exported JSON:** `Pending — M2`

### WF-00b WhatsApp Inbound *(Should — M12)*
- **Purpose:** Same as WF-00 for the WhatsApp Cloud API (verify the `X-Hub-Signature-256` header; webhook verification challenge).
- **Note:** messages outside the 24-hour window must use approved templates from `Templates.md`.
- **Exported JSON:** `Pending — M12`

---

## 7. Exported JSON store

Paste each exported workflow below once it reaches **Tested** status. Keep one block per workflow and replace it on every new version.

### WF-00 Telegram Inbound — v—
```json
{ "status": "Pending — M2" }
```

### WF-01 Channel Adapter — v—
```json
{ "status": "Pending — M2" }
```

### WF-02 Auth & Session — v—
```json
{ "status": "Pending — M2" }
```

### WF-03 Media Normaliser — v—
```json
{ "status": "Pending — M3" }
```

### WF-04 Intent Parser — v—
```json
{ "status": "Pending — M3" }
```

### WF-05 Entity Resolver — v—
```json
{ "status": "Pending — M3" }
```

### WF-06 Draft & Confirm — v—
```json
{ "status": "Pending — M3" }
```

### WF-07 Router — v—
```json
{ "status": "Pending — M3" }
```

### WF-10 Customers — v—
```json
{ "status": "Pending — M4" }
```

### WF-11 Items — v—
```json
{ "status": "Pending — M4" }
```

### WF-12 Quotations — v—
```json
{ "status": "Pending — M5" }
```

### WF-13 Invoices — v—
```json
{ "status": "Pending — M5" }
```

### WF-14 Payments — v—
```json
{ "status": "Pending — M6" }
```

### WF-15 Expenses — v—
```json
{ "status": "Pending — M7" }
```

### WF-20 Reports — v—
```json
{ "status": "Pending — M8" }
```

### WF-30 Scheduled Reports — v—
```json
{ "status": "Pending — M11" }
```

### WF-08 PIN Handler — v—
```json
{ "status": "Pending — M2" }
```

### WF-40 User Admin — v—
```json
{ "status": "Pending — M2" }
```

### WF-90 Error Handler — v—
```json
{ "status": "Pending — M2" }
```

### WF-91 Zoho Auth — v—
```json
{ "status": "Pending — M1" }
```

### WF-92 Zoho Client — v—
```json
{ "status": "Pending — M1" }
```

### WF-93 Housekeeping — v—
```json
{ "status": "Pending — M2" }
```

### WF-94 Draft Expiry — v—
```json
{ "status": "Pending — M3" }
```

### WF-00b WhatsApp Inbound — v—
```json
{ "status": "Pending — M12" }
```

## 8. Change history

| Date | Workflow | Version | Change | CR / GAP | By |
|---|---|---|---|---|---|
| 2026-10-04 | All | 0.1 | Register, conventions, specs, starter template | — | Architect |
| 2026-10-07 | WF-91 | 0.2 (spec) | Single-flight token refresh | CR-002 / GAP-005 | Architect |
| 2026-10-07 | Template, §2 | 0.2 (spec) | Typed inputs on Execute Workflow Trigger (jsonExample, typeVersion 1.2) | CR-006 / GAP-013 | Architect |
| 2026-10-07 | WF-91 | 0.3 (spec) | Loser timeout result, winner failure branch, `zoho_refresh_failed()`, tests T3b/T3c | CR-002 rev 2 / GAP-005 | Architect |
| 2026-10-07 | WF-91 | 0.4 (spec) | Alert only when `zoho_refresh_failed()` returns `alert = true`; null = INTERNAL; test T3d | CR-002 rev 3 / GAP-005 C3 | Architect |
| 2026-10-07 | §2, WF-00/02/06/90, new WF-08 | 0.5 (spec) | PIN isolation, redaction, PROD no-save exception | CR-010 / GAP-001 | Architect |
| 2026-10-07 | WF-06, WF-08 | 0.5 (spec) | `verify_pin_for_action()` | CR-011 / GAP-002 | Architect |
| 2026-10-07 | WF-02, WF-40, WF-08 | 0.5 (spec) | PIN setup flow, user.add by ID, reset clears PIN | CR-012 / GAP-003 | Architect |
| 2026-10-07 | WF-00 | 0.5 (spec) | Inbound de-duplication | CR-013 / GAP-004 | Architect |
| 2026-10-07 | new WF-94 | 0.5 (spec) | Draft expiry notice | CR-014 / GAP-007 | Architect |
| 2026-10-07 | WF-00, WF-90 | 0.5 (spec) | Tenant from `TENANT_ID` | CR-016 / GAP-009 | Architect |
| 2026-10-07 | §5.2, WF-00/02/06/08 | 0.6 (spec) | `awaiting` values; narrow late-PIN rule | CR-010 rev 2 / GAP-001 C4 | Architect |
| 2026-10-07 | §5.2, §5.4, WF-02/06/08 | 0.6 (spec) | TOUCH vs STATE session writes; `SESSION_BUSY` | CR-018 / GAP-006 C6 | Architect |
| 2026-10-07 | WF-40 | 0.6 (spec) | Sole-Owner PIN recovery via operator runbook | CR-019 / GAP-017 | Architect |
| 2026-10-07 | §2, WF-00/02/08 | 0.7 (spec) | DB access via PostgREST only; WF-08 input checks + Cancel; WF-02 PIN-gate write; layout fix (WF-08 moved after WF-07) | CR-021 / GAP-020 | Architect |
| 2026-10-07 | WF-02, WF-40 | 0.7 (spec) | Account move; fill chat_id; remove cancels drafts | CR-022 / GAP-019 | Architect |
| 2026-10-07 | WF-05, WF-08 | 0.8 (spec) | Cancel per PIN state; WF-05 trigram search via RPC (M3 note) | CR-023 / GAP-021 | Architect |
| 2026-10-07 | §2, WF-90/91/92 | 0.3 (spec) | Token protection: no saved executions for WF-91/92, handled errors, sanitised error_log, WF-90 redaction | CR-008 / GAP-012 | Architect |
