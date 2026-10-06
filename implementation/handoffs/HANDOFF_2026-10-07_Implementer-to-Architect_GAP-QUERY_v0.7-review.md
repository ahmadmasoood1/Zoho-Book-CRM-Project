=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-QUERY
REF:  Database.md v0.7 / Workflows.md v0.7 review · CR-021, CR-022 (proposed GAP-021)
DATE: 2026-10-07

VERIFIED OK (read-only review, not executed):
  - Database.md §4.6 extensions.gin_trgm_ops;
  - §5.1: search_path lines on the 4 PIN functions, verify_user_pin user_not_found (before the
    tenant_settings check), move_user_channel (user lock, id_in_use first, chat_id null, PIN kept
    unless p_clear_pin, drafts cancelled with count, session deleted, audit row with truncation);
  - §5.2 touch_session coalesce;
  - §5.6 second revoke after the grants + identity note;
  - M1 T2 wording.
  - Workflows.md §2 "Database access" row; §5.2 writer list incl. the WF-02 gate; WF-00 step 6
    (every message type + late_pin); WF-02 chat_id fill; WF-08 input checks 1–4 and the
    expired/action_invalid branch; WF-40 remove/move/reset; WF-06/WF-08 layout fixed.

FINDINGS
  (a) MEDIUM: the two views can be read with the anon key, and future tables would be open too.
      Source: https://supabase.com/docs/guides/database/postgres/row-level-security
        - "Views bypass RLS by default because they are usually created with the postgres user."
        - On Supabase, "a new table in public starts with every privilege already granted to all
          three roles" (anon, authenticated, service_role).
      Our 0007 revokes only the 16 tables listed by name. v_daily_health and
      v_tenants_without_owner are created later (views migration), so anon/authenticated
      receive SELECT by default, and the views run as their owner, past RLS. Anyone with the
      public anon key could read tenant names and daily counts through /rest/v1/v_daily_health.
      Any table added by a later milestone migration would be open in the same way.
      Recommend:
        1) Views: create … with (security_invoker = true) (Postgres 15+), AND
           revoke all on v_daily_health, v_tenants_without_owner from anon, authenticated;
        2) In 0007 add:
             alter default privileges in schema public revoke all on tables    from anon, authenticated;
             alter default privileges in schema public revoke all on sequences from anon, authenticated;
           so later tables, views and sequences start closed.
        3) Database.md §3 convention: every migration that creates a table enables RLS on it and
           adds it to the revoke list.
        4) Extend M1 T2: the anon key cannot read either view (and n8n's service_role still can).
  (b) LOW: the CR-021 (c) search_path fix also applies to zoho_save_tokens() and
      zoho_get_credentials(). They use pgp_sym_encrypt/pgp_sym_decrypt (pgcrypto) and have no
      "set search_path". This is fine under PostgREST on Supabase, but fails on plain Postgres.
      Add "set search_path = public, extensions" now. CR-005 (Vault) may replace these functions
      later.
  (c) LOW, consequence of CR-021 for the M3 note: PostgREST filters can't run a trigram similarity
      search, so WF-05's entity_cache lookup needs an RPC function. For example:
        search_entity_cache(p_tenant, p_entity, p_text, p_limit) returns setof entity_cache
      using similarity() / % from pg_trgm, with set search_path = public, extensions.
      No change now; please record it for the M3 design note.
  (d) LOW: WF-08 check 2 (Cancel) only describes a draft. Please state Cancel per state:
        - 'pin' + pending_action_id → draft cancelled (as written);
        - 'pin' + waiting read request → drop the request, no draft;
        - 'pin_setup' / 'pin_setup_confirm' → setup can't be cancelled; reply pin.setup_required.

QUESTION:  Why are the views and future objects left out of the 0007 revokes? Do you agree with
           the recommendations?
SEVERITY:  (a) Medium: close it before the M1 note is approved (it changes migrations 0007/0010
           and T2). (b)–(d) Low.
