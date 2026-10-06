# Milestones — Zoho Books Chat Assistant

| | |
|---|---|
| **Version** | 0.1 (Draft for sign-off) |
| **Date** | 4 October 2026 |
| **Related docs** | `PRD.md`, `Architecture.md`, `Rules.md` |

There is no fixed deadline, so work is phased by priority. A milestone starts only when the previous milestone's **exit criteria** are met and signed off. Effort is a rough estimate for one implementer and will be re-estimated at the start of each milestone.

**Status key:** ⬜ Not started · 🟨 In progress · 🟦 In review · ✅ Done · ⛔ Blocked

---

## Overview

| # | Milestone | Phase | MoSCoW | Est. effort | Status |
|---|---|---|---|---|---|
| M0 | Documentation & sign-off | 1 | Must | 3–5 days | 🟨 |
| M1 | Infrastructure & Zoho Books setup | 1 | Must | 4–6 days | ⬜ |
| M2 | Bot core: channel, auth, roles, PIN, session, audit | 1 | Must | 5–7 days | ⬜ |
| M3 | AI understanding & confirm flow | 1 | Must | 7–10 days | ⬜ |
| M4 | Customers & services | 1 | Must | 3–4 days | ⬜ |
| M5 | Quotations & invoices (+ PDFs) | 1 | Must | 6–8 days | ⬜ |
| M6 | Payments received | 1 | Must | 3–4 days | ⬜ |
| M7 | Expenses with receipt photos | 1 | Must | 3–5 days | ⬜ |
| M8 | Reports on request | 1 | Must | 4–5 days | ⬜ |
| M9 | Hardening, UAT & go-live | 1 | Must | 5–7 days | ⬜ |
| M10 | Purchases: vendors, bills, vendor payments, POs, credit notes | 2 | Should | 7–10 days | ⬜ |
| M11 | Scheduled reports, quick questions, more reports, AI cost tracking | 2 | Should | 4–6 days | ⬜ |
| M12 | WhatsApp channel | 2 | Should | 5–8 days | ⬜ |
| M13 | UAE e-invoicing readiness | 2* | Should* | TBD | ⬜ |
| M14 | Web admin panel | 3 | Could | TBD | ⬜ |
| M15 | Multi-client (SaaS) onboarding & billing | 3 | Could | TBD | ⬜ |

\* M13 moves into Phase 1 as a **Must** if the client's revenue is AED 50M or more (see PRD Q1).

---

## Phase 1 — MVP (Must have)

### M0 — Documentation & sign-off
**Goal:** Agree on what we build and how, before any implementation.
- **Deliverables**
  - `PRD.md`, `Architecture.md`, `milestones.md`, `Rules.md` reviewed and approved
  - Permission matrix confirmed by the client (PRD Q2)
  - Intent catalogue + JSON schemas drafted for MVP intents
  - Test set v1: 100 sample messages (EN / AR / UR / Roman Urdu) with expected intents and fields
  - Client checklist issued: company details, TRN, logo, bank details, service list, users, consent for AI processing
- **Exit criteria:** All four documents are marked **Approved** by the Architect and the client; open questions are logged with owners.

### M1 — Infrastructure & Zoho Books setup
**Goal:** Environments ready; Zoho Books org configured.
- **Deliverables**
  - VPS with Docker, reverse proxy, HTTPS, firewall
  - n8n (self-hosted) with its own Postgres; editor protected
  - Supabase project; schema v1 from `Architecture.md` §6 (migrations in Git)
  - Zoho Books **test** org (DEV) and **client** org (PROD): VAT 5%, TRN, tax treatments, currencies, numbering, bilingual EN/AR templates, payment terms
  - Zoho OAuth client; tokens stored encrypted; WF-91 Zoho Auth and WF-92 Zoho Client working
  - Telegram bots: DEV bot + PROD bot
  - Daily backups configured; Git repo for workflows, prompts, schemas
  - Verify the report endpoints/export formats on the chosen plan (Arch T2)
- **Exit criteria:** A test call from n8n creates and reads a contact in the DEV org; a backup restore has been tested once.

### M2 — Bot core
**Goal:** Secure skeleton that receives and answers messages.
- **Deliverables:** WF-00 Inbound, WF-01 Channel Adapter, WF-02 Auth & Session, WF-40 User Admin, WF-90 Error Handler, WF-93 Housekeeping
- **Covers:** FR-1.1 – FR-1.5, FR-11.1, NFR-3, NFR-9
- **Exit criteria**
  - Unknown users are rejected and logged
  - Owner can add a Staff user, set a role, and reset a PIN in chat
  - PIN lockout works after 5 failures
  - Errors reach the admin chat
  - Every message is in `message_log`

### M3 — AI understanding & confirm flow
**Goal:** The bot understands requests and safely confirms them.
- **Deliverables:** WF-03 Media Normaliser, WF-04 Intent Parser, WF-05 Entity Resolver, WF-06 Draft & Confirm, WF-07 Router; prompt v1; schemas v1
- **Covers:** FR-2.1 – FR-2.7, FR-3.1 – FR-3.4
- **Exit criteria**
  - ≥ 90% intent accuracy on the test set (all 4 languages)
  - Voice notes are transcribed and processed
  - Ambiguity buttons work
  - Drafts expire after 30 minutes
  - A double Confirm creates only one record (idempotency test)

### M4 — Customers & services
- **Deliverables:** WF-10 Customers, WF-11 Items
- **Covers:** FR-4.1 – FR-4.4, FR-5.1 – FR-5.3
- **Exit criteria:** All customer and item intents pass end-to-end tests in DEV, in at least 2 languages each, including UAE tax treatment and place of supply.

### M5 — Quotations & invoices
- **Deliverables:** WF-12 Quotations, WF-13 Invoices; bilingual PDF delivery
- **Covers:** FR-6.1 – FR-6.6, FR-7.1 – FR-7.5
- **Exit criteria**
  - Create → preview → confirm → PDF works for AED and one foreign currency
  - VAT and totals match Zoho Books exactly
  - Convert quote → invoice works
  - Void is Owner + PIN only
  - The client's accountant approves the PDF template (PRD Q4)

### M6 — Payments received
- **Deliverables:** WF-14 Payments
- **Covers:** FR-8.1 – FR-8.3
- **Exit criteria:** Full and partial payments, payment across multiple invoices, a receipt PDF, a PIN request, and the correct balance shown afterwards.

### M7 — Expenses with receipt photos
- **Deliverables:** WF-15 Expenses; receipt extraction prompt
- **Covers:** FR-9.1 – FR-9.2
- **Exit criteria:** On 20 sample receipts (mixed languages and quality), ≥ 85% of fields are correct before user edits; the photo is attached in Zoho Books.

### M8 — Reports on request
- **Deliverables:** WF-20 Reports
- **Covers:** FR-10.1 – FR-10.5
- **Exit criteria:** All 4 reports are delivered as PDF and Excel for any date range; access rules are enforced.

### M9 — Hardening, UAT & go-live
- **Deliverables**
  - Security review (Architecture §11 checklist)
  - Load sanity test (2× expected volume)
  - User guide (EN + AR + UR, one page each); short training session
  - UAT with the Owner + 1 Staff for 1 week in DEV
  - Switch to the PROD bot and PROD Zoho org; monitoring on
- **Exit criteria:** UAT sign-off by the client; no open critical or high issues; restore drill passed; go-live checklist complete.

---

## Phase 2 — Should have

### M10 — Purchases
Vendors, bills, vendor payments, payables aging, purchase orders (+ convert to bill), credit notes.
**Exit criteria:** End-to-end tests pass; permissions and PINs are applied as agreed.

### M11 — Scheduled reports & quick questions
WF-30 Scheduled Reports; quick-question intents; VAT summary/return, expense details, purchases by vendor; AI cost report.
**Exit criteria:** The Owner receives a scheduled report on time for 2 weeks; quick answers match Zoho Books.

### M12 — WhatsApp channel
WhatsApp Business Platform setup, WF-00b Inbound, WhatsApp branch in WF-01, message templates.
**Exit criteria:** All MVP intents work on WhatsApp without changes to the domain workflows.

### M13 — UAE e-invoicing readiness
Confirm the revenue band and timeline; enable Zoho Books' accredited provider integration; handle any new required fields; show e-invoice status in the bot.
**Exit criteria:** Test e-invoices are accepted in the provider's test environment before the client's mandatory date.

---

## Phase 3 — Could have

### M14 — Web admin panel
Users, roles, PINs, permission matrix, logs, usage, and cost.

### M15 — Multi-client (SaaS)
Tenant onboarding (connect own Zoho Books via OAuth), plans and billing, tenant isolation tests, support process.

---

## Milestone sign-off log

| Milestone | Date | Signed off by | Notes |
|---|---|---|---|
| M0 | | | |
| M1 | | | |
| M2 | | | |
| M3 | | | |
| M4 | | | |
| M5 | | | |
| M6 | | | |
| M7 | | | |
| M8 | | | |
| M9 | | | |
