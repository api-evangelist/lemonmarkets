---
name: Consume lemon.markets webhook events
description: Register a webhook, verify the LMG-Signature HMAC, process the 83 event types idempotently and in per-entity order, and reconcile gaps against the events endpoint.
api: openapi/lemon-markets-brokerage-openapi.json
base_url: https://sandbox.api.lemon.markets/v1
operations:
  - create_webhook
  - list_webhooks
  - delete_webhook
  - list_events
generated: '2026-07-19'
method: generated
---

# Consume webhook events

The Brokerage API is asynchronous. Almost every meaningful state change - an account opening, an
order executing, tax being processed - arrives as an event, not as the response to your call. Getting
this loop right is the difference between a correct integration and a guessing one.

## Register

1. `create_webhook` (`POST /webhooks`) with `{"url": "...", "events": ["account.created", ...]}`.
   `events` must be a non-empty array of unique `EventType` values; the full 83-value enum is in
   `asyncapi/lemon-markets-brokerage-webhooks.yml`.
2. The response returns the webhook `id` (`wbh_` prefix) and a `signature_secret`. **Store the secret
   securely - it is shown on creation and is what you verify deliveries with.**
3. `list_webhooks` (`GET /webhooks`) enumerates registrations; `delete_webhook`
   (`DELETE /webhooks/{webhook_id}`) removes one.

## Verify every delivery

Each delivery carries an `LMG-Signature` header:

```
t=1688367562,v1=22db5f658011a57a8fb1e766755716a74b95973e34dc3a8e0aa1c35b7debd4be
```

Split on `,`. `t=` is Unix seconds; `v1=` is a hex HMAC-SHA256 computed over the timestamp, a literal
`.`, and the **raw** request body, keyed with the `signature_secret`:

```
mac = HMAC<SHA256>(key: signature_secret)
mac.update(timestamp)      // without the "t=" prefix
mac.update(".")
mac.update(rawRequestBody) // bytes as received, before any JSON parsing
verifier = "v1=" + hex(mac.finalize())
```

Compare in constant time. Also sanity-check `t` against your system clock within a tolerance window -
that is what stops replay attacks. Reject anything that fails either check.

Test vector for your unit tests, published by lemon.markets:

```
signature      = "t=0,v1=3523dcc0013f08dfa1855772441107330218793f399d7452bd3ff2159c6e0285"
signing_secret = "0000000000000000000000000000000000000000000000000000000000000000"
request_body   = "{}"
```

## Process

The payload is an `EventExternal`:

```json
{
  "id": "evt_a8b11a37eccc49ecacb372e3a92a184d",
  "type": "account.created",
  "created_at": "2023-06-29T14:40:30.523373+00:00",
  "context": { "account": "cusa_33531f7f801d4ce997747470ab7208fd" }
}
```

`context` is polymorphic - which identifiers it carries depends on the event type (account, order,
trade, transaction, workflow, and so on).

Four rules, all from the provider's own guidance:

1. **Acknowledge immediately.** Return `200 OK` in under 10 seconds. Validate the signature, enqueue
   the event, return. Never process synchronously in the handler, or you will be retried.
2. **Process idempotently.** Delivery is at-least-once. Dedupe on the event `id`, store processed ids,
   and make database writes conditional on current state.
3. **Preserve per-entity order.** Events for the same entity must be applied in order - partition your
   queue on the entity id from `context`. Different entities can go in parallel. Use `created_at` to
   order within a partition.
4. **Reconcile.** Run a periodic sweep against `list_events` (`GET /events`) - the same payload shape -
   to catch anything your listener missed, and to repair drift in a mirrored datastore.

## Event families

23 entity families across 83 types: `account.*`, `securities_account.*`, `cash_account.*`, `check.*`,
`identification.*`, `document.*`, `order.*`, `batch_order.*`, `trade.*`, `deposit.*`, `withdrawal.*`,
`sweep.*`, `tax.*`, `tax_exemption_order.*`, `corporate_action.*`, `dividend_distribution.*`,
`income_distribution.*`, `securities_transfer.*`, `settlement.*`, `treasury_mandate.*`,
`treasury_transfer.*`, `workflow.*`, `update.*`. The full list with per-family grouping is in
`asyncapi/lemon-markets-brokerage-webhooks.yml`.

## Related

- `https://developer.lemon.markets/docs/mirror-data-using-events` - building a shadow copy
- `conventions/lemon-markets-conventions.yml`
