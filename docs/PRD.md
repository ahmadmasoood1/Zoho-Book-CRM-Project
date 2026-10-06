# PRD — Zoho Books Chat Assistant

| | |
|---|---|
| **Document** | Product Requirements Document |
| **Version** | 0.2 (Draft for sign-off) |
| **Date** | 7 October 2026 |
| **Status** | Draft — must be approved before any implementation starts (see `Rules.md`) |
| **Related docs** | `Architecture.md`, `milestones.md`, `Rules.md` |

---

## 1. Purpose

Build a chat assistant that lets a business owner and their staff run day-to-day accounting by sending messages on **Telegram** (WhatsApp later). They can type, send a voice note, or send a photo. Every quotation, invoice, payment, customer, service item, expense, and report is created in and saved to **Zoho Books**. The bot does not keep its own copy of the accounting data.

## 2. Background and decisions so far

| Topic | Decision |
|---|---|
| Business model | One client now; designed so it can become a multi-client (SaaS) product later |
| First client | UAE services business (no stock), small team (1–3 users, under ~20 documents/day) |
| Zoho Books | No account yet — we set up a new Zoho Books UAE organisation from scratch |
| Channel at launch | Telegram. WhatsApp comes later |
| Input types | Free text, voice notes, photos/files |
| Languages | English, Arabic, Urdu, Roman Urdu (bot replies in the user's language) |
| Document language | Bilingual English + Arabic PDFs |
| Currency | Multi-currency |
| Tax | UAE VAT 5%. UAE e-invoicing readiness is an open item (revenue band unknown) |
| Users | Owner + staff with role-based permissions |
| Access | Allow-listed Telegram accounts + PIN for sensitive actions |
| Confirmation | Every write to Zoho Books needs a preview and explicit **Confirm** |
| Delivery to end customers | Bot sends the PDF back to the user; the user forwards it |
| Stack | Self-hosted n8n (VPS), OpenAI, Supabase (Postgres), Zoho Books API |
| Admin | In chat for now; web admin panel later |
| Timeline | No fixed date; phased by priority |

## 3. Target users

The target users are **business owners and account managers**. They use the bot to create quotations, invoices, and other records for **their own business customers**. Those business customers never talk to the bot.

### Persona 1 — Owner
- Runs the business and is responsible for the books.
- Wants answers fast: who owes money, how much was sold, what was spent.
- Approves sensitive actions and manages staff access.

### Persona 2 — Staff / Account Manager
- Creates quotations and invoices, records expenses, and adds customers and services.
- Usually on the move and prefers voice notes or quick text.
- Must not void documents, see financial reports, or manage users unless the Owner allows it.

## 4. Goals and success measures

| Goal | Success measure |
|---|---|
| Faster document creation | A quotation or invoice can be created by chat in **under 2 minutes** |
| Accuracy | **0** unconfirmed writes to Zoho Books; under 5% of drafts need more than one edit |
| Adoption | Owner and staff use the bot for **80%+** of quotes/invoices within 1 month of go-live |
| Trust and control | 100% of bot actions appear in the audit log with user, time, and Zoho record ID |
| Reliability | 99% of confirmed actions succeed or return a clear error the user can act on |

## 5. Scope — MoSCoW

### M — Must have (MVP)
1. Zoho Books UAE organisation setup: VAT 5% and TRN, currencies, numbering, bilingual templates, user roles.
2. Telegram bot channel.
3. Allow-listed users, roles (Owner/Staff), and a PIN for sensitive actions.
4. AI understanding of free text, voice notes, and photos in EN / AR / UR / Roman Urdu.
5. A preview with **Confirm / Edit / Cancel** before every write. The bot asks for missing fields and lets the user choose when a name is ambiguous.
6. Customers: add, update, view, search, balance.
7. Services (items): add, update, view, search, price.
8. Quotations: create, update, view, list, mark accepted/declined, convert to invoice, PDF.
9. Invoices: create, update, view, list (unpaid/overdue), void (Owner + PIN), PDF.
10. Payments received: record full or partial, receipt PDF.
11. Multi-currency transactions.
12. Expenses: receipt photo → extracted data → confirm → saved with the attachment.
13. Reports on request (PDF/Excel): Profit & Loss, Receivables Aging/Unpaid Invoices, Sales by Customer, Expense Summary.
14. Bot database (Supabase): users, roles, PINs, conversation state, audit log; multi-tenant-ready.
15. Reliability: automatic Zoho token refresh, retries, rate-limit handling, error alerts, no duplicates.
16. Self-hosted n8n with HTTPS, backups, and secure secrets.

### S — Should have
1. Scheduled reports (daily/weekly/monthly) to the Owner.
2. Vendors, bills, vendor payments, payables aging.
3. Purchase orders, and converting a PO to a bill.
4. Credit notes: create, apply, refund.
5. VAT summary/return report, expense details, purchases by vendor.
6. Quick questions ("How much does X owe?", "Today's sales").
7. WhatsApp channel.
8. UAE e-invoicing readiness through Zoho's accredited provider. This becomes **Must** if revenue is AED 50M or more.
9. AI usage and cost tracking.

### C — Could have
1. Web admin panel (users, roles, PINs, logs, usage).
2. Self-service onboarding for many businesses and subscription billing.
3. Sales orders.
4. Bank balances and recording bank transactions.
5. Sending quotes and invoices by email through Zoho Books.
6. Payment reminders through Zoho Books.
7. Recurring expenses.

### W — Won't have (this phase)
- Inventory/stock (services business)
- Recurring invoices, retainer invoices, projects and timesheets
- The bot messaging end customers directly
- Deleting records by chat (use **void** instead)
- Bank reconciliation, manual journals, chart of accounts changes
- KSA ZATCA e-invoicing (not applicable)
- Payroll, fixed assets, online payment collection

## 6. Functional requirements (Must-haves)

Each requirement has an ID so `milestones.md`, tests, and change requests can refer to it.

### FR-1 Access and security
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-1.1 | Only allow-listed Telegram user IDs can use the bot | A message from an unknown ID gets a polite "not authorised" reply, nothing else happens, and the attempt is logged |
| FR-1.2 | Each user has a role: **Owner** or **Staff** | Permission checks happen before any Zoho call (see the matrix in `Architecture.md`) |
| FR-1.3 | Sensitive actions require a PIN | Void, record payment, financial reports, and user admin ask for a PIN. 5 wrong tries lock the user for 15 minutes and alert the Owner |
| FR-1.4 | Owner manages users in chat | Owner can add/remove a user, change their role, and reset a PIN. All of these need a PIN |
| FR-1.5 | PIN is never stored in plain text | PIN is hashed. The PIN message is deleted from the chat where Telegram allows it |

### FR-2 Understanding messages
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-2.1 | Understands free text in EN, AR, UR, Roman Urdu | 90%+ intent accuracy on the agreed test set of 100 sample messages |
| FR-2.2 | Voice notes are transcribed and processed like text | The transcript is shown back in the preview |
| FR-2.3 | Photos/PDFs of receipts are read | Vendor, date, amount, VAT, and currency are pre-filled for confirmation |
| FR-2.4 | Replies in the user's language | Language is detected per message; the user can set a preferred language |
| FR-2.5 | Asks for missing required fields one at a time | No Zoho call is made with missing required data |
| FR-2.6 | Resolves ambiguous names | If more than one customer or item matches, the bot shows up to 5 options as buttons |
| FR-2.7 | Out-of-scope requests are handled politely | The bot explains what it can do and offers a short help menu |

### FR-3 Confirm-before-write
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-3.1 | Every create/update/void shows a preview | The preview lists all key fields, totals, VAT, and currency |
| FR-3.2 | Buttons: **Confirm / Edit / Cancel** | Edit lets the user change a field by text or voice, then shows a new preview |
| FR-3.3 | Drafts expire | An unconfirmed draft expires after 30 minutes and the user is told |
| FR-3.4 | No duplicates | Pressing Confirm twice, or a network retry, never creates two records (idempotency key) |

### FR-4 Customers
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-4.1 | Add customer | Name required. Optional: email, phone, address, currency, TRN, tax treatment, place of supply (emirate) |
| FR-4.2 | Update customer | Any of the fields above |
| FR-4.3 | View/search customer | Search by name, phone, or email; shows key details |
| FR-4.4 | Customer balance | Shows outstanding receivables in the customer's currency |

### FR-5 Services (items)
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-5.1 | Add service | Name and rate required; tax rate defaults to VAT 5% |
| FR-5.2 | Update service | Name, rate, description, tax |
| FR-5.3 | View/search services; check price | Returns the matching service(s) with rate |

### FR-6 Quotations
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-6.1 | Create quotation | Customer, line items (service, qty, rate, discount), currency, VAT, validity date |
| FR-6.2 | Update quotation | Only when the status allows it in Zoho Books |
| FR-6.3 | View/list quotations | Filter by customer, status, and date range |
| FR-6.4 | Mark accepted/declined | Status changes in Zoho Books |
| FR-6.5 | Convert to invoice | Creates an invoice linked to the quotation; preview first |
| FR-6.6 | Get PDF | Bilingual PDF is sent in the chat |

### FR-7 Invoices
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-7.1 | Create invoice | Same fields as a quotation, plus due date/payment terms |
| FR-7.2 | Update invoice | Only when allowed by status |
| FR-7.3 | View/list invoices | Filters: unpaid, overdue, paid, by customer, by date |
| FR-7.4 | Void invoice | Owner only + PIN |
| FR-7.5 | Get PDF | Bilingual tax-invoice PDF is sent in the chat |

### FR-8 Payments received
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-8.1 | Record payment | Customer, amount, date, mode, reference; applied to one or more invoices; PIN required |
| FR-8.2 | Partial payments | Remaining balance shown after saving |
| FR-8.3 | Receipt PDF | Sent in the chat |

### FR-9 Expenses
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-9.1 | Record expense from text/voice | Account/category, amount, date, currency, VAT, vendor (optional) |
| FR-9.2 | Record expense from a receipt photo | Fields extracted, user confirms, photo attached to the expense in Zoho Books |

### FR-10 Reports
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-10.1 | Profit & Loss | Any date range; PDF or Excel |
| FR-10.2 | Receivables aging / unpaid invoices | PDF or Excel |
| FR-10.3 | Sales by customer | PDF or Excel |
| FR-10.4 | Expense summary | PDF or Excel |
| FR-10.5 | Report access | Owner only + PIN (unless the Owner grants Staff access) |

### FR-11 Audit and logging
| ID | Requirement | Acceptance criteria |
|---|---|---|
| FR-11.1 | Every action is logged | User, time, intent, Zoho record ID, result, and an error if any |
| FR-11.2 | Owner can ask "what did <user> do today?" | Returns a summary from the audit log |

## 7. Example conversations

**Create invoice (text, English)**
> **User:** Invoice Al Noor Trading for 3 hours consulting at 500 AED and one website audit 2,000
> **Bot:** Draft invoice — Al Noor Trading (AED)
> • Consulting × 3 @ 500 = 1,500
> • Website audit × 1 @ 2,000 = 2,000
> Subtotal 3,500 · VAT 5% 175 · **Total AED 3,675** · Due in 30 days
> [Confirm] [Edit] [Cancel]
> **User:** *Confirm*
> **Bot:** ✅ Invoice INV-000123 created. [PDF attached]

**Ambiguous customer (Roman Urdu)**
> **User:** Ahmed ko quotation bana do 2 logo design 1500 each
> **Bot:** Ahmed naam ke 2 customers hain. Kaun sa? [Ahmed Traders] [Ahmed Khan Studio]

**Receipt photo**
> **User:** *(photo of a fuel receipt)*
> **Bot:** Expense draft — ENOC · 2 Oct 2026 · Fuel · AED 210.00 incl. VAT 10.00 · [Confirm] [Edit] [Cancel]

## 8. Non-functional requirements

| ID | Area | Requirement |
|---|---|---|
| NFR-1 | Performance | Text reply in under 5 seconds (p90); voice/photo under 15 seconds; PDF under 10 seconds |
| NFR-2 | Availability | 99% monthly uptime of the bot service |
| NFR-3 | Security | HTTPS only; Telegram webhook secret token; secrets in n8n credentials/env; Zoho tokens encrypted at rest; PINs hashed |
| NFR-4 | Privacy | Client agrees in writing that messages, voice, and images are processed by OpenAI; no accounting data is kept outside Zoho Books except logs and drafts |
| NFR-5 | Data retention | Drafts are deleted after 7 days, or redacted (Zoho IDs kept, content cleared) when referenced by the audit log (CR-015); message logs are kept for 90 days; audit logs for 5 years (UAE record-keeping — confirm with the client's accountant) |
| NFR-6 | Backups | Daily backups of the n8n DB and Supabase, kept for 14 days; restore tested before go-live |
| NFR-7 | Scalability | Every bot-side table has `tenant_id`; no client-specific values hard-coded in workflows |
| NFR-8 | Maintainability | Modular n8n workflows, one per domain; channel adapter layer so WhatsApp can be added without changing business logic |
| NFR-9 | Observability | Error workflow alerts the admin on Telegram; daily health summary |
| NFR-10 | Cost | AI usage logged per user and per day; monthly cost report |

## 9. Assumptions
- The client buys a Zoho Books plan that includes multi-currency, custom roles, vendor bills, purchase orders, and the API call volume needed.
- The client provides company details, TRN, logo, bank details, service list, and opening data.
- Telegram is acceptable for business use at launch.
- Volume stays small (≤ 3 users, ≤ 20 docs/day) during Phase 1.

## 10. Dependencies
- Zoho Books UAE organisation and API (OAuth 2.0 client)
- Telegram Bot API (BotFather token)
- OpenAI API account
- VPS with a domain and SSL certificate
- Supabase project (hosted or self-hosted)

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| AI misreads a message | Wrong data in the books | Confirm before every write; show a clear preview; regular test set |
| Zoho API limits or plan restrictions | Failed actions | Check the plan before purchase; caching; retries with backoff |
| Arabic/Urdu understanding quality | Poor experience | Test set in all 4 languages; fallback to button menus |
| UAE e-invoicing deadline | Compliance risk | Confirm revenue band early; plan accredited provider integration |
| Scope creep | Delays | MoSCoW + change process in `Rules.md` |
| Single point of failure (VPS) | Downtime | Backups, monitoring, documented restore |

## 12. Open questions

| # | Question | Owner | Status |
|---|---|---|---|
| Q1 | Client's annual revenue band (decides e-invoicing date) | Client | Open |
| Q2 | Final Staff permission matrix | Client | Open — proposal in `Architecture.md` §7 |
| Q3 | Written consent for OpenAI processing | Client | Open |
| Q4 | Bilingual PDF template meets UAE tax-invoice rules | Client's accountant | Open |
| Q5 | Zoho Books plan and data centre | Architect + Client | Open |
| Q6 | Retention periods confirmed | Client's accountant | Open |

## 13. Glossary
- **Draft / pending action:** what the bot plans to save, shown to the user before Confirm.
- **Intent:** what the user wants to do (e.g. `create_invoice`).
- **Tenant:** one client business. Phase 1 has one tenant.
- **TRN:** UAE Tax Registration Number.
- **Idempotency key:** a unique ID per action so it can never be saved twice.

## Change history

| Version | Date | Change | CR | By |
|---|---|---|---|---|
| 0.1 | 2026-10-04 | Initial draft | — | Architect |
| 0.2 | 2026-10-07 | NFR-5: audited drafts are redacted, not deleted | CR-015 | Architect |
