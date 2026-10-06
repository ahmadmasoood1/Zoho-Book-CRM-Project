=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-003
DATE: 2026-10-07

REASON:          Onboarding was treated as a WF-40 detail and never specified. Using status 'invited'
                 would conflict with the WF-02 lookup (status='active') and with M1 T1
                 (v_tenants_without_owner = 0 rows right after seeding the Owner).

RECOMMENDATION:  Change: PIN setup is driven by pin_hash IS NULL, not by status.
  1. New users (seed Owner and user.add) are created status='active', pin_hash null. 'invited'
     stays in the enum, unused in Phase 1 (kept for M15 self-service onboarding). The WF-02
     lookup and M1 T1 stay unchanged.
  2. WF-02 gate: if pin_hash is null, the ONLY thing the user can do is PIN setup. Every other
     message gets the setup prompt.
  3. Setup flow (PIN messages redacted and deleted per GAP-001):
       "Welcome, choose a 4–6 digit PIN" → awaiting='pin_setup'
       → pin_setup_first(user, pin) → "send it again" → awaiting='pin_setup_confirm'
       → pin_setup_confirm(user, pin) → match: "PIN set" / mismatch: start again.
  4. Schema: users.pin_pending_hash text (bcrypt of the first entry; never plain text).
       pin_setup_first(p_user, p_pin): format check ^[0-9]{4,6}$; pin_pending_hash = crypt(pin, gen_salt('bf',10)).
       pin_setup_confirm(p_user, p_pin) returns jsonb:
         - pending is null → {ok:false, reason:'restart'};
         - match → pin_hash = pin_pending_hash, pin_pending_hash = null, pin_set_at = now(),
           counters reset → {ok:true};
         - else → pin_pending_hash = null, {ok:false, reason:'mismatch'}.
       Setup mismatches don't count toward lockout.
  5. user.reset_pin (Owner + PIN) = clear pin_hash and pin_pending_hash, so the user sets a new
     PIN on their next message. The Owner never knows anyone's PIN, and set_user_pin(user, pin)
     is no longer called by any workflow (remove, or keep for admin scripts only).
  6. Finding the new user's Telegram ID: a bot can't look up a user ID from a username, and
     can't message someone who never wrote to it. So the unknown-sender reply (FR-1.1) adds:
     "Your ID: <their own id>. Ask the Owner to add you." The Owner then sends "add user <id>
     <name> <role>". Optional, for the Architect to decide: offer recent unknown senders as
     buttons instead.

IMPACT:
  - Database.md §4.3: +1 column; §5.1: +2 functions.
  - WF-02 gate, WF-40 user.add / reset_pin, WF-00 unknown reply text, message keys in 4 languages.
  - Tests (M2):
    (a) a new user sets a PIN in 2 steps;
    (b) a mismatch restarts setup;
    (c) a user without a PIN can't create anything;
    (d) after a reset the user must set a new PIN;
    (e) seed Owner → T1 still 0 rows.
  - Medium effort; blocks M2.
