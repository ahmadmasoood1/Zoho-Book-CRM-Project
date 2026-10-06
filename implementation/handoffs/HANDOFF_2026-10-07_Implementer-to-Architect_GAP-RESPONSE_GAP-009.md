=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-009
DATE: 2026-10-07

REASON:          error_log.tenant_id is nullable because an error can happen before the tenant is
                 known. The purge joins tenant_settings, so null rows never match. Phase 1 has one
                 tenant, so the bot→tenant mapping was postponed without being written down.

RECOMMENDATION:  Change (small) + record for M15:
  1. run_housekeeping(): add
       delete from error_log where tenant_id is null and created_at < now() - interval '180 days';
     (180 days = the tenant_settings default).
  2. Phase 1: WF-00 sets tenant_id from env TENANT_ID as its first step, and WF-90 falls back to
     TENANT_ID. So unknown-sender message_log rows and almost all error_log rows carry the tenant
     and follow normal retention; null rows become rare.
  3. M15 (no change now): a bot→tenant mapping (e.g. table channel_bots: tenant_id, channel,
     bot identifier, webhook path) replaces TENANT_ID. Please add this to the M15 scope in
     milestones.md so it isn't lost.

IMPACT:
  - Database.md §5.5 (+1 line); WF-00 / WF-90 specs (TENANT_ID fallback); milestones.md M15 note.
  - Tests (M2): an error_log row with null tenant older than 180 days is purged; an
    unknown-sender message_log row has the tenant set.
  - Very low effort; blocks M2.
