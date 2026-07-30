---
name: Place, confirm and reconcile an order
description: Look up an instrument, place a buy or sell order idempotently, confirm it, and reconcile the resulting trade and positions on the lemon.markets Brokerage API.
api: openapi/lemonmarkets-brokerage-openapi.json
generated: '2026-07-19'
method: generated
source: openapi/lemonmarkets-brokerage-openapi.json + https://developer.lemon.markets/docs/idempotency
operations:
  - list_instruments
  - get_instrument
  - get_prices
  - get_financials_accounts__account_id__financials_get
  - create_order
  - confirm_order
  - get_order
  - list_orders
  - cancel_order
  - list_trades
  - get_positions
---

# Place, confirm and reconcile an order

This is the money path. Orders move real customer assets, so the idempotency and
reconciliation rules below are not optional.

## Before you start

- **Auth**: `Authorization: Bearer <your-api-key>`.
- **Data-privacy headers** are required: `LMG-Data-Privacy-Access-Principal` and
  `LMG-Data-Privacy-Access-Justification`.
- Amounts are **decimal strings** (`"100.00"`), currency is ISO 4217 and **EUR is the only
  supported currency**. Instruments are addressed by **ISIN**.

## Steps

1. **Find the instrument.** Call `list_instruments` (`GET /instruments`) to search, or
   `get_instrument` (`GET /instruments/{isin}`) when you already have the ISIN. Check for an
   active **trading halt** before offering the instrument — a halted instrument will reject
   the order with 422.

2. **Get a price** (optional, for display). Call `get_prices`
   (`GET /instruments/{isin}/prices`).

3. **Check buying power.** Call `get_financials_accounts__account_id__financials_get`
   (`GET /accounts/{account_id}/financials`) for the cash position before a buy, or
   `get_positions` (`GET /accounts/{account_id}/positions`) to confirm the holding before a sell.

4. **Generate an idempotency key.** One fresh UUID per logical order attempt:

   ```shell
   uuidgen | tr '[:upper:]' '[:lower:]'
   ```

   Persist it alongside your local order record **before** making the call. If your process
   dies mid-request, the stored key is what lets you recover safely.

5. **Place the order.** Call `create_order` (`POST /accounts/{account_id}/orders`) with the
   `Idempotency-Key` header:

   ```shell
   curl --request POST \
        --url https://sandbox.api.lemon.markets/v1/accounts/{account_id}/orders \
        --header 'authorization: Bearer <your-api-key>' \
        --header 'content-type: application/json' \
        --header 'Idempotency-Key: <uuid>' \
        --header 'LMG-Data-Privacy-Access-Principal: <principal>' \
        --header 'LMG-Data-Privacy-Access-Justification: <justification>' \
        --data '{"side":"buy","type":"market","instrument":"US0378331005","amount":"100.00"}'
   ```

   A `201` returns the order (`ord_…`) with a `history[]` of status transitions.

6. **Confirm the order.** Call `confirm_order`
   (`POST /accounts/{account_id}/orders/{order_id}/confirm`). Creation alone does not send the
   order to market — an unconfirmed order will not execute.

7. **Track to execution.** Subscribe to `order.created`, `order.accepted`, `order.confirmed`,
   `order.executed`, `order.rejected`, `order.canceling`, `order.canceled`. On each event,
   re-fetch with `get_order` (`GET /accounts/{account_id}/orders/{order_id}`) — the API is the
   source of truth.

8. **Reconcile the trade.** On `order.executed`, call `list_trades`
   (`GET /accounts/{account_id}/trades`) and re-read `get_positions`. Settlement and tax follow
   asynchronously via `trade.cash_settled` and `trade.taxes_processed`.

9. **Cancel if needed.** Call `cancel_order`
   (`POST /accounts/{account_id}/orders/{order_id}/cancel`). Cancellation is a request, not a
   guarantee — wait for `order.canceled` before telling the customer it is cancelled.

## Idempotency rules (read these)

`create_order` is the only operation that currently accepts `Idempotency-Key`.

| Situation | What to do |
|---|---|
| Timeout / no response / 5XX | **Retry with the exact same key and same payload.** You get the cached `201`, never a duplicate order. |
| You want to change the order | **Use a new key.** Reusing a key with a different payload returns `422`. |
| Original call returned 400/422 | The key was **not consumed** — fix the payload and reuse the same key. |
| More than 24h has passed | The key has expired; the cached response is gone. |
| Key longer than 100 chars | `422`. Keep to a UUID. |

**Never retry a create_order without the original key.** That is how you double a customer's
trade.

## Rules

- **Confirm is a separate step.** Create → confirm → executed. Do not report success at create.
- **Errors** are `{"message": "..."}` with no code. `422` on order creation typically means a
  business rejection: trading halt, insufficient funds, or an idempotency-key conflict.
- **Never skip or dedupe away events** — `order.executed` may arrive before `order.accepted`.
  Re-fetch state rather than inferring it from arrival order.
- **Sandbox halt testing**: `XF0000000046` (buy halt), `XF0000000053` (sell halt),
  `XF0000000061` (both). Orders against these return 422 in sandbox. These ISINs resolve on
  `get_instrument` only and never appear in `list_instruments`.
