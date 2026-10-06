=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  GAP-021 · CR-023
DATE: 2026-10-07

VERDICT: Accepted, all of (a)–(d). GAP-021 CLOSED. Docs updated first (Rules.md §5).
Thank you also for verifying CR-021/CR-022: no findings there.

ANSWER TO YOUR "WHY"
I designed the 0007 revokes as a fixed list of named tables, assuming new objects start
closed. On Supabase they start open, and views run as their owner past RLS. I didn't check the
platform defaults (same root cause as lesson L-007, now extended). Verified your sources on the
Supabase "Row Level Security" docs page (views bypass RLS; security_invoker on PG15+; new public
tables granted to anon/authenticated/service_role).

DECISIONS → CR-023 APPROVED
(a) Database.md v0.8:
    - §4.7: alter default privileges in schema public revoke all on tables / sequences
      from anon, authenticated (run as the owner role).
    - §6 is now "0010_views.sql": both views WITH (security_invoker = true) and
      revoke all on both views from anon, authenticated.
    - §3 convention: every migration that creates a table enables RLS on it and revokes
      anon/authenticated; every view uses security_invoker + revoke.
    - §3 note: 00nn labels are logical names; apply order = M1 note §3
      (… 0008 functions, 0008b privileges, 0010 views, 0009 seed last).
    - M1 note v0.6: rows 7/10 updated; T2 extended (anon can't read either view; a table created
      after 0007 is closed to anon; service_role still reads both views).
(b) set search_path = public, extensions on zoho_save_tokens() and zoho_get_credentials()
    (CR-005 may replace them later).
(c) Recorded in Workflows.md v0.8 WF-05: the trigram lookup goes through an RPC function
    (e.g. search_entity_cache(p_tenant, p_entity, p_text, p_limit)), to be specified in the
    M3 design note. No SQL added now.
(d) Workflows.md v0.8 WF-08 check 2: Cancel per state, exactly as you proposed.

DOC VERSIONS NOW: Database.md 0.8 · Workflows.md 0.8 · M1 note 0.6. Records: docs/changes.md
CR-023, docs/gaps.md GAP-021.

PLEASE VERIFY: Database.md v0.8 §3, §4.7, §5.3, §6; Workflows.md v0.8 WF-05, WF-08; M1 note v0.6
§3 + T2. Report findings as a GAP-QUERY (box + file), or reply "no findings".

STATUS: every design gap is closed except GAP-011 (CR-004, Client) and GAP-012/CR-005 (Supabase
hosting). SCRATCH-DB TEST: still the Owner's decision. Do not build yet.
