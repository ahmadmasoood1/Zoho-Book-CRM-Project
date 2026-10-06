=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: GAP-QUERY
REF:  GAP-019 (new; from your GAP-017 response, point 6)
DATE: 2026-10-07

TITLE:     No procedure when an Owner loses their Telegram account
WHERE:     WF-40; users.channel_user_id / users.chat_id; PIN-recovery runbook (CR-019)
OBSERVED:  If an Owner loses their Telegram account (new number, deleted account), the new account
           is an unknown sender. Only an Owner can add users, so a sole Owner can't get back
           in. Nothing describes moving a user to a new channel account.
EXPECTED:  FR-1.4; NFR-2. Access can always be restored, with an audit record.
SEVERITY:  Low (blocks M2 design note only)
QUESTION:  Please recommend the procedure. For example: extend the operator runbook with a function
           (operator-only, like recover_user_pin) that moves channel_user_id/chat_id to the new
           account after the same identity check, writes an audit_log row, and clears the PIN so a
           new one is set. Also say:
           - what happens to the old account's session and pending drafts (cancel?);
           - how the new Telegram ID is obtained (the unknown-sender reply already shows it, CR-012);
           - whether a non-Owner (Staff) moving accounts simply uses user.remove + user.add in chat.

PLEASE REPLY WITH ONE GAP-RESPONSE (box + file, Memory.md M-004):
  REASON / RECOMMENDATION / IMPACT
