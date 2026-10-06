=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-005 / CR-002 rev 2 · clarification C3
DATE: 2026-10-07

VERIFIED YOUR REVIEW (CR-002 rev 2 / CR-005 / CR-008 / CR-009): all claimed changes are present in
Database.md v0.3 (§4.7 note, §5.3 zoho_claim_refresh status='active', zoho_refresh_failed, §5.6
0008b with default grant), Workflows.md v0.3 (§2 exception, WF-91 branches + T3b/T3c, WF-92
CR-008 items 1-4), M1 note v0.2 (§2, §3 order 7/8/9/10, §5, T5, T7), changes.md, gaps.md GAP-016,
SelfImprovement L-005. No new findings.

REASON:          Oversight. C1b described the suppression behaviour but not where its state lives.

RECOMMENDATION:  Option (a), decided atomically inside the database.
  1. New column: zoho_connections.last_alert_at timestamptz (null = never alerted).
  2. zoho_refresh_failed() returns jsonb instead of void, and decides "alert or not" in the same
     UPDATE that records the failure:

     create or replace function zoho_refresh_failed(p_tenant uuid, p_permanent boolean, p_error text)
     returns jsonb language sql as $$
       update zoho_connections
          set status             = case when p_permanent then 'error' else status end,
              refresh_lock_until = case when p_permanent then null else now() + interval '5 seconds' end,
              last_error         = left(p_error, 500),
              last_alert_at      = case when p_permanent
                                          or last_alert_at is null
                                          or last_alert_at < now() - interval '10 minutes'
                                          or last_refreshed_at > last_alert_at
                                        then now() else last_alert_at end
        where tenant_id = p_tenant
       returning jsonb_build_object('alert', last_alert_at = now(), 'status', status);
     $$;

     (SET expressions read the old row values; RETURNING reads the new ones. now() is the
     transaction time, so "last_alert_at = now()" is true only when this call set it.)
  3. Exact behaviour:
     - Permanent (invalid_code): always alert ("reconnect Zoho"). It happens once per incident,
       because status='error' stops all further claims (CR-002 rev 2).
     - Transient, first failure of an incident: alert.
     - Transient, repeated within 10 min with no successful refresh in between: no alert.
     - Transient, still failing after 10 min: one reminder alert, then again every 10 min while it lasts.
     - A successful refresh (last_refreshed_at > last_alert_at) ends the incident; the next failure
       alerts at once.
     - 10 min is a constant, matching Zoho's 10-token / 10-minute minting window.
     - No "recovered" message is proposed (keeps scope to NFR-9 as written).
  4. WF-91 failure branch: call zoho_refresh_failed() → IF result.alert = true → admin alert;
     else nothing. If the function returns null (no row for the tenant), treat it as INTERNAL and
     alert.
  Why not (b) error_log lookup:
     - error_log has no code column (only error_message text) and tenant_id is nullable, so the
       match would be fragile.
     - Check-then-insert is not atomic: two failures close together could both alert.
     - Housekeeping purges error_log, and alert logic would depend on the log format.

IMPACT:
  - Database.md §4.2: +1 column zoho_connections.last_alert_at timestamptz.
  - Database.md §5.3: zoho_refresh_failed() returns jsonb {alert, status}.
  - Workflows.md WF-91: failure branch alerts only when alert = true; null result = INTERNAL + alert.
  - M1 note §7: new T3d. (a) Force a transient failure 3 times within 10 min → exactly 1 alert.
    (b) After a successful refresh, force a new failure → alert at once.
    (c) Still failing 10+ min later → 1 reminder. Evidence in M1/wf91.md.
  - Effort low; no other workflow affected.
