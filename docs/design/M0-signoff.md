# M0 — Documentation & sign-off · Readiness Tracker

| | |
|---|---|
| **Status** | In progress |
| **Owner** | Architect |
| **Date** | 2026-10-06 |

M0 exits when all four core documents are **Approved** (`Rules.md` §9) and open questions have owners (`milestones.md` M0).

## 1. Deliverables

| # | Deliverable (`milestones.md` M0) | Owner | Status | Notes |
|---|---|---|---|---|
| D1 | PRD, Architecture, milestones, Rules reviewed | Architect | 🟨 Review done 2026-10-06 | 15 gaps logged in `gaps.md`; 3 High |
| D2 | Permission matrix confirmed by client (PRD Q2) | Client | ⬜ | Proposal: Architecture §7; also see GAP-014 |
| D3 | Intent catalogue + JSON schemas for MVP intents | Architect | ⬜ | To be written by the Architect in `docs/design/M0-intent-schemas.md` |
| D4 | Test set v1: 100 messages EN/AR/UR/Roman Urdu | Architect + Client | ⬜ | To be written by the Architect in `docs/design/M0-intent-test-set.md` (anonymised) |
| D5 | Client checklist issued | Architect | ✅ Drafted | `docs/client-checklist.md` |
| D6 | Role names filled in `Rules.md` §1 | Client | ⬜ | Open question #6 |

## 2. Gaps that must be answered before M1 starts
Must be closed (justified or resolved through a CR) before the M1 design note can be approved: **GAP-005, GAP-010, GAP-011, GAP-012, GAP-013, GAP-015**.

**Status 2026-10-07:** GAP-005 (CR-002 rev 2), GAP-010 (CR-003), GAP-013 (CR-006), GAP-015 (CR-007), and GAP-016 (CR-009) are resolved; GAP-012 is partly resolved (CR-008). **Still blocking M1 approval:** CR-004 (Client approval of a second instance), CR-005 (Supabase hosted vs self-hosted), and ~~clarification C3~~ (closed 2026-10-07, CR-002 rev 3).
The rest can close before the milestone they block (see the `gaps.md` register).

## 3. Open questions → owners

| # | Question (PROJECT_OVERVIEW §8) | Owner | Needed by |
|---|---|---|---|
| 1 | Revenue band → e-invoicing date | Client | M1 (decides whether M13 becomes a Must) |
| 2 | Staff permission matrix | Client | M2 |
| 3 | Written consent for OpenAI processing | Client | M3 (before any real message goes to OpenAI) |
| 4 | Bilingual PDF meets UAE tax-invoice rules | Client's accountant | M5 |
| 5 | Zoho plan + data centre; Supabase hosted vs self-hosted; VPS | Architect + Client | M1 · *VPS seems to be Contabo (existing n8n), to be confirmed under GAP-011* |
| 6 | Architect / Implementer / Client names | Client | M0 |
| 7 | Are purchase orders needed? | Client | M10 |
| 8 | Marketing opt-in field (CR to Database.md) | Architect | M12 |

## 4. Sign-off
Fill in `Rules.md` §9 (not here) once D1–D6 are done.

## 5. Gap status (2026-10-07)
- **Closed:** GAP-001 … GAP-022, except the two below. Implementer verification of the design docs: **no findings** (2026-10-07, after CR-024).
- **Partly open:** GAP-012 (CR-005: Supabase hosting).
- **Open:** GAP-011 (CR-004: Client approval of the PROD instance).
