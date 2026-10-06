=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-006
DATE: 2026-10-07

REASON:          The design assumed one message at a time per user (1–3 users). n8n runs each
                 webhook call as its own execution, and Telegram opens up to 40 parallel
                 connections by default (setWebhook max_connections, 1–100). A slow voice note
                 plus a quick text can therefore overwrite each other's session state.

RECOMMENDATION:  Change: optimistic locking.
  1. Column sessions.version int not null default 0.
  2. Function save_session(p_user, p_expected_version, p_state, p_last_intent, p_language)
     returns int: update … set …, version = version + 1 where user_id = p_user and
     version = p_expected_version, returning the new version (null = conflict).
     ALL state writes in WF-02/WF-06 go through it.
  3. On conflict: no automatic merge (the newer message may have started a different flow).
     Reply with message key session.busy: "I was still working on your previous message.
     Please send this again." The message is in message_log either way.
  4. A double button tap is already safe through claim_pending_action().
  5. Rejected: max_connections=1. WF-00 answers 200 at once and keeps processing, so it would
     not serialise the processing, and it would slow down every user of the bot.

IMPACT:
  - Database.md §4.4 (+1 column), §5 (+1 function); WF-02/06 use save_session().
  - New key session.busy in 4 languages.
  - Tests (M3): two parallel messages from one user → exactly one state write wins, the other
    gets session.busy, no corrupted state.
  - Low effort; blocks M3.
