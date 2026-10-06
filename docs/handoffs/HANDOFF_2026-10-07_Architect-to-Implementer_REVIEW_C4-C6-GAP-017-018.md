=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  GAP-001 C4 / GAP-006 C6 / GAP-017 / GAP-018 · CR-010 rev 2, CR-018, CR-019, CR-020
DATE: 2026-10-07

VERDICT: All four accepted. GAP-001, 006, 017, 018 CLOSED. Docs updated first (Rules.md §5).
One new GAP-QUERY follows separately (GAP-019, the lost-Telegram-account point you raised).

DECISIONS
C4 / GAP-001 → CR-010 rev 2 APPROVED (narrow late-PIN rule as proposed).
   Workflows.md v0.6 §5.2: awaiting values null | field | choice | pin | pin_setup | pin_setup_confirm,
   plus last_pin_prompt_at. WF-00 late-PIN rule (digits AND awaiting null AND prompt within
   draft_ttl_minutes); WF-02 redaction; WF-06 sets last_pin_prompt_at, Cancel clears it;
   WF-08 late_pin mode → pin.not_awaited, clears it. Residual risk accepted.
C6 / GAP-006 → CR-018 APPROVED (TOUCH vs STATE).
   Database.md v0.6 §4.4 sessions.version; §5.2 touch_session(), save_session().
   Architect refinements: touch_session() also clears last_intent when an expired session is
   reset, and returns null if tenant_settings is missing (WF-02 → INTERNAL). save_session() takes
   the TTL from tenant_settings. Workflows.md v0.6 §5.2 write rules, §5.4 SESSION_BUSY →
   session.busy, WF-02/06/08.
GAP-017 → CR-019 APPROVED (option a; b recommended).
   Database.md v0.6: set_user_pin() REMOVED; recover_user_pin(p_user_id, p_operator, p_reason)
   (operator/reason truncated to 100/200 chars); §5.6 revoke from service_role AFTER the grants,
   with a note that any later "grant … on all functions" must repeat it; §9 operations row.
   Runbook location decided: implementation/runbooks/pin-recovery.md (your area), required
   steps listed in CR-019. client-checklist.md: second Owner recommended + identity-check method.
   M1 note v0.4 T2: service_role → permission denied on recover_user_pin().
GAP-018 → CR-020 APPROVED (all three). Answer to your "why": (a) and (b) were my oversights;
   (c) wasn't checked against Supabase's schema layout.
   Database.md v0.6 §4.1: create schema if not exists extensions; pgcrypto and pg_trgm WITH
   SCHEMA extensions; search_path note (security definer → extensions.crypt()).
   §5.1 verify_user_pin(): raise exception when tenant_settings is missing.
   §5.4 check_permission(): allowed requires u.pin_hash is not null.
   M1 note v0.4 T1: verify the extensions layout + search_path and record the Postgres version.

SCRATCH-DB TEST: your offer to execute the SQL on a local scratch DB before M1 is the
Owner's decision. I've asked the Owner separately. Until then, read-only review stands.

DOC VERSIONS NOW: Database.md 0.6 · Workflows.md 0.6 · M1 note 0.4 · client-checklist updated.
Records: docs/changes.md (CR-010 rev 2, CR-018, CR-019, CR-020), docs/gaps.md.

PLEASE VERIFY: Database.md v0.6 §4.1, §4.4, §5.1 (recover_user_pin, verify_user_pin), §5.2
(touch_session, save_session), §5.4, §5.6; Workflows.md v0.6 §5.2, WF-00/02/06/08/40.
Report findings as a GAP-QUERY (box + file).

STILL BLOCKING M1 APPROVAL: CR-004 (Client: PROD instance), CR-005 (Supabase hosting).
Do not build yet.
