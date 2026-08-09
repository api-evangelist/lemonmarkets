---
name: Run savings plans with lemon.markets workflows
description: Set up recurring investment schemes (savings plans) as workflows, confirm and cancel them, and follow the orders they generate.
api: openapi/lemon-markets-brokerage-openapi.json
base_url: https://sandbox.api.lemon.markets/v1
operations:
  - create_workflow
  - confirm_workflow
  - get_workflow
  - list_workflows
  - cancel_workflow
  - list_orders
  - get_order
  - list_trades
generated: '2026-07-19'
method: generated
---

# Run savings plans with workflows

Workflows are how lemon.markets models recurring investment schemes - the German *Sparplan*. Rather
than your system placing an order on a cron, you register the intent once and the platform generates
the orders on schedule.

## Steps

1. **Create the workflow.** `create_workflow` (`POST /accounts/{account_id}/workflows`) describing the
   recurring investment: the instrument, the amount per execution, and the schedule. The response
   carries a workflow identifier and emits `workflow.created`.

2. **Confirm it.** `confirm_workflow`
   (`POST /accounts/{account_id}/workflows/{workflow_id}/confirm`). As with orders, creation stages
   the workflow and confirmation activates it. `workflow.confirmed` follows. An unconfirmed workflow
   generates nothing.

3. **Inspect.** `get_workflow` (`GET /accounts/{account_id}/workflows/{workflow_id}`) for one,
   `list_workflows` for the account's set (cursor-paginated).

4. **Follow the generated orders.** Each execution produces a normal order, so it flows through the
   ordinary `order.created` -> `order.confirmed` -> `order.accepted` -> `order.executed` lifecycle and
   is visible through `list_orders` / `get_order`, with the resulting fills in `list_trades`. Events
   whose context carries both an `order` and a `workflow` identifier
   (`OrderWorkflowEventContext`) are the ones your workflow generated - use that to attribute
   executions back to the savings plan.

5. **Cancel.** `cancel_workflow`
   (`POST /accounts/{account_id}/workflows/{workflow_id}/cancel`) stops future executions and emits
   `workflow.canceled`. It does not unwind orders already placed.

## Notes

- Amounts are strings holding fixed-point decimals (2 decimal places for the investment amount).
- Ensure the account has cash before an execution date - a savings plan execution that cannot be
  funded will fail like any other order. `get_financials_accounts__account_id__financials_get` reads
  available cash, and `list_sweeps` shows balance sweeps in flight.
- Document events with a workflow in context (`CustomerWorkflowDocumentEventContext`) signal
  contract notes generated for the plan; retrieve them via `list_documents_of_account` /
  `get_document`.

## Related

- `https://developer.lemon.markets/docs/recurring-investments`
- `skills/lemon-markets-place-and-track-an-order.md`
