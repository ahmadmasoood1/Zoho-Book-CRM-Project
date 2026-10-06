# Workflows — Zoho Books Chat Assistant (n8n)

| | |
|---|---|
| **Version** | 0.1 (Draft for sign-off) |
| **Date** | 4 October 2026 |
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
| Settings | Execution order `v1`; **Error workflow = WF-90**; save failed executions = on; save successful executions = off in PROD (privacy) |
| Credentials (names) | `Telegram Bot (DEV)` / `Telegram Bot (PROD)`, `Supabase Service Role`, `OpenAI`, `Zoho Books OAuth (DEV/PROD)`, `Admin Alert Bot` |
| Environment variables | `ENV` (`dev`/`prod`), `TENANT_ID` (Phase 1), `DB_ENCRYPTION_KEY`, `TELEGRAM_WEBHOOK_SECRET`, `ADMIN_CHAT_ID`, `OPENAI_INTENT_MODEL`, `OPENAI_STT_MODEL`, `OPENAI_VISION_MODEL`, `PROMPT_VERSION` |
| Sub-workflows | Called with **Execute Workflow**; input via **Execute Workflow Trigger**; always return the **standard result** (§5.3) |
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
| WF-00b | WhatsApp Inbound | channel | WhatsApp webhook | Meta | WF-01, WF-02 | M12 | — | Planned |

**Status values:** Planned → Spec → Spec approved → Built (DEV) → Tested → Live (PROD) → Deprecated

## 4. Starter template (every sub-workflow begins from this)

Import into n8n (*Workflows → Import from file/clipboard*), rename, then build between `Validate input` and `Return result`. Node type versions may differ on your n8n version. If n8n upgrades a node on import, keep the upgraded version and note it in the change history.

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
      "parameters": { "inputSource": "passthrough" },
      "id": "a1b2c3d4-0000-4000-8000-000000000002",
      "name": "When called by another workflow",
      "type": "n8n-nodes-base.executeWorkflowTrigger",
      "typeVersion": 1.1,
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
  "session": { "state": {}, "last_intent": null },
  "intent": "invoice.create",
  "fields": { "customer_name": "Al Noor Trading", "line_items": [ { "item_name": "Consulting", "quantity": 3, "rate": 500 } ] },
  "resolved": { "customer_id": "4600000000012345", "currency_code": "AED" },
  "permission": { "allowed": true, "needs_pin": false },
  "prompt_version": "intent-v1"
}
```

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
| `DRAFT_EXPIRED` | Draft past `expires_at` | `draft.expired` |
| `ALREADY_EXECUTED` | Double confirm | `draft.already_done` |
| `AMBIGUOUS_ENTITY` | Multiple matches | `resolve.choose` |
| `NOT_FOUND` | No match in Zoho | `resolve.not_found` |
| `ZOHO_VALIDATION` | Zoho rejected the data | `error.zoho_validation` |
| `ZOHO_RATE_LIMIT` | 429 after retries | `error.busy_try_later` |
| `ZOHO_AUTH` | Token refresh failed | `error.service_unavailable` (+ admin alert) |
| `AI_UNCLEAR` | Low-confidence intent | `ai.clarify` |
| `INTERNAL` | Anything else | `error.generic` (+ admin alert) |

---

## 6. Workflow specifications

Each spec covers: **Purpose · Trigger · Input · Steps · Output · Errors · Tests · Exported JSON · Change history**.

### WF-00 Telegram Inbound
- **Purpose:** Entry point for every Telegram update.
- **Trigger:** Webhook (POST `/webhook/zb/telegram/<env>`). Telegram is registered with `secret_token = TELEGRAM_WEBHOOK_SECRET`.
- **Steps:**
  1. Check header `X-Telegram-Bot-Api-Secret-Token`. If wrong, return 401 and stop (log to `error_log`).
  2. Respond 200 immediately (Telegram must not retry).
  3. WF-01 (inbound) → normalised message.
  4. WF-02 → user, session (stop if not authorised).
  5. If button press → WF-06; else if voice/photo → WF-03 → WF-04; else → WF-04.
  6. WF-05 (if entities to resolve) → WF-06 (draft/preview) or WF-07 (reads).
  7. WF-01 (outbound) to send the reply.
- **Output:** none (reply sent through WF-01).
- **Errors:** any failure → WF-90.
- **Tests:** wrong secret rejected; unknown user rejected; text, voice, photo, and button each routed correctly.
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
- **Steps:** look up `users` by `(channel, channel_user_id, status='active')` → if none, log and reply `auth.not_authorised` → load/create `sessions` row (extend `expires_at`) → update `last_seen_at` → write `message_log` (inbound).
- **Tests:** unknown user; disabled user; locked user still can read but PIN actions are blocked.
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
- **Steps:** search `entity_cache` (trigram) → if stale or no hit, search Zoho through WF-92 → 0 matches: `NOT_FOUND` (offer to create); 1 match: resolved; 2–5 matches: `AMBIGUOUS_ENTITY` with buttons; more than 5: ask for more detail.
- **Tests:** exact, fuzzy, Arabic-name, and ambiguous cases.
- **Exported JSON:** `Pending — M3`

### WF-06 Draft & Confirm
- **Purpose:** Build the preview, store the draft, and handle Confirm / Edit / Cancel / PIN.
- **Steps (new write):** ask for missing required fields one at a time (state in `sessions`) → build the Zoho body + recompute totals and VAT → insert `pending_actions` (unique `idempotency_key`, `expires_at = now + draft_ttl`) → send the preview with buttons.
- **Steps (Confirm):** if `needs_pin` and not verified → ask for the PIN → `verify_user_pin()` → delete the PIN message where possible → `claim_pending_action()` → if no row → `ALREADY_EXECUTED`/`DRAFT_EXPIRED` → WF-07 → `complete_pending_action()` → `audit_log`.
- **Steps (Edit):** take the change by text/voice → rebuild → new preview (old draft cancelled). **Cancel:** status `cancelled`.
- **Tests:** double Confirm → one record; expired draft; PIN wrong ×5 → lockout + owner alert.
- **Exported JSON:** `Pending — M3`

### WF-07 Router
- **Purpose:** Send a validated intent and payload to the correct domain workflow; re-check permission just before execution.
- **Steps:** switch on `intent` domain → Execute Workflow (WF-10…WF-40) → return the standard result.
- **Exported JSON:** `Pending — M3`

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
- **Rules:** Owner + PIN; the last active owner cannot be removed or demoted; a new user sets their own PIN on first message (one-time setup flow).
- **Exported JSON:** `Pending — M2`

### WF-90 Error Handler
- **Trigger:** Error Trigger (set as the error workflow on every workflow).
- **Steps:** insert `error_log` → if a user is known, send `error.generic` in their language → alert `ADMIN_CHAT_ID` with workflow, node, execution ID (no personal data).
- **Exported JSON:** `Pending — M2`

### WF-91 Zoho Auth
- **Steps:** `zoho_get_credentials(tenant, DB_ENCRYPTION_KEY)` → if the access token is valid for more than 5 minutes, return it → else POST `{accounts_domain}/oauth/v2/token` (refresh_token grant) → `zoho_save_tokens()` → return the token. On failure, mark the connection `error` and alert the admin.
- **Exported JSON:** `Pending — M1`

### WF-92 Zoho Client
- **Input:** `{ tenant_id, method, path, query, body, accept }`.
- **Steps:** WF-91 token → HTTP request to `{api_domain}/books/v3{path}?organization_id=…` → `increment_zoho_usage()` → on 429/5xx retry with backoff 2s/4s/8s (max 3) → on 401 refresh the token once and retry → map Zoho errors to §5.4 codes → alert when daily calls exceed 80% of `zoho_daily_call_limit`.
- **Exported JSON:** `Pending — M1`

### WF-93 Housekeeping
- **Trigger:** Schedule, daily at 03:00 Asia/Dubai.
- **Steps:** `run_housekeeping()` → read `v_daily_health` and `v_tenants_without_owner` → send the daily health summary to the admin chat.
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

### WF-00b WhatsApp Inbound — v—
```json
{ "status": "Pending — M12" }
```

## 8. Change history

| Date | Workflow | Version | Change | CR / GAP | By |
|---|---|---|---|---|---|
| 2026-10-04 | All | 0.1 | Register, conventions, specs, starter template | — | Architect |
