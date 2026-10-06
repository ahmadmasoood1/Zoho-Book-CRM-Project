# Zoho Books Chat Assistant

A Telegram-first (WhatsApp later) chat assistant that lets a business owner and staff manage accounting by text, voice note or photo. Quotations, invoices, payments, customers, service items, expenses and reports are all saved in **Zoho Books**.

**Stack:** n8n (self-hosted) · OpenAI · Supabase (Postgres) · Zoho Books API v3 · Telegram Bot API (WhatsApp Cloud API in Phase 2)

**Current phase:** M0 — Documentation & sign-off. No implementation starts until the documents are approved (see `docs/Rules.md`).

## Start here
- [`PROJECT_OVERVIEW.md`](PROJECT_OVERVIEW.md): one-page summary of scope, decisions, MoSCoW, milestones and open questions

## Documents
| File | Purpose |
|---|---|
| [`docs/PRD.md`](docs/PRD.md) | Product requirements, MoSCoW, functional/non-functional requirements |
| [`docs/Architecture.md`](docs/Architecture.md) | System design, pipeline, workflows, data model, security |
| [`docs/milestones.md`](docs/milestones.md) | Milestones M0–M15 with deliverables and exit criteria |
| [`docs/Rules.md`](docs/Rules.md) | Architect/Implementer roles, docs-first rule, Gap Protocol, change process |
| [`docs/Workflows.md`](docs/Workflows.md) | n8n workflow register, specs, starter template, exported JSON store |
| [`docs/Database.md`](docs/Database.md) | Supabase schema (SQL), functions, seed data, retention |
| [`docs/SelfImprovement.md`](docs/SelfImprovement.md) | Lessons learned, retrospectives, role self-checks |
| [`docs/Templates.md`](docs/Templates.md) | WhatsApp message templates (EN/AR/UR) for Meta approval |

## Planned repository layout (for implementation)
```
docs/                    project documents (this folder)
n8n/workflows/           exported n8n workflow JSON (WF-xx_<name>.json)
supabase/migrations/     SQL migrations from docs/Database.md
prompts/                 versioned AI prompts and JSON schemas
tests/                   intent test set, receipt samples, evidence
```
