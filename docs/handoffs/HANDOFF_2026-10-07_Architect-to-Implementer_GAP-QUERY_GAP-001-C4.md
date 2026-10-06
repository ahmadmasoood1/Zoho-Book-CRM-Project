=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: GAP-QUERY
REF:  GAP-001 / CR-010 point 4 · clarification C4
DATE: 2026-10-07

TITLE:     Bare 4–6 digit messages treated as a PIN when no PIN is awaited
WHERE:     Your GAP-001 response, point 4
OBSERVED:  Point 4 treats ANY message matching ^[0-9]{4,6}$ as a possible PIN (redacted, deleted,
           not sent to OpenAI, reply "No action is waiting for a PIN"). Legitimate answers match
           too: when the bot asks for a missing field one at a time (FR-2.5), e.g. "What is the
           rate?" → "1500", "Quantity?" → "1000", or an invoice number reply "1234". These
           would be deleted and the flow would break.
EXPECTED:  FR-2.5 (ask for missing fields), FR-1.5 (PIN never stored in plain text).
SEVERITY:  Medium
QUESTION:  Why does point 4 apply when the session is waiting for a field answer? Recommend
           exact conditions, e.g. apply only when no PIN and no field question is pending,
           and say whether a late PIN after a draft expired is still caught.

PLEASE REPLY WITH ONE GAP-RESPONSE (box + file, Memory.md M-004):
  REASON / RECOMMENDATION / IMPACT
