=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-001 / CR-010 point 4 · clarification C4
DATE: 2026-10-07

REASON:          Point 4 was written to catch a PIN typed late, after its draft expired. I did not
                 check it against the field questions of FR-2.5, where bare numbers are normal
                 answers. Your examples are correct: as written, it would break those flows.

RECOMMENDATION:  Change point 4 to a narrow "late PIN" rule.
  1. Define the session "awaiting" values once (Workflows.md §5.2):
     null | 'field' (+ field name) | 'choice' (+ options) | 'pin' | 'pin_setup' | 'pin_setup_confirm'.
  2. When WF-06 asks for a PIN it also sets sessions.state.last_pin_prompt_at = now().
     WF-08 clears it on {ok:true}; Cancel clears it.
  3. A message is treated as a late PIN ONLY when ALL of these hold:
     - text matches ^[0-9]{4,6}$;
     - awaiting is null (no field question, no choice, no PIN state; PIN states go to WF-08 anyway);
     - last_pin_prompt_at is set and within the last 30 min (= draft_ttl_minutes).
     Then: WF-02 logs '[REDACTED]' → WF-00 routes to WF-08 (mode 'late_pin') → delete the message →
     reply "No action is waiting for a PIN. Your draft may have expired; please send the request
     again." → clear last_pin_prompt_at. Never sent to OpenAI.
  4. Everything else is normal text: "1500" answering "What is the rate?" (awaiting='field'),
     "1234" as an invoice-number answer, or any number with no recent PIN prompt.
  5. Late PIN after draft expiry: yes, still caught (within 30 min of the PIN prompt, and the
     session TTL of 60 min keeps the timestamp).
     Residual risk, accepted: a user typing their PIN with no PIN prompt in the last 30 min is not
     caught; it is handled as normal text. The bot can't tell it from a number.
  6. This changes session state, so it follows the C6 rules (save_session).

IMPACT:
  - Workflows.md §5.2 (awaiting values), WF-02 (rule), WF-06 (set last_pin_prompt_at),
    WF-08 (late_pin mode + clear). No schema change.
  - Tests (M2/M3):
    (a) "1500" as a field answer is accepted and not deleted;
    (b) a PIN sent 10 min after its draft expired is redacted, deleted, and answered;
    (c) "1234" with no PIN prompt in 30 min → normal flow;
    (d) after a successful PIN, a bare number → normal flow.
  - Low effort.
