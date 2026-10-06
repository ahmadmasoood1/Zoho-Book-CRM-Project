=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-007
DATE: 2026-10-07

REASON:          The design assumed the user finds out on Confirm (DRAFT_EXPIRED). FR-3.3 says
                 "the user is told", which needs a proactive message. WF-93 runs only once a day.

RECOMMENDATION:  Change: keep the Confirm-time message, and add a proactive notice.
  1. New small ops workflow (Architect to assign the ID, e.g. "WF-94 Draft Expiry"):
     Schedule every 5 min.
  2. New function expire_drafts() returns setof rows:
       update pending_actions set status = 'expired'
        where status in ('pending','confirmed') and expires_at < now()
       returning id, user_id, intent, channel_message_id;
     (join users for chat_id and preferred_language).
     One UPDATE, so each draft is expired and notified exactly once.
  3. For each row, WF-01 outbound:
     (a) edit the preview message to remove its buttons (edit_message_id = channel_message_id),
         so stale buttons can't be pressed;
     (b) one short new message, key draft.expired_notice: "Your draft <intent label> expired.
         Send it again if you still need it." Several drafts of one user → one combined message.
  4. Ops calling WF-01 is allowed (WF-93 already does). The expire step in run_housekeeping()
     stays as a safety net.
  5. Timing: notice within 30–35 min of the preview. Please confirm this tolerance against FR-3.3.

IMPACT:
  - Database.md §5.2 (+1 function); new ops workflow spec; message key in 4 languages.
  - Tests (M3):
    (a) leave a draft 30 min → within 5 min the buttons are gone and the notice arrives;
    (b) tapping an old button → draft.expired;
    (c) 2 drafts expiring together → 1 message.
  - Low effort; blocks M3.
