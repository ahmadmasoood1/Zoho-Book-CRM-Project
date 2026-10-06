=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-QUERY
REF:  Database.md v0.5 SQL review · CR-010…017 (proposed GAP-018)
DATE: 2026-10-07

VERIFICATION RESULT
  Matches my proposals: §4.3 pin_pending_hash; §4.4 redacted_at; §4.5 inbound_updates (in 0005,
  RLS list incl. inbound_updates); §5.1 verify_user_pin (pin_not_set, just_locked),
  verify_pin_for_action (your refinement is better than mine: action lock first, no count on an
  invalid action, reuses the counting logic), pin_setup_first/confirm; §5.2 expire_drafts;
  §5.4 check_permission (active intents), register_inbound; §5.5 redaction, null-tenant purge,
  inbound purge; §7 inactive intents, report.schedule needs_pin. WF-08 / WF-94 specs consistent.
  Lock order is safe: only verify_pin_for_action takes two row locks (action, then user), so no
  deadlock path.
  Syntax: reviewed by reading only, NOT executed. To my reading, all statements are valid on
  PostgreSQL 15/16/17. One spot to confirm at run time: expire_drafts() uses "language" as an
  output column name (an unreserved keyword, so it should parse). I will execute the full script
  on the provisioned Postgres version before M1 T1 (CR-007), or earlier on a local scratch DB if
  the Owner agrees.

FINDINGS (all Low; please log them as one gap or split them)
  (a) check_permission() allows an active user who has no PIN yet (pin_hash null).
      Today only the WF-02 gate stops them. Defence in depth: in check_permission(), use
        'allowed', rp.allowed and u.status = 'active' and u.pin_hash is not null
      so the data layer agrees with CR-012.
      (PIN setup never calls check_permission, so nothing breaks.)
  (b) verify_user_pin(): if the tenant has no tenant_settings row, s.* is null, the lockout
      comparison is null, and lockout silently never happens (attempts_left null).
      Recommend: if not found → raise exception 'tenant_settings missing'. Fail loud; WF-90 alerts.
  (c) §4.1 extensions: on Supabase, extensions normally live in the "extensions" schema
      (pgcrypto is usually pre-installed there). "create extension if not exists pg_trgm" may
      install into public. crypt()/gen_salt() then depend on the caller's search_path, and a
      future SECURITY DEFINER function with search_path='' (CR-005) must qualify them as
      extensions.crypt().
      Recommend: "create extension if not exists … with schema extensions", plus a note that
      functions run with search_path including extensions.
      I will verify the actual layout on the provisioned project at M1 (no doc change needed
      before that).

QUESTION:  Why are (a) and (b) not covered? Do you agree with the recommended changes?
SEVERITY:  Low (none blocks M1 approval)
