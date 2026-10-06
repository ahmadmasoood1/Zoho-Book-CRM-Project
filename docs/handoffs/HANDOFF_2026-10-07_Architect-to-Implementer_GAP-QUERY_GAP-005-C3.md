=== HANDOFF ===
FROM: Architect
TO:   Implementer
TYPE: GAP-QUERY
REF:  GAP-005 / CR-002 rev 2 · clarification C3
DATE: 2026-10-07

TITLE:     Where is the "last alert sent" time stored for WF-91 alert suppression?
WHERE:     Workflows.md v0.3 WF-91 failure branch; your C1b proposal
OBSERVED:  C1b says the transient-failure alert is suppressed while last_refreshed_at is unchanged
           and "an alert was sent in the last 10 min". No column or table in Database.md stores
           when the last alert was sent. zoho_refresh_failed() doesn't record it either.
EXPECTED:  One admin alert per incident, no alert storms (NFR-9), with every state in the
           documented schema.
SEVERITY:  Low
QUESTION:  Why does the proposal leave out where this time is kept? Options I can see:
           (a) a new column zoho_connections.last_alert_at, set inside zoho_refresh_failed()
               and returned to WF-91 so it can decide whether to alert;
           (b) a check of error_log for a recent ZOHO_AUTH row for this tenant.
           Or propose another.

PLEASE REPLY WITH ONE GAP-RESPONSE (box + file, Memory.md M-004):
  REASON / RECOMMENDATION (option + exact behaviour) / IMPACT
