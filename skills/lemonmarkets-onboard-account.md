---
name: Onboard a customer and open a securities account
description: Take a new retail customer from agreement acceptance through KYC identity verification to an open, tradable securities account on the lemon.markets Brokerage API.
api: openapi/lemonmarkets-brokerage-openapi.json
generated: '2026-07-19'
method: generated
source: openapi/lemonmarkets-brokerage-openapi.json + https://developer.lemon.markets/docs/create-account
operations:
  - get_agreements
  - create_account
  - accept_agreement
  - get_account
  - get_account_checks
  - set_experience_profile
  - update_knowledge
  - start_identity_verification
  - get_person_identification
  - submit_identification_payload
  - create_securities_account
---

# Onboard a customer and open a securities account

Use this skill to bring a natural person from zero to a securities account that can place
orders. lemon.markets is a BaFin-licensed investment firm, so this flow is a regulated
onboarding: agreements, suitability and identity verification are mandatory gates, not
optional steps.

## Before you start

- **Base URL**: `https://sandbox.api.lemon.markets/v1` for testing,
  `https://api.lemon.markets/v1` for production. Keys are not interchangeable.
- **Auth**: `Authorization: Bearer <your-api-key>` on every request.
- **Required on nearly every call**: both data-privacy headers.
  - `LMG-Data-Privacy-Access-Principal` — who the access is on behalf of.
  - `LMG-Data-Privacy-Access-Justification` — why. These are auditable; send truthful values.
- **Access is invite-only.** If you have no key, there is no self-service signup — go through
  https://www.lemon.markets/en-de/contact.

## Steps

1. **Fetch the current agreements.** Call `get_agreements` (`GET /agreements`) to get the
   latest contractual documents the customer must accept. Never hard-code agreement IDs —
   they change, and accepting a stale one will fail.

2. **Create the account.** Call `create_account` (`POST /accounts`) with the customer block
   (name, date/place of birth, email, phone, nationalities, registered address, employment,
   marital status, tax residencies), `declaration_of_acting_on_own_account`, and
   `accepted_agreements[]` carrying each `agreement_id` from step 1 with its `accepted_at`
   timestamp. Capture the returned account id (`cusa_…`).
   - Response now includes an `agreements` field listing all valid signed agreements
     (added 2026-07-13).
   - Read the customer's person id from the `person` field, not the deprecated `id` field.

3. **Record the suitability profile.** Call `set_experience_profile`
   (`POST /accounts/{account_id}/experience`) to record the customer's investment experience
   and knowledge. Use `update_knowledge` (`PATCH /accounts/{account_id}/experience`) to amend
   knowledge later. This is a MiFID suitability requirement — an account will not become
   tradable without it.

4. **Start identity verification.** Call `start_identity_verification`
   (`POST /accounts/{account_id}/identity_verification`). This begins the KYC process and
   returns an identification the customer must complete.

5. **Track and complete identification.** Poll `get_person_identification`
   (`GET /persons/{person_id}/identifications/{identification_id}`) or — better — subscribe to
   the `identification.*` events. Where your flow submits the identification payload directly,
   call `submit_identification_payload`
   (`POST /persons/{person_id}/identifications/{identification_id}/submit`).

6. **Watch the checks.** Call `get_account_checks` (`GET /accounts/{account_id}/checks`) to see
   which compliance checks are outstanding. Mirror `check.action_required`, `check.pending`,
   `check.processing`, `check.succeeded` and `check.failed` events into your own UI so the
   customer knows what is blocking them.

7. **Open the securities account.** Once the account reaches an opened state, call
   `create_securities_account` (`POST /accounts/{account_id}/securities_accounts`). The account
   can now hold positions and place orders.

8. **Handle later agreement updates.** When lemon.markets publishes new agreements for an
   already-provisioned account, call `accept_agreement`
   (`POST /accounts/{account}/accept_agreement`) with the person id, agreement id and
   acceptance timestamp — do not try to recreate the account.

## Events to subscribe to

Register a webhook (see the `consume-events` skill) for:
`account.created`, `account.opened`, `account.rejected`, `account.closed`,
`identification.created`, `identification.started`, `identification.succeeded`,
`identification.failed`, `check.*`, `securities_account.created`, `securities_account.opened`,
`securities_account.rejected`.

Treat every event as a trigger to re-fetch with `get_account` or `get_account_checks` — the
event payload is a notification, not state.

## Rules

- **Never branch on the event `context` structure.** It may change. Branch on `type`, then
  re-fetch from the API.
- **Onboarding is asynchronous.** `account.created` is not `account.opened`. Do not let a
  customer attempt to trade until the securities account exists and checks have succeeded.
- **Deprecated fields**: use `person` / `business` instead of `id` on customer responses.
- **Errors** are `{"message": "..."}` with no error code. Branch on HTTP status:
  400 malformed, 401 bad key, 404 wrong id or wrong tenant, 422 business-rule rejection.
- **Do not retry a 422 unchanged** — it is a semantic rejection; read `message` and fix the input.
