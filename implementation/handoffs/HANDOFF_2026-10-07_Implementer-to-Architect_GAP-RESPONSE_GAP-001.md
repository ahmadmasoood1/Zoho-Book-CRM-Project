=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-001
DATE: 2026-10-07

REASON:          The PIN reply was only considered inside WF-06. WF-02 logs every message before WF-06
                 runs, and the "save failed executions = on" convention didn't consider secrets in
                 item data. The PIN is in item data of WF-00 (raw update), WF-01 (normalise) and WF-02.

RECOMMENDATION:  Change:
  1. PIN state in the session: whenever the bot asks for a PIN, it sets
     sessions.state.awaiting = 'pin' | 'pin_setup' | 'pin_setup_confirm' (setup: see GAP-003),
     plus the pending_action_id or the pending read request.
  2. Separate PIN path: after WF-02, WF-00 routes a PIN reply straight to a small dedicated
     PIN sub-workflow (Architect to assign the WF ID, e.g. "WF-08 PIN Handler"). It:
     - deletes the PIN message (Telegram deleteMessage);
     - calls the verify function (GAP-002) or the setup functions (GAP-003);
     - passes on only the result ({pin_ok, locked, ...}), never the PIN.
     So WF-06, WF-04 and OpenAI never see a PIN.
  3. Logging: WF-02 writes message_log.text = '[REDACTED]' when awaiting is a PIN state.
  4. Any message that is just 4–6 digits (^[0-9]{4,6}$) is also treated as a possible PIN, even
     when no PIN is awaited (e.g. sent late after expiry):
     - logged as '[REDACTED]', deleted, never sent to OpenAI;
     - reply "No action is waiting for a PIN".
  5. Execution data: in PROD, WF-00, WF-01, WF-02 and the PIN workflow save nothing (same 4
     settings as CR-008). Failures are still captured by WF-90 → error_log (workflow, node,
     execution ID, truncated error message, never item data). DEV keeps saving, because DEV uses
     test PINs only (Rules §2.2.7).
  6. Telegram fact (https://core.telegram.org/bots/api#deletemessage): "Bots can delete incoming
     messages in private chats" and "A message can only be deleted if it was sent less than 48
     hours ago". If deletion fails, it is logged (no retry) and the flow continues.

IMPACT:
  - Workflows.md §2: extend the CR-008 settings exception to WF-00/01/02 + the PIN workflow (PROD only).
  - New PIN sub-workflow spec. WF-00 routing step. WF-02 redaction step. WF-90: never log item data.
  - No schema change.
  - Tests (M2):
    (a) a PIN reply is '[REDACTED]' in message_log and deleted in the chat;
    (b) a bare 6-digit message with no PIN awaited is redacted and not sent to OpenAI;
    (c) forced failure with PROD settings → grep the n8n execution DB for the test PIN → 0 hits.
  - Medium effort; blocks M2.
