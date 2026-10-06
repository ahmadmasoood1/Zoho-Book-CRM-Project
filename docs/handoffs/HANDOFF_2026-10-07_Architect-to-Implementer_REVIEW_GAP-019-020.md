=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  GAP-019 / GAP-020 · CR-021, CR-022 (+ CR-019 runbook path)
DATE: 2026-10-07

VERDICT: Both accepted. GAP-019 and GAP-020 CLOSED. Docs updated first (Rules.md §5).
All design gaps are now closed except GAP-011 (Client) and GAP-012/CR-005 (hosting).

ANSWER TO YOUR "WHY" (GAP-020)
(a) The credential was named but its database role was never pinned down; I didn't connect the
    privilege model (CR-003/009/019) to how n8n logs in. (b) WF-08 was designed for the happy
    path only. Lessons L-006 and L-007 added; the design-note template now asks for runtime
    identities and unexpected-input handling.

DECISIONS
GAP-020 → CR-021 APPROVED, all of (a)–(g).
  (a) DECIDED NOW: option (i). n8n reaches the DB ONLY via PostgREST with the service_role key
      (Supabase node for tables, HTTP Request to /rest/v1/rpc/<fn> for functions). No Postgres-node
      credential with the owner login. It holds for hosted and self-hosted Supabase, so it doesn't
      wait for CR-005; option (ii) only if we ever leave Supabase (raise a GAP).
      Workflows.md v0.7 §2 "Database access" row; Database.md v0.7 §5.6 identity note;
      M1 note v0.5 §6.2 + T2 (n8n identity can't run recover_user_pin / move_user_channel).
  (b) WF-08 input checks 1–4 as proposed (non-text deleted, never transcribed; Cancel; non-digit
      text not verified, no count; Cancel button on every PIN prompt; expired/action_invalid →
      clear state + draft.expired). WF-00 step 6 routes every message type in a PIN state, plus
      late_pin. New keys pin.type_it, pin.format.
  (c) set search_path = public, extensions on verify_user_pin, verify_pin_for_action,
      pin_setup_first, pin_setup_confirm; extensions.gin_trgm_ops in 0006.
  (d) §5.2 writer list + WF-02: PIN gate writes through save_session().
  (e) touch_session(): coalesce(excluded.language, sessions.language).
  (f) verify_user_pin(): user not found → {ok:false, reason:'user_not_found'}.
  (g) Layout fixed: WF-06 has its Exported JSON line again; WF-08 moved after WF-07 with one line.
GAP-019 → CR-022 APPROVED as proposed.
  Database.md v0.7 §5.1 move_user_channel() (PIN kept by default, p_clear_pin optional,
  id_in_use checked first, chat_id null, open drafts cancelled, session deleted, audit row);
  §5.6 revoke from service_role after the grants; §9 row.
  Workflows.md v0.7: WF-02 fills chat_id when null; WF-40 user.remove cancels open drafts,
  audit.summary lists disabled users, Staff moves via remove + add, sole Owner → Procedure B.
  Runbook renamed: implementation/runbooks/account-recovery.md (A: recover_user_pin,
  B: move_user_channel). CR-019 note updated.

DOC VERSIONS NOW: Database.md 0.7 · Workflows.md 0.7 · M1 note 0.5. Records: docs/changes.md
CR-021/022, docs/gaps.md GAP-019/020, docs/SelfImprovement.md L-006/L-007.

PLEASE VERIFY: Database.md v0.7 §4.6, §5.1 (search_path lines, user_not_found,
move_user_channel), §5.2 coalesce, §5.6 (second revoke + identity note); Workflows.md v0.7 §2,
§5.2, WF-00 step 6, WF-02, WF-08 checks, WF-40. Report findings as a GAP-QUERY (box + file).

SCRATCH-DB TEST: still the Owner's decision (asked separately).
STILL BLOCKING M1 APPROVAL: CR-004 (Client: PROD instance), CR-005 (Supabase hosting).
Do not build yet.
