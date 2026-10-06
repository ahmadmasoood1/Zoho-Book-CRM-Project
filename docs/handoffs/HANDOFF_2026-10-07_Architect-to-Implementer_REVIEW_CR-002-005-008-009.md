=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  CR-002 rev 2 / CR-005 / CR-008 / CR-009 · GAP-005 C1 / GAP-012 C2 / GAP-016
DATE: 2026-10-07

VERDICT: All points accepted. Docs updated first (Rules.md §5).

DECISIONS
1. CR-002 condition MET. Zoho minting limit (10 tokens / 10 min, 1 h lifetime, invalid_code)
   re-checked by Architect on the official page; recorded and ticked in M1 note §5.
2. C1a + C1b → CR-002 rev 2 APPROVED.
   Database.md v0.3 §5.3: zoho_claim_refresh() now requires status='active';
   new zoho_refresh_failed(p_tenant, p_permanent, p_error) as you proposed.
   Workflows.md v0.3 WF-91: loser result ZOHO_AUTH/refresh_wait_timeout, no loser alerts,
   winner token call 10 s timeout and no node retry, failure branch, tests T3b/T3c.
   Alert suppression: see the separate GAP-QUERY handoff (C3).
3. C2a → NEW CR-008 APPROVED (split from CR-005, independent of hosting).
   Workflows.md v0.3: §2 settings exception for WF-91/WF-92 (all saving off), WF-92 items 1–4
   (continueErrorOutput, sanitised error_log row, own alerts for ZOHO_AUTH/INTERNAL,
   whitelisted {ok,data,error} return), WF-90 redaction patterns.
   M1 note T5 (evidence = error_log rows) and T7 (n8n DB + error_log + Postgres logs, 0 hits)
   updated. Added to T7: confirm WF-90 still fires for an unhandled WF-92 error with saving off.
4. C2b → accepted as the CR-005 design (recorded in changes.md). CR-005 stays ON HOLD until
   Supabase hosted vs self-hosted is decided. If self-hosted: verify Vault there first.
5. GAP-016 → CR-009 APPROVED. Root cause is the Architect's CR-003 text (lesson L-005).
   Database.md v0.3: block removed from §4.7; new §5.6 0008b_function_privileges.sql, applied
   right after functions, with the default grant to service_role added; any other n8n DB role
   is granted too and named in Workflows.md §2.
   M1 note v0.2 §3 order: 7 rls · 8 functions · 9 function_privileges · 10 views · 11 seed.
6. M1 note minor items done: §2 now references CR-002/CR-006/CR-008, §5 ticked, T3b/T3c added,
   version 0.2.

STILL BLOCKING M1 APPROVAL: CR-004 (Client), CR-005 (hosting), C3 (your answer).
Do not build yet.

FORMAT NOTE (Memory.md M-002 / M-004, rewritten 2026-10-07): put each complete handoff inside ONE
fenced block, one handoff per block, AND save the same content as a file in
implementation/handoffs/HANDOFF_<date>_Implementer-to-Architect_<TYPE>_<REF>.md, then give the
Owner that file as an attachment. A file always arrives in the other thread as an attachment.
