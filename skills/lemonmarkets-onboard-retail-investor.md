---
name: Onboard a retail investor on lemon.markets
description: Open a person brokerage account end to end - agreements, account creation, identity verification, knowledge and experience profile, and a securities account ready to trade.
api: openapi/lemon-markets-brokerage-openapi.json
base_url: https://sandbox.api.lemon.markets/v1
operations:
  - get_agreements
  - create_account
  - get_account
  - get_account_checks
  - start_identity_verification
  - get_person_identification
  - submit_identification_payload
  - set_experience_profile
  - get_experience_profile
  - create_securities_account
  - list_securities_accounts
generated: '2026-07-19'
method: generated
---

# Onboard a retail investor

Opens a natural-person brokerage account on the lemon.markets Brokerage API and leaves it in a state
where orders can be placed. Every step is asynchronous behind the scenes: the API accepts your input,
then emits events as checks complete. Never assume an account is tradeable because a `201` came back.

## Before you start

- Authenticate every request with `Authorization: Bearer <api-key>`.
- Every customer-data request **must** also carry `LMG-Data-Privacy-Access-Principal` (who is acting)
  and `LMG-Data-Privacy-Access-Justification` (why). These are required parameters, not optional
  hygiene - omitting them fails the request.
- Money is a **string** holding a fixed-point decimal (2dp for transaction values, 4dp for prices,
  5dp for quantities). Parse into a decimal type, never a float. See
  `conventions/lemon-markets-conventions.yml`.
- The API is invite-only and only the sandbox host is published:
  `https://sandbox.api.lemon.markets/v1`.

## Steps

1. **Fetch the current agreements.** Call `get_agreements` (`GET /agreements`). It returns the
   contractual documents the customer must accept - terms, privacy policy and so on. You need their
   `agreement_id` values and the timestamp at which the customer accepted each one.

2. **Create the account.** Call `create_account` (`POST /accounts`) with the customer block (name,
   date and place of birth, nationalities, registered address, tax residencies, employment,
   marital status, email, phone), `declaration_of_acting_on_own_account`, and
   `accepted_agreements[]` carrying each `agreement_id` plus its `accepted_at`. The response carries
   the account identifier (`cusa_` prefix) and the `agreements` currently on file.

3. **Watch the checks.** Call `get_account_checks` (`GET /accounts/{account_id}/checks`) to see the
   compliance checks running against the account. Do not poll tightly - subscribe to the
   `check.processing`, `check.action_required`, `check.pending`, `check.succeeded` and `check.failed`
   webhook events instead (see the webhooks skill).

4. **Run identity verification.** Call `start_identity_verification`
   (`POST /accounts/{account_id}/identity_verification`) to begin ID+V for the person. Track it with
   `get_person_identification`
   (`GET /persons/{person_id}/identifications/{identification_id}`), and where the flow requires you
   to hand back captured data, call `submit_identification_payload`
   (`POST /persons/{person_id}/identifications/{identification_id}/submit`). The
   `identification.created`, `identification.started`, `identification.succeeded` and
   `identification.failed` events tell you where the customer is without polling.

5. **Record knowledge and experience.** Call `set_experience_profile`
   (`POST /accounts/{account_id}/experience`) with the customer's investment knowledge and experience -
   this is a regulatory suitability requirement, not an optional profile field. Use `update_knowledge`
   (`PATCH /accounts/{account_id}/experience`) to amend it later and `get_experience_profile` to read
   it back.

6. **Create the securities account.** Call `create_securities_account`
   (`POST /accounts/{account_id}/securities_accounts`). This is the depot that will hold positions.
   Confirm with `list_securities_accounts` or `get_securities_account`.

7. **Wait for the account to open.** The account is only tradeable once you have seen
   `account.opened` and `securities_account.opened`. `account.rejected` or
   `securities_account.rejected` mean onboarding failed - read `get_account` and
   `get_account_checks` for the reason. Do not place orders before `account.opened`.

## Errors

Errors return `application/json` shaped as `{"message": "..."}` - there is no machine-readable error
code, so branch on the HTTP status and log `message`. See `errors/lemon-markets-problem-types.yml`.
A `422` on onboarding calls generally means the payload is regulatorily incomplete rather than
malformed.

## Related

- `skills/lemon-markets-place-and-track-an-order.md`
- `skills/lemon-markets-consume-webhook-events.md`
- `conventions/lemon-markets-conventions.yml`
