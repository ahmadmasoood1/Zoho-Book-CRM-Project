=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-QUERY
REF:  Database.md v0.6 / Workflows.md v0.6 review · CR-010 rev 2, CR-018, CR-019, CR-020 (proposed GAP-020)
DATE: 2026-10-07

VERIFIED OK (read-only review, not executed):
  - Database.md §4.1 extensions schema;
  - §4.4 sessions.version;
  - §5.1 recover_user_pin (one transaction, audit row, truncation) and verify_user_pin raise;
  - §5.2 touch_session (insert…select…on conflict, expired reset incl. last_intent, null without
    tenant_settings) and save_session (conditional update, TTL from tenant_settings);
  - §5.4 check_permission pin_hash rule;
  - §5.6 revoke after the grants;
  - Workflows.md §5.2 awaiting values + write rules; §5.4 PIN_NOT_SET / SESSION_BUSY;
    WF-00/02/06/08/40 match CR-010 rev 2, CR-012, CR-018, CR-019.
  Your touch_session refinements (clear last_intent, null → INTERNAL) are improvements.

FINDINGS
  (a) MEDIUM: the database role n8n uses is not defined, and the privilege model depends on it.
      Workflows.md §2 names the credential "Supabase Service Role (Postgres or Supabase credential)".
      With a Postgres-node credential, the usual Supabase login is "postgres", the OWNER of all
      tables and functions. The owner ignores the CR-019 revoke (it can run recover_user_pin()),
      and RLS without policies does not restrict it. So CR-003/009/019 would not hold for n8n.
      Recommend: n8n never connects as the owner role. Options:
        (i)  all DB access through PostgREST with the service_role key (Supabase node for tables,
             HTTP Request to /rest/v1/rpc/<fn> for functions). This exactly matches the current
             grants. Preferred.
        (ii) a dedicated login role (e.g. n8n_app) with explicit grants. RLS bypass then needs
             BYPASSRLS or policies; verify on the chosen hosting.
      Decide at M1 together with CR-005 (hosting), and record it in Workflows.md §2. Add to M1 T2:
      "n8n's DB identity cannot execute recover_user_pin()".
  (b) MEDIUM: input in a PIN state is always treated as a PIN attempt.
      WF-00 step 6 sends EVERY message in awaiting 'pin' to WF-08. Then:
        - "cancel" or a new request typed by the user is counted as a wrong PIN by
          verify_user_pin(); the user can be locked out without entering a PIN;
        - a voice note or photo in a PIN state could contain a spoken PIN, and must not go to
          STT/vision;
        - after WF-94 expires the draft, the session still says awaiting 'pin'.
      Recommend, in WF-08 before calling any PIN function:
        1) non-text (voice/photo/document) → delete it, never transcribe, reply "please type your PIN";
        2) text not matching ^[0-9]{4,6}$ → do NOT verify (no count). A Cancel button press /
           cancel word → cancel flow. Otherwise reply "send your 4–6 digit PIN or tap Cancel";
        3) the PIN prompt always carries a Cancel button;
        4) verify_pin_for_action() → reason expired/action_invalid → clear awaiting and
           last_pin_prompt_at, reply draft.expired.
      The same format check covers pin_setup (pin_setup_first already returns 'format').
  (c) LOW: search_path on plain Postgres (self-hosted non-Supabase, or the scratch test).
      The default search_path there is "$user", public. So crypt()/gen_salt() in the PIN
      functions fail at run time, and in 0006 "using gin (name gin_trgm_ops)" fails at migration
      time.
      Recommend: add "set search_path = public, extensions" to verify_user_pin,
      verify_pin_for_action, pin_setup_first, pin_setup_confirm; write extensions.gin_trgm_ops in
      0006. This holds whatever the caller's search_path is.
  (d) LOW: WF-02 writes state, but §5.2 says only WF-06/WF-08 do. The PIN gate sets
      awaiting='pin_setup', and so does the late-PIN logging decision. Recommend: add "WF-02
      (PIN gate only), through save_session()" to the writer list in §5.2 and WF-02.
  (e) LOW: touch_session() sets language = excluded.language, so a null p_language erases the
      stored language. Recommend coalesce(excluded.language, sessions.language).
  (f) LOW: verify_user_pin() for a non-existent user raises "tenant_settings missing for tenant
      <NULL>", which is misleading. Recommend: after selecting the user, if not found → return
      {ok:false, reason:'user_not_found'}.
  (g) COSMETIC: in Workflows.md, the WF-08 section sits inside WF-06. WF-06 has lost its
      "Exported JSON: Pending — M3" line and WF-08 now shows two. Also, WF-00 step 6 and the WF-08
      trigger don't mention the late_pin route.

QUESTION:  Why are (a) and (b) not covered? Do you agree with the recommendations?
SEVERITY:  (a) and (b) Medium: must be closed before the M1 and M2 design notes respectively.
           (c)–(g) Low.
