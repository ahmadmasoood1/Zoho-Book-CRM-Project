=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: GAP-QUERY
REF:  GAP-006 / CR-018 · clarification C6
DATE: 2026-10-07

TITLE:     Which session writes bump sessions.version?
WHERE:     Your GAP-006 response; WF-02 step "load/create sessions row (extend expires_at)"
OBSERVED:  WF-02 writes the session on EVERY message (extends expires_at). If that write goes
           through save_session() and bumps version, then: a slow voice note reads version 5;
           a quick text arrives, WF-02 extends expiry → version 6; the voice note's WF-06 state
           write expects 5 → conflict → session.busy. The FIRST message is lost even though the
           second only extended expiry. Also unclear: the first message of a new user (session
           row doesn't exist yet), with two parallel inserts.
EXPECTED:  FR-2.5 / FR-3.2: no lost or corrupted conversation state; session.busy only for a
           real conflict.
SEVERITY:  Medium
QUESTION:  Why is the boundary not defined? Recommend: (1) which writes bump version (state
           changes only?) and which don't (expiry touch, last_seen); (2) how the session row is
           created safely (insert … on conflict); (3) which message gets session.busy.

PLEASE REPLY WITH ONE GAP-RESPONSE (box + file, Memory.md M-004):
  REASON / RECOMMENDATION / IMPACT
