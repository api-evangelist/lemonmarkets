---
name: Place and track an order on lemon.markets
description: Look up an instrument and its price, place a securities order idempotently, confirm it, and follow it through to an executed trade and a settled position.
api: openapi/lemon-markets-brokerage-openapi.json
base_url: https://sandbox.api.lemon.markets/v1
operations:
  - list_instruments
  - get_instrument
  - get_prices
  - create_order
  - confirm_order
  - get_order
  - list_orders
  - cancel_order
  - list_trades
  - get_trade
  - get_positions
  - get_financials_accounts__account_id__financials_get
generated: '2026-07-19'
method: generated
---

# Place and track an order

Places a securities order for an already-opened account and follows it to a trade. This flow moves
real money and real securities - treat every write step as consequential and never retry blindly.

## Before you start

- Bearer API key on `Authorization`, plus `LMG-Data-Privacy-Access-Principal` and
  `LMG-Data-Privacy-Access-Justification` on every call.
- The account must be open (`account.opened`) and have a securities account
  (`securities_account.opened`).
- Prices are strings with 4 decimal places, transaction amounts 2, quantities up to 5. Use a decimal
  type.

## Steps

1. **Find the instrument.** `list_instruments` (`GET /instruments`) searches the tradeable universe;
   `get_instrument` (`GET /instruments/{isin}`) returns one instrument by ISIN (ISO 6166). Check the
   instrument for an active **trading halt** before ordering - an order against a halted side is
   rejected with `422`.

2. **Check the price.** `get_prices` (`GET /instruments/{isin}/prices`) returns current pricing. For
   distributing funds, `get_latest_yield` (`GET /instruments/{instrument}/yield`) gives the latest
   monthly annualised yield.

3. **Place the order idempotently.** `create_order`
   (`POST /accounts/{account_id}/orders`) with the ISIN, side, and either a quantity or an amount.
   **Always send an `Idempotency-Key` header** - a UUID, maximum 100 characters. This is the one
   endpoint on the API that supports it, and it is exactly the endpoint where a duplicate is
   expensive. On a network failure, retry with the *same* key and the *same* body: you get the
   original `201` back rather than a second order. Change the body and reuse the key and you get
   `422`. Keys expire after 24 hours. See `https://developer.lemon.markets/docs/idempotency`.

4. **Confirm the order.** `confirm_order`
   (`POST /accounts/{account_id}/orders/{order_id}/confirm`). Creating an order stages it; confirming
   it releases it to the venue. An order left unconfirmed does not execute.

5. **Track it.** `get_order` (`GET /accounts/{account_id}/orders/{order_id}`) reads current state;
   `list_orders` enumerates with cursor pagination. Prefer the webhook events over polling:
   `order.created` -> `order.confirmed` -> `order.accepted` -> `order.executed`, with `order.rejected`
   and `order.canceling`/`order.canceled` as terminal alternatives.

6. **Cancel if needed.** `cancel_order`
   (`POST /accounts/{account_id}/orders/{order_id}/cancel`). Cancellation is a request, not a
   guarantee - you will see `order.canceling` first and only then `order.canceled`. An order that
   executed in the meantime cannot be cancelled.

7. **Read the trade.** Once `order.executed` fires, `list_trades`
   (`GET /accounts/{account_id}/trades`) and `get_trade` return the execution: `price`, `quantity`,
   `amount`, `fee`, `currency` and `execution_venue` (an ISO 10383 MIC). `trade.cash_settled` and
   `trade.taxes_processed` follow as settlement and German withholding tax complete.

8. **Reconcile.** `get_positions` (`GET /accounts/{account_id}/positions`) for holdings and
   `get_financials_accounts__account_id__financials_get`
   (`GET /accounts/{account_id}/financials`) for cash and buying power. `get_transactions` gives the
   ledger.

## Batching

For many orders at once use the batch surface instead: `create_batch_order`, `confirm_batch_order`,
`get_batch_order`, `list_batch_orders`, `cancel_batch_order` - same create/confirm/track shape, with
`batch_order.*` events. Note the `Idempotency-Key` header is documented only on `create_order`, not
on `create_batch_order`.

## Pagination

List endpoints are cursor-paginated: read `pagination.next_cursor` from the response and pass it back
as the `cursor` query parameter. Absence of `next_cursor` means you are done.

## Related

- `skills/lemon-markets-consume-webhook-events.md`
- `sandbox/lemon-markets-sandbox.yml` - test ISINs that reproduce trading halts
- `errors/lemon-markets-problem-types.yml`
