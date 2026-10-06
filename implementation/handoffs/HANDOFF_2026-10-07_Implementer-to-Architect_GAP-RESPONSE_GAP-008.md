=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-008
DATE: 2026-10-07

REASON:          The "keep drafts referenced by audit_log" rule protects the audit foreign key. Its
                 side effect on NFR-5 (payloads with customer and amount data kept for 5 years)
                 was not considered.

RECOMMENDATION:  Change: redact instead of delete for audited drafts.
  1. New column pending_actions.redacted_at timestamptz.
  2. In run_housekeeping(), after draft_retention_days:
     (a) drafts NOT referenced by audit_log → deleted (as today);
     (b) referenced drafts with redacted_at null:
           payload      = jsonb_build_object('meta', payload->'meta', 'redacted', true)
           preview_text = '[redacted]'
           error        = case when error is null then null else jsonb_build_object('code', error->'code') end
           redacted_at  = now()
     Kept: id, intent, status, zoho_entity / zoho_record_id / zoho_record_number, timestamps.
     That's enough to trace the audit row to the Zoho record; Zoho holds the data itself.
  3. PRD NFR-5 wording: "Drafts are deleted, or redacted when referenced by the audit log,
     after 7 days."

IMPACT:
  - Database.md §4.4 (+1 column), §5.5 (housekeeping); PRD NFR-5 wording.
  - Tests (M2): an executed draft older than 7 days keeps meta + Zoho IDs only, and its audit
    link is intact; a cancelled unaudited draft is deleted; a second run changes nothing.
  - Low effort; blocks M2.
