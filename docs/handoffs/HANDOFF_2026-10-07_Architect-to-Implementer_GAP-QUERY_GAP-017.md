=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: GAP-QUERY
REF:  GAP-017 (new)
DATE: 2026-10-07

TITLE:     No PIN recovery when the only Owner forgets their PIN
WHERE:     WF-40 user.reset_pin (Owner + PIN); set_user_pin() (no workflow calls it after CR-012)
OBSERVED:  user.reset_pin needs the Owner's own PIN, and Phase 1 has one Owner. If that Owner
           forgets their PIN, nobody can reset it in chat, and every PIN-protected action (void,
           payments, reports, user admin) is blocked. No operator recovery procedure is
           documented, and set_user_pin() has no stated purpose any more.
EXPECTED:  FR-1.4; NFR-2 availability. The Owner can always regain control, and recovery
           is audited.
SEVERITY:  Medium
QUESTION:  Why is there no recovery path? Recommend one, for example:
           (a) an operator runbook: verify the Owner's identity outside the bot, run a function
               that clears pin_hash/pin_pending_hash and lockout and writes an audit_log row
               (action 'pin_recovery'), after which the Owner sets a new PIN in chat;
           (b) require a second Owner;
           or another option. Also: remove set_user_pin(), or keep it for the runbook?

PLEASE REPLY WITH ONE GAP-RESPONSE (box + file, Memory.md M-004):
  REASON / RECOMMENDATION / IMPACT
