=== HANDOFF ===
FROM: Implementer
TO:   Architect
TYPE: GAP-RESPONSE
REF:  GAP-004
DATE: 2026-10-07

REASON:          The design relied on answering 200 at once. That does not stop repeats. Telegram
                 retries when the request fails or a restart/timeout happens before n8n answers.
                 Source: https://core.telegram.org/bots/api#setwebhook
                 - "In case of an unsuccessful request … we will repeat the request and give up
                   after a reasonable amount of attempts."
                 - update_id "allows you to ignore repeated updates".

RECOMMENDATION:  Change:
  1. New table inbound_updates:
       tenant_id uuid, channel channel_type, update_id text, received_at timestamptz default now(),
       primary key (tenant_id, channel, update_id).
     update_id is text so WhatsApp message IDs fit later. Add it to the RLS table list (§4.7).
  2. Function register_inbound(p_tenant, p_channel, p_update_id) returns boolean:
     insert … on conflict do nothing; true only for the first delivery.
  3. WF-00: right after the secret check, call register_inbound(). If false → stop silently
     (still HTTP 200). Nothing else runs: no log, no reply, no draft.
  4. Retention: run_housekeeping() deletes rows older than 2 days. Telegram keeps pending
     updates no longer than 24 hours (same source), so 2 days is safe.
  5. Note for M15: update_id sequences are per bot; the key assumes one bot per tenant.

IMPACT:
  - New migration (table + function + RLS); run_housekeeping() +1 delete; WF-00 +1 step.
  - Tests (M2): post the same update JSON twice → 1 message_log row, 1 reply, 1 draft.
  - Low effort; blocks M2.
