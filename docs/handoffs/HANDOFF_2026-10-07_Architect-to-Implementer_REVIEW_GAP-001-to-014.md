=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  GAP-001, 002, 003, 004, 006, 007, 008, 009, 014 · CR-010 … CR-018
DATE: 2026-10-07

VERDICT: All 9 responses justified. 7 approved in full, GAP-001 approved except point 4,
GAP-006 on hold. Docs updated first (Rules.md §5). Three separate GAP-QUERY handoffs follow
(C4, C6, GAP-017).

DECISIONS
GAP-001 → CR-010 APPROVED for points 1, 2, 3, 5, 6. Point 4 on hold → see GAP-QUERY C4.
          New ID: WF-08 PIN Handler (core, M2). Workflows.md v0.5: §2 PROD no-save exception for
          WF-00/01/02/08; WF-00 step 6 (PIN states → WF-08 and stop); WF-02 redaction; WF-06 no
          longer sees the PIN; new WF-08 spec; WF-90 never logs item data.
GAP-002 → CR-011 APPROVED. Database.md v0.5 §5.1. Architect refinement: verify_pin_for_action()
          locks the action row, checks it (action_invalid/expired → no count), then CALLS
          verify_user_pin() for counting and lockout, all in one transaction; on ok it sets
          pin_verified_at and status='confirmed'. verify_user_pin() returns pin_not_set and just_locked.
GAP-003 → CR-012 APPROVED. Database.md v0.5: users.pin_pending_hash; pin_setup_first() returns
          {ok,reason:'format'|'pin_already_set'} instead of raising; pin_setup_confirm() as proposed
          (+ 'pin_already_set'). Seed Owner is active with no PIN. WF-02 gate, unknown-sender reply shows the
          sender's own ID, WF-40 user.add by ID and reset_pin clears the PIN. New error code PIN_NOT_SET
          → pin.setup_required.
          Decision: recent-unknown-sender buttons are NOT in Phase 1.
          set_user_pin() and sole-Owner recovery → see GAP-QUERY GAP-017.
GAP-004 → CR-013 APPROVED. inbound_updates is placed in 0005_logs (so RLS in 0007 and privileges
          in 0008b apply in order, lesson L-005); added to the RLS list; register_inbound() in
          §5.4; 2-day purge. WF-00 step 3. M15 note on one bot per tenant.
GAP-006 → CR-018 DRAFT, ON HOLD → see GAP-QUERY C6.
GAP-007 → CR-014 APPROVED. New ID: WF-94 Draft Expiry (ops, M3, every 5 min). expire_drafts()
          in §5.2 returns tenant, user, intent, channel_message_id, channel, chat_id, language.
          Decision: a notice within 30–35 min of the preview meets FR-3.3.
GAP-008 → CR-015 APPROVED. pending_actions.redacted_at; redaction step in run_housekeeping()
          after the delete; PRD v0.2 NFR-5 reworded.
GAP-009 → CR-016 APPROVED. Null-tenant error purge (180 days); TENANT_ID in WF-00 and WF-90;
          milestones v0.2 M15 note on the bot→tenant mapping.
GAP-014 → CR-017 APPROVED. All intents seeded then "update intents set active = false";
          each milestone migration activates its own; check_permission() joins active intents;
          report.schedule added to the needs_pin list; Architecture v0.2 §8.

DOC VERSIONS NOW: Database.md 0.5 · Workflows.md 0.5 · PRD 0.2 · Architecture 0.2 ·
milestones 0.2 (Rules.md §9 table updated). Full records: docs/changes.md CR-010…018,
docs/gaps.md.

PLEASE VERIFY: the new SQL in Database.md v0.5 §4.3–§4.5, §5.1, §5.2, §5.4, §5.5, §7 matches
your proposals and is syntactically sound on the target Postgres version. Report any finding
as a GAP-QUERY.

STILL BLOCKING M1 APPROVAL: CR-004 (Client), CR-005 (hosting). Do not build yet.
