=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-019
DATE: 2026-10-07

REASON:          Accounts were treated as fixed: one Telegram ID per user, for life. A new account
                 was only covered for new users (user.add), and a sole Owner can't add themselves.

RECOMMENDATION:  Change: an operator-only "move account" function, plus a combined runbook.
  1. New function move_user_channel(p_user_id uuid, p_new_channel_user_id text,
     p_clear_pin boolean, p_operator text, p_reason text) returns jsonb
     (plpgsql, one transaction):
       - new ID already used in that tenant + channel → {ok:false, reason:'id_in_use'}
         (checked first, so no raw unique-constraint error);
       - update users: channel_user_id = new ID, chat_id = null (WF-02 fills it from the next
         inbound message; no assumption about Telegram chat IDs). If p_clear_pin:
         pin_hash = null, pin_pending_hash = null, pin_failed_count = 0, locked_until = null;
       - cancel open drafts: pending_actions status 'pending'/'confirmed' → 'cancelled'
         ('executing' is left alone; it is already in flight);
       - delete the user's sessions row (the next message starts fresh through touch_session);
       - audit_log row: intent 'user.move_account' (free text, no FK), action 'channel_moved',
         details {operator (100), reason (200), old_channel_user_id, new_channel_user_id,
         pin_cleared};
       - return {ok:true, drafts_cancelled:n}.
     In 0008b: revoke execute on it from service_role, after the grants (same as recover_user_pin, L-005).
  2. Keep the PIN by default (p_clear_pin = false). The real Owner usually still knows their PIN.
     If the operator is ever fooled by a fake identity, PIN actions (payments, void, reports, user
     admin) stay protected. Clear it only when the Owner has also forgotten the PIN. This replaces
     a separate recover_user_pin() call in that case.
  3. Old account: it is no longer on the allow-list, so it becomes an unknown sender. Old preview
     buttons fail at WF-02 (not authorised). Drafts are cancelled (point 1). The session is deleted.
  4. New Telegram ID: the Owner messages the bot from the new account, gets the unknown-sender
     reply with their own ID (CR-012), and reads it to the operator during the identity call.
     Nothing to type in chat.
  5. Staff (and an Owner when another active Owner exists): in chat, user.remove the old account
     + user.add the new ID; no operator needed. The old row stays disabled with its audit
     history. Note: the history is split across two user rows; audit.summary should be able to
     list disabled users.
     Small related point: WF-40 user.remove should also cancel the removed user's open drafts
     (not specified today; WF-94 would expire them anyway within 30 min).
  6. Runbook: put both procedures in one file. Recommend renaming the CR-019 path to
     implementation/runbooks/account-recovery.md, with
       Procedure A: PIN recovery (recover_user_pin);
       Procedure B: account move (move_user_channel).
     Same identity check outside the bot, same audit check, same "tell the Owner" step.

IMPACT:
  - Database.md §5.1: +move_user_channel(); §5.6: +revoke from service_role.
  - WF-02: fill users.chat_id when it is null.
  - WF-40: remove cancels open drafts; reference to the runbook. Runbook path (CR-019 note).
  - M2 tests:
    (a) move → old ID is rejected, new ID works, PIN kept;
    (b) move with p_clear_pin → the new account must set a PIN;
    (c) the user's open drafts are cancelled and the session deleted;
    (d) a used ID → id_in_use;
    (e) service_role cannot execute it;
    (f) 1 audit_log row.
  - Low effort; blocks the M2 design note only.
