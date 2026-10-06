=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: REVIEW
REF:  GAP-005 / CR-002 rev 3 · clarification C3
DATE: 2026-10-07

VERDICT: Accepted. GAP-005 CLOSED (design complete; build verification at M1).
Thank you also for verifying the previous review: no new findings noted.

REVIEW OF YOUR PROPOSAL
- Option (a) accepted. Option (b) rejected for the reasons you gave (no code column, nullable
  tenant_id, check-then-insert not atomic, error_log purged).
- SQL checked: SET reads the old row and RETURNING the new; "last_alert_at = now()" is true only
  when this call set it; a never-refreshed connection (last_refreshed_at null) still alerts
  on its first failure via "last_alert_at is null"; no row returns null.

DOCS UPDATED (CR-002 rev 3)
- Database.md v0.4 §4.2: zoho_connections.last_alert_at timestamptz.
- Database.md v0.4 §5.3: zoho_refresh_failed() returns jsonb {alert, status}, exactly your SQL.
- Workflows.md v0.4 WF-91: alert only when alert = true; null result = INTERNAL + alert; T3d.
- M1 note v0.3 §7: T3d (a) 3 transient failures within 10 min → 1 alert; (b) after a successful
  refresh a new failure alerts at once; (c) still failing 10+ min later → 1 reminder.
- changes.md CR-002 rev 3; gaps.md C3 recorded, GAP-005 closed.

STILL BLOCKING M1 APPROVAL: CR-004 (Client: second instance), CR-005 (Supabase hosting).
Do not build yet.

NEXT FROM YOU (before M2/M3, as you planned): GAP-RESPONSEs for GAP-001, 002, 003, 004, 006,
007, 008, 009, 014. One handoff per box + file (Memory.md M-004).
