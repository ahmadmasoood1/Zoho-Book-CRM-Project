# Client Onboarding Checklist — Zoho Books Chat Assistant

Issued by the Architect (M0 deliverable D5). Please return the items below. **Send secrets (passwords, API keys) through a password manager, never by chat or email.**

## A. Company & tax
- [ ] Legal company name (EN + AR) and trade licence number
- [ ] Registered address (EN + AR)
- [ ] TRN (Tax Registration Number) and VAT registration date
- [ ] Annual revenue band: **below / at or above AED 50M** (decides the e-invoicing date)
- [ ] Fiscal year start
- [ ] Logo (PNG/SVG, transparent background)
- [ ] Bank details to print on invoices (bank, IBAN, SWIFT, account name)

## B. Zoho Books
- [ ] Zoho Books plan chosen (must include multi-currency, custom roles, bills, POs, enough API calls)
- [ ] Who creates the Zoho account (admin email)
- [ ] Currencies you invoice in
- [ ] Default payment terms (e.g. Net 30)
- [ ] Opening data to import: customers, services and prices (spreadsheet)
- [ ] Expense categories you use (they become the mapped expense accounts)

## C. Users & permissions
- [ ] Owner: name, Telegram username, preferred language (EN/AR/UR/Roman Urdu)
- [ ] Staff: same details for each person
- [ ] Confirm the Staff permission matrix (Architecture §7): may Staff record payments? See reports?
- [ ] **Recommended:** a second Owner, so one Owner can reset the other's PIN in chat (CR-019)
- [ ] Identity check for PIN recovery: the Owner's phone number on file for a call-back, plus one company detail only the Owner knows (agreed here, not sent by chat)

## D. Documents
- [ ] Sample of your current quotation and invoice (for the bilingual template)
- [ ] Your accountant's contact (to approve the tax-invoice template and retention periods)

## E. Consent & policy
- [ ] **Written consent** that chat messages, voice notes, and images are processed by OpenAI (PRD NFR-4)
- [ ] Retention: audit logs 5 years, message logs 90 days (confirm with your accountant)

## F. Hosting
- [ ] Domain or subdomain for the bot server (e.g. `bot.example.ae`)
- [ ] Confirm hosting: dedicated server for the bot, or shared with existing automations (see GAP-011)
