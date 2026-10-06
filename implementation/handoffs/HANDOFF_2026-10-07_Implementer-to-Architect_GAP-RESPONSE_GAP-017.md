=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-017
DATE: 2026-10-07

REASON:          user.reset_pin was designed as an Owner-on-Staff action. With one Owner in Phase 1,
                 the "Owner forgot their own PIN" case was missed, and CR-012 left set_user_pin()
                 without a purpose.

RECOMMENDATION:  Option (a) as the required path; (b) recommended, not required.
  1. New function recover_user_pin(p_user_id uuid, p_operator text, p_reason text) returns jsonb
     (plpgsql, one transaction):
       update users set pin_hash = null, pin_pending_hash = null, pin_failed_count = 0,
                        locked_until = null where id = p_user_id returning tenant_id …
       not found → {ok:false, reason:'user_not_found'};
       insert into audit_log (tenant_id, user_id, intent, action, result, details)
       values (…, p_user_id, 'user.reset_pin', 'pin_recovery', 'success',
               jsonb_build_object('operator', p_operator, 'reason', left(p_reason, 200)));
       → {ok:true}.
     The Owner's next message hits the WF-02 gate and they set a new PIN in chat (CR-012).
     The operator never knows or chooses a PIN.
  2. Not callable from chat or n8n: in 0008b, AFTER "grant execute on all functions … to
     service_role", add
       revoke execute on function recover_user_pin(uuid, text, text) from service_role;
     so only the database owner can run it (SQL editor / psql). The order matters (lesson L-005).
  3. Operator runbook (Implementer writes it in M2, path for you to decide, e.g.
     docs/runbooks/pin-recovery.md):
       1) request arrives;
       2) verify the Owner's identity OUTSIDE the bot: call back on the phone number on file plus
          one company detail agreed in the client checklist;
       3) run recover_user_pin() on PROD;
       4) check the audit_log row;
       5) tell the Owner to message the bot and set a new PIN;
       6) note it in the next daily update.
  4. set_user_pin(): REMOVE it. Nothing should set a known PIN any more (the user sets their own;
     recovery clears). Keeping an unused function that takes a PIN only adds attack surface
     through /rpc.
  5. (b) Second Owner: recommend it in the client checklist (then one Owner can reset the other in
     chat), but don't require it; the client may have only one owner.
  6. Related, for you to decide (no proposal yet): an Owner who loses their Telegram account needs
     channel_user_id/chat_id changed by an operator, with the same identity check. It could be a
     second step in the same runbook.

IMPACT:
  - Database.md §5.1: −set_user_pin(), +recover_user_pin(); §5.6: explicit revoke after the grant.
  - New runbook. client-checklist.md: second Owner (recommended) + the identity-check method.
  - Tests (M2):
    (a) recover_user_pin() clears the PIN and lockout and writes 1 audit_log row;
    (b) service_role calling it via /rpc → permission denied;
    (c) the Owner then sets a new PIN in chat.
  - Low effort.
