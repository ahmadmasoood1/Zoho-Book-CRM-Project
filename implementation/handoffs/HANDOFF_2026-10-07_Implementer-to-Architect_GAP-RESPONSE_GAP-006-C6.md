=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-006 / CR-018 · clarification C6
DATE: 2026-10-07

REASON:          I wrote "all state writes" without separating the expiry touch from conversation
                 state, and I didn't cover creating the first session row. Your scenario is right:
                 as written, a message that only extended expiry would cancel the previous one.

RECOMMENDATION:  Change: two kinds of session writes.
  1. TOUCH (no version bump): expires_at extension and sessions.language. Done once per message
     by WF-02 through a new function that also creates the row safely:
       touch_session(p_tenant uuid, p_user uuid, p_language text) returns sessions
       insert into sessions (tenant_id, user_id, language, expires_at)
       values (p_tenant, p_user, p_language, now() + ttl)        -- ttl from tenant_settings.session_ttl_minutes
       on conflict (user_id) do update set
         language   = excluded.language,
         expires_at = excluded.expires_at,
         state      = case when sessions.expires_at < now() then '{}'::jsonb else sessions.state end,
         version    = case when sessions.expires_at < now() then sessions.version + 1 else sessions.version end
       returning *;
     - Two parallel first messages: one inserts, the other updates, and both get the row.
       No error, version 0. (sessions.user_id is already unique.)
     - An expired session is reset to an empty state (a real state change → bump), so an old
       half-finished conversation never resumes.
  2. STATE (version bump): every write of state (awaiting, collected fields, options shown,
     pending_action_id, last_pin_prompt_at from C4) or last_intent. Done by WF-06 and WF-08 only,
     through save_session(p_user, p_expected_version, p_state, p_last_intent) returns int:
     update … where version = p_expected_version, set version = version + 1 and refresh
     expires_at; returns the new version, or null = conflict.
     p_expected_version is the version from touch_session() for this message.
  3. Read-only turns (reads, help, a reply that changes nothing) don't write state, so they never
     cause a conflict. In your scenario (slow voice note + quick text that only extends expiry,
     or only reads data), nothing conflicts.
  4. Which message gets session.busy: the one whose state write comes SECOND and finds a newer
     version. Its own state change is not saved. The reply names only the message type, never its
     content ("I was still working on your previous voice note/message. Please send this one
     again."). The first writer's state stays intact.
  5. PIN path: WF-08's DB result (pin_verified_at, counters) is saved by the PIN functions before
     its session write. If that session write conflicts, the user gets session.busy, and pressing
     Confirm again works because pin_verified_at is already set.

IMPACT:
  - Database.md §4.4: sessions.version (as in CR-018).
  - §5: +touch_session(), +save_session().
  - WF-02 uses touch_session() instead of load/create + extend; WF-06/WF-08 use save_session().
  - Tests (M3):
    (a) slow voice note + quick read-only text → no session.busy;
    (b) two state-changing messages → exactly one busy reply, for the later writer, and the
        state equals the first writer's;
    (c) two parallel first messages of a new user → one session row, no error;
    (d) message after session expiry → empty state.
  - Low effort.
