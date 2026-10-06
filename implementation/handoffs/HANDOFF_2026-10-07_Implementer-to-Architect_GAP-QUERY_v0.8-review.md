=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-QUERY
REF:  Database.md v0.8 / Workflows.md v0.8 / M1 note v0.6 review · CR-023 (proposed GAP-022)
DATE: 2026-10-07

VERIFIED OK (read-only review, not executed):
  - Database.md §3 "new objects start closed" convention and the apply-order note
    (logical 00nn labels; 0010 views before 0009 seed);
  - §4.7 default-privilege revokes on tables + sequences, run as the owner;
  - §5.3 set search_path on zoho_save_tokens / zoho_get_credentials;
  - §6 0010_views with security_invoker = true + revoke;
  - Workflows.md WF-05 (RPC lookup, to be specified in M3) and WF-08 check 2 (Cancel per state);
  - M1 note v0.6 §3 rows 7/10/11 and the extended T2.
  All match CR-023 exactly.

FINDING (one, LOW; does not block the M1 note)
  The sequences that ALREADY exist when 0007 runs stay granted to anon/authenticated.
  "alter default privileges … on sequences" only covers sequences created later. The identity
  sequences of audit_log, message_log, ai_usage and error_log are created in 0005, before 0007,
  and the 0007 loop revokes on the tables only. So the Supabase default grants on those 4
  sequences remain.
  Practical exposure is minimal (PostgREST doesn't expose sequences), but it breaks the
  "everything starts closed" rule and T2 would not detect it.
  Recommend one line in 0007, after the loop:
    revoke all on all sequences in schema public from anon, authenticated;
  and add "no sequence in public is granted to anon/authenticated" (check
  information_schema.usage_privileges / has_sequence_privilege) to T2.

QUESTION:  Was leaving the existing sequences intended? If not, do you agree with the one-line fix?
SEVERITY:  Low
