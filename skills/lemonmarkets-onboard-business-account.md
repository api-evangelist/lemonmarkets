---
name: Onboard a business account on lemon.markets
description: Open a corporate brokerage account - create the business, register legal representatives and beneficial owners, submit corporate documents, and open the securities account.
api: openapi/lemon-markets-brokerage-openapi.json
base_url: https://sandbox.api.lemon.markets/v1
operations:
  - create_business
  - get_business
  - list_businesses
  - add_legal_representative
  - list_legal_representatives
  - add_beneficial_owner
  - list_beneficial_owners
  - submit_document_upload
  - create_person
  - get_person
  - create_account
  - get_account_checks
  - create_securities_account
generated: '2026-07-19'
method: generated
---

# Onboard a business account

Corporate onboarding is heavier than retail because KYB requires the legal entity, the people who can
act for it, and the people who ultimately own it - each evidenced by documents.

## Steps

1. **Create the business.** `create_business` (`POST /businesses`) with the legal entity details.
   Read it back with `get_business` (`GET /businesses/{business_id}`); `list_businesses` enumerates.

2. **Register the people.** Legal representatives and beneficial owners are `person` records, so
   create them with `create_person` (`POST /persons`) where they do not already exist, then:
   - `add_legal_representative` (`POST /businesses/{business_id}/legal_representatives`) for those
     authorised to act for the entity;
   - `add_beneficial_owner` (`POST /businesses/{business_id}/beneficial_owners`) for the ultimate
     beneficial owners.
   Verify with `list_legal_representatives` and `list_beneficial_owners`.

3. **Submit corporate documents.** Use `submit_document_upload` (`POST /submit_document`) - the
   unified endpoint introduced 2026-06-24. It takes a `type` discriminator selecting a person or
   business document and covers person identification bundles, articles of association, commercial
   register extracts and shareholder lists. It returns a **presigned `upload_location` URL that
   expires**; PUT the file bytes to that URL promptly, then treat the upload as complete.

   > Do **not** use `submit_document` (`POST /businesses/{business_id}/submit_document`). It is
   > deprecated as of 2026-06-24 and scheduled for removal.

4. **Create the account.** `create_account` (`POST /accounts`) for the business customer, with the
   accepted agreements exactly as in retail onboarding (`get_agreements` first).

5. **Watch the checks.** `get_account_checks` (`GET /accounts/{account_id}/checks`) plus the
   `check.*` events. Corporate checks routinely land on `check.action_required` - that means a
   document or data point is missing, not that onboarding failed. Resolve and resubmit.

6. **Open the securities account.** `create_securities_account`
   (`POST /accounts/{account_id}/securities_accounts`) once the account reaches `account.opened`.

## Field deprecation to respect

`PersonCustomerResponse.id` and `BusinessCustomerResponse.id` are deprecated as of 2026-07-07. Read
`person` for person customers and `business` for business customers instead - the `id` field will be
removed.

## BPO accounts

lemon.markets also supports a BPO (business process outsourcing) operating model, documented at
`https://developer.lemon.markets/docs/create-bpo-account`. It uses the same account and securities
account operations with a different customer configuration.

## Related

- `https://developer.lemon.markets/docs/create-business-account`
- `lifecycle/lemon-markets-lifecycle.yml` - the deprecation register
