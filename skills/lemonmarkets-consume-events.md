---
name: Consume webhook events and mirror lemon.markets state
description: Register a webhook, verify and acknowledge deliveries, and maintain a correct local mirror of accounts, orders and trades from the lemon.markets event stream.
api: openapi/lemonmarkets-brokerage-openapi.json
generated: '2026-07-19'
method: generated
source: openapi/lemonmarkets-brokerage-openapi.json + https://developer.lemon.markets/docs/mirror-data-using-events
operations:
  - create_webhook
  - list_webhooks
  - delete_webhook
  - list_events
  - get_account
  - get_order
---

# Consume webhook events and mirror lemon.markets state

lemon.markets emits **85 event types** across accounts, orders, trades, withdrawals,
settlements, corporate actions, taxes, treasury and workflows. This skill covers registering
for them and processing them without corrupting your local state.

## Steps

1. **Register the webhook.** Call `create_webhook` (`POST /webhooks`) with your HTTPS endpoint
   and the event types you want:

   ```json
   { "url": "https://your-app.example/webhooks/lemon", "events": ["account.created"] }
   ```

   The `201` returns `id` (`wbh_…`), `url`, `events`, and a **`signature_secret`**. Store the
   secret in your secret manager — it is how you authenticate inbound deliveries, and it is
   returned only at registration.

2. **Verify every inbound delivery** against the `signature_secret` before doing anything else.
   Reject unverified payloads.

3. **Acknowledge in under 10 seconds.** Return `200 OK` as soon as the event is safely queued.
   Deliveries that do not get a fast `200` are retried.

   ```
   POST /webhooks/lemon
   → verify signature
   → enqueue event
   → return 200 OK immediately
   ```

   **Never process synchronously in the webhook handler.**

4. **Process from the queue.** Each event has `id` (`evt_…`), `type`, `created_at` and a
   `context` object of related entity identifiers:

   ```json
   {
     "id": "evt_a8b11a37eccc49ecacb372e3a92a184d",
     "type": "account.created",
     "created_at": "2023-06-29T14:40:30.523373+00:00",
     "context": { "account": "cusa_33531f7f801d4ce997747470ab7208fd" }
   }
   ```

5. **Dedupe on the event `id`.** Delivery is at-least-once. Store processed ids and drop repeats.

6. **Re-fetch state from the API.** Read `type` to learn *what changed*, then call the relevant
   read operation — `get_account` (`GET /accounts/{account_id}`),
   `get_order` (`GET /accounts/{account_id}/orders/{order_id}`), and so on — and write **that**
   into your mirror. The event tells you *when* to sync, not *what* to sync to. Parallel
   fetches are fine; the API supports HTTP/2 multiplexing with 100+ streams per connection.

7. **Order per entity.** Partition your queue by entity id so events for one order or account
   are processed in sequence. Different entities can run in parallel.

8. **Handle out-of-order arrivals** by trusting the API. Never skip an event; fetch current
   state, reconcile, mark processed, and log the anomaly.

9. **Reconcile periodically.** Sweep entities against the API on a sliding window (skip
   terminal-state entities) to catch anything the event stream missed.

10. **Manage subscriptions.** `list_webhooks` (`GET /webhooks`) to audit,
    `delete_webhook` (`DELETE /webhooks/{webhook_id}`) to remove.

11. **Backfill with polling.** `list_events` (`GET /events`) returns the same payload format
    and is the recovery path after an outage in your receiver.

## Failure handling

- **Transient** (network, timeouts, 5XX, contention): retry with exponential backoff
  (1m, 2m, 4m …).
- **Permanent** (invalid data, logic errors): do not retry — dead-letter and alert.
- **Dead letter queue** must preserve the full payload, the error, and the retry count.
  Anything in the DLQ is a known gap in your mirror.
- **Monitor**: processing lag, failure rate, DLQ growth.

## Event families

`account.*` · `securities_account.*` · `person`/`identification.*` · `check.*` · `order.*` ·
`batch_order.*` · `trade.*` · `deposit.*` · `withdrawal.*` · `sweep.*` · `settlement.*` ·
`transaction` · `corporate_action.*` · `dividend_distribution.*` · `income_distribution.*` ·
`tax.*` · `tax_exemption_order.*` · `treasury_mandate.*` · `treasury_transfer.*` ·
`workflow.*` · `document.created` · `update.*` · `cash_account.created`

The full enumerated list is in `asyncapi/lemonmarkets-brokerage-webhooks.yml`. lemon.markets
states the list is not final — **ignore unknown event types gracefully** rather than erroring.

## Rules

- **The REST API is the source of truth, always.** If the mirror and the API disagree, the API
  wins.
- **Never make decisions from the `context` structure** — it may evolve. Use `type` + a re-fetch.
- **Never drop events**, even ones you do not currently handle. Store them; you will want the
  audit trail.
