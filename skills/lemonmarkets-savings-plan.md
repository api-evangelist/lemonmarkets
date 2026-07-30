---
name: Set up a recurring savings plan
description: Create, confirm, monitor and cancel a recurring-investment workflow (Sparplan) on the lemon.markets Brokerage API.
api: openapi/lemonmarkets-brokerage-openapi.json
generated: '2026-07-19'
method: generated
source: openapi/lemonmarkets-brokerage-openapi.json + https://developer.lemon.markets/docs/recurring-investments
operations:
  - get_instrument
  - create_workflow
  - confirm_workflow
  - get_workflow
  - list_workflows
  - cancel_workflow
  - list_orders
  - get_positions
---

# Set up a recurring savings plan

Savings plans (*Sparpläne*) are modelled as **workflows** — a standing instruction that
generates orders on a schedule. This is the highest-volume retail flow in German brokerage,
and the one most partners build first.

## Steps

1. **Validate the instrument.** Call `get_instrument` (`GET /instruments/{isin}`) and confirm
   it is savings-plan eligible and not halted before you let a customer commit to a schedule.

2. **Create the workflow.** Call `create_workflow`
   (`POST /accounts/{account_id}/workflows`) with the instrument, the contribution amount and
   the recurrence schedule. Capture the returned workflow id (`wf_…`).

3. **Confirm it.** Call `confirm_workflow`
   (`POST /accounts/{account_id}/workflows/{workflow_id}/confirm`). As with orders, creation
   alone does not arm the plan — an unconfirmed workflow will not execute.

4. **Verify.** Call `get_workflow` (`GET /accounts/{account_id}/workflows/{workflow_id}`) and
   surface the schedule and next execution to the customer. Use `list_workflows`
   (`GET /accounts/{account_id}/workflows`) to render all of an account's plans.

5. **Track executions.** Each cycle produces an order. Watch `order.*` events and use
   `list_orders` (`GET /accounts/{account_id}/orders`) to attribute orders back to the plan,
   then `get_positions` to show accumulated holdings.

6. **Cancel.** Call `cancel_workflow`
   (`POST /accounts/{account_id}/workflows/{workflow_id}/cancel`) and wait for
   `workflow.canceled` before telling the customer the plan has stopped.

## Events

`workflow.created`, `workflow.confirmed`, `workflow.canceled` — plus the `order.*` and
`trade.*` events for every execution the plan generates. Re-fetch state on each; do not infer
plan health from order events alone.

## Rules

- **Confirm is mandatory.** A created-but-unconfirmed workflow silently never runs. This is the
  single most common integration bug in this flow.
- **Sufficient funds are the partner's problem.** A cycle with insufficient cash produces a
  rejected order (`order.rejected`), not a workflow error. Monitor order rejections and tell
  the customer — the plan stays armed.
- **Cancellation is asynchronous.** Treat `cancel_workflow` as a request; the plan is stopped
  when `workflow.canceled` arrives.
- **`create_workflow` does not accept `Idempotency-Key`** — only `create_order` does today. Guard
  against double submission on your side (dedupe on your own request id before calling).
- **Errors** are `{"message": "..."}` with no error code; branch on HTTP status.
