# Project Overview — Zoho Books Chat Assistant

> **Read this first in every new chat in this project.** It is the single-page reference for what this project is, what has been decided, and where the detailed documents are.
> Last updated: 4 October 2026 · Phase: **M0 — Documentation & sign-off** (no implementation yet)

---

## 1. What we are building (one paragraph)

A chat assistant that lets a business owner and their staff run day-to-day accounting by sending messages on **Telegram** (WhatsApp later). They can type, send a **voice note**, or send a **photo**. The bot creates quotations, invoices, payments, customers, service items, and expenses, and runs reports. **Everything is saved in Zoho Books**, the single source of truth. Built with **n8n (self-hosted)** + **OpenAI** + **Supabase** + **Zoho Books API v3**.

**Target users:** business owners and account managers, who use the bot to create documents for **their own business customers**. End customers never talk to the bot.

## 2. Key decisions (agreed with the user)

| Topic | Decision |
|---|---|
| Business model | One client now; designed to become a multi-client SaaS later (every table has `tenant_id`) |
| First client | **UAE** services business (no stock), small team (1–3 users, under ~20 docs/day) |
| Zoho Books | New UAE organisation set up from scratch; plan must support multi-currency, custom roles, bills, POs |
| Tax | UAE VAT 5% + TRN; **ZATCA does NOT apply** (that is KSA). UAE e-invoicing readiness depends on revenue band (unknown) |
| Channel | **Telegram first**, WhatsApp in Phase 2 |
| Input | Free text (AI), voice notes, photos/files (receipts) |
| Languages | English, Arabic, Urdu, Roman Urdu; bot replies in the user's language |
| Documents | Bilingual **English + Arabic** PDFs |
| Currency | Multi-currency |
| Users | Owner + Staff, **role-based** permissions (stored as data) |
| Security | Allow-listed Telegram IDs + **PIN** for sensitive actions (void, payments, reports, user admin) |
| Confirmation | **Every write** to Zoho shows a preview with **Confirm / Edit / Cancel** |
| Delivery | Bot sends the PDF **back to the user**, who forwards it (bot never messages end customers) |
| Hosting | **Self-hosted n8n on a VPS** |
| AI | **OpenAI** (LLM for intent JSON, speech-to-text, vision for receipts) |
| Bot database | **Supabase (Postgres)**: users, roles, PINs, sessions, drafts, audit/message logs. No accounting data |
| Admin | In chat now; web admin panel later (Could) |
| Timeline | No fixed date; phased by priority |

## 3. MoSCoW summary

- **Must (MVP):** Zoho UAE setup · Telegram bot · allow-list + roles + PIN · AI text/voice/photo in 4 languages · confirm-before-write · Customers · Services (items) · Quotations (incl. convert to invoice, PDF) · Invoices (void = Owner + PIN, PDF) · Payments received · Multi-currency · Expenses from receipt photos · Reports on request (P&L, receivables aging, sales by customer, expense summary; PDF/Excel) · Supabase audit/logs · reliability (token refresh, retries, idempotency) · secure VPS + backups.
- **Should:** scheduled reports · vendors, bills, vendor payments · purchase orders · credit notes · VAT/expense/purchase reports · quick questions · **WhatsApp** · UAE e-invoicing readiness (becomes Must if revenue ≥ AED 50M) · AI cost tracking.
- **Could:** web admin panel · SaaS onboarding & billing · sales orders · banking · Zoho emails/reminders · recurring expenses.
- **Won't (this phase):** inventory/stock · recurring/retainer invoices, projects/timesheets · bot messaging end customers · deleting records by chat (void only) · bank reconciliation/journals · ZATCA · payroll, fixed assets, online payments.

## 4. Architecture in brief

Telegram webhook → **n8n** pipeline: Channel Adapter → Auth & Session → Media Normaliser (voice/photo) → Intent Parser (OpenAI, JSON only) → Entity Resolver → Draft & Confirm (+PIN) → Router → domain workflow → **WF-92 Zoho Client** → reply + PDF → audit log.

Core rules: the LLM never calls Zoho; n8n recomputes totals/VAT; drafts are idempotent (`claim_pending_action`); domain workflows never call Telegram and channel workflows never call Zoho; permissions are stored in the `role_permissions` table; DEV and PROD are separate (test Zoho org + test bot).

## 5. Milestones

M0 Docs & sign-off (current) → M1 Infra & Zoho setup → M2 Bot core (auth, roles, PIN) → M3 AI + confirm flow → M4 Customers & services → M5 Quotes & invoices → M6 Payments → M7 Expenses (receipts) → M8 Reports → M9 Hardening, UAT, go-live → Phase 2: M10 Purchases · M11 Scheduled reports · M12 WhatsApp · M13 e-invoicing → Phase 3: M14 Admin panel · M15 SaaS.

## 6. Working rules (from Rules.md)

- **Roles:** **Architect** (designs, owns the documents; does not implement) and **Implementer** (builds exactly what is documented). Names are still placeholders.
- **No implementation before documentation:** the four core documents must be approved, and each milestone needs a design note first.
- **Gap Protocol:** if the Architect finds a gap, they **do not change anything**. They first ask the Implementer **why** it exists. A change is made only after the Implementer gives a proper justification and feedback, and then through a Change Request with the documents updated first.
- Every lesson learned goes into `SelfImprovement.md`.

## 7. Project documents (in this project under `docs/`)

| File | What it holds |
|---|---|
| `docs/PRD.md` | Requirements, personas, MoSCoW, functional requirements (FR IDs), NFRs, risks, open questions |
| `docs/Architecture.md` | System design, pipeline, workflows, data model, permissions, Zoho/AI/security design, ADRs |
| `docs/milestones.md` | M0–M15 with deliverables and exit criteria |
| `docs/Rules.md` | Architect/Implementer roles, docs-first gate, Gap Protocol, change process |
| `docs/Workflows.md` | n8n workflow register, conventions, specs, starter JSON template, exported JSON store |
| `docs/Database.md` | Supabase schema SQL (tested on Postgres 16), functions, seed data, retention |
| `docs/SelfImprovement.md` | Lessons log, retros, role self-checks, AI improvement log |
| `docs/Templates.md` | WhatsApp templates (EN/AR/UR) for Meta approval: utility, authentication, marketing |

Word versions of PRD, Architecture, Milestones, and Rules were also produced for management (navy/gold executive style).

## 8. Open questions (still to answer)

1. Client's annual revenue band, which decides the UAE e-invoicing go-live date (AED 50M+ → 1 Jan 2027; below → 1 Jul 2027, per the current FTA timeline).
2. Final Staff permission matrix (proposal in Architecture §7).
3. Client's written consent for OpenAI processing of messages, voice, and images.
4. Bilingual PDF template meets UAE tax-invoice rules (accountant check).
5. Zoho Books plan and data centre; Supabase hosted vs self-hosted; VPS provider.
6. Architect / Implementer / Client names for Rules.md.
7. Whether purchase orders are really needed (services business).
8. Marketing opt-in field for WhatsApp needs a CR to `Database.md` before M12.
