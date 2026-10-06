=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-002
DATE: 2026-10-07

REASON:          verify_user_pin() was designed per user. Linking a correct PIN to the draft was left
                 to the workflow and never written down. Counting a missing PIN (pin_hash null) as a
                 failed attempt is a bug.

RECOMMENDATION:  Change:
  1. New function verify_pin_for_action(p_user_id uuid, p_action_id uuid, p_pin text) returns
     jsonb (plpgsql = one transaction). It locks the user row (for update), then:
     - The action must belong to the user, be needs_pin, status pending/confirmed, not expired.
       Otherwise return {ok:false, reason:'action_invalid' | 'expired'} with no PIN count.
     - pin_hash is null → {ok:false, reason:'pin_not_set'}, no count; the flow sends the user to
       PIN setup (GAP-003).
     - Locked → {ok:false, locked:true, locked_until}.
     - Correct → reset the counter, and in the same transaction:
         update pending_actions set pin_verified_at = now(), status = 'confirmed' where id = p_action_id;
       return {ok:true}.
     - Wrong → same counting and lockout as verify_user_pin(). Add just_locked:true when this
       call caused the lock, so WF-06 alerts the Owner exactly once (FR-1.3).
  2. verify_user_pin(): return {ok:false, pin_not_set:true} without counting when pin_hash is
     null; add the same just_locked flag. It stays for PIN-protected READ intents
     (report.run, needs_pin=true, is_write=false), which have no pending_actions row: the request
     waits in sessions.state and runs right after {ok:true}.
  3. claim_pending_action() is unchanged; it already requires pin_verified_at.

IMPACT:
  - Database.md §5.1: +1 function, verify_user_pin() fix; WF-06 / PIN workflow (GAP-001) call it.
  - Tests (M3):
    (a) correct PIN → pin_verified_at set → claim succeeds once;
    (b) 5 wrong PINs → lockout, exactly 1 Owner alert;
    (c) pin_hash null → pin_not_set, counter unchanged;
    (d) PIN for someone else's draft → action_invalid.
  - Low–medium effort; blocks M3.
