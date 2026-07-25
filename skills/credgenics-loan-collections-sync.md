---
name: Sync a loan and record a payment in Credgenics
description: Authenticate, upload or update a loan into the Credgenics recovery platform, retrieve its collections status, and record a payment with recovery-amount bifurcation.
api: openapi/credgenics-recovery-openapi.yml
operations: [createAccessToken, uploadLoan, getLoan, updatePayment]
---

# Sync a loan and record a payment

Use this skill to push a lender's loan into Credgenics for collections and to
report repayments back. All calls are JSON over HTTPS against
`https://apiprod.credgenics.com`.

## Auth (do this first)
1. Call **createAccessToken** — `POST /user/public/access-token` with body
   `{ "client_id", "client_secret", "token_expiry_duration" }`
   (`token_expiry_duration` optional, seconds, default 900, max 86400).
2. Read `api_key` and the `company_details[]` from the response. The client
   secret is valid 6 months; the token expires quickly, so cache it and refresh
   before expiry.
3. On every subsequent call send header `authenticationtoken: <api_key>` and
   the query parameter `company_id=<company_id>`.

## Steps
1. **uploadLoan** — `POST /recovery/v1/loan/{loan_id}?company_id=...` to create
   the loan / EMI record. A `201` means created.
2. **getLoan** — `GET /recovery/v1/loans/{loan_id}?company_id=...&attribute_levels=...`
   to read applicant, loan_variables, allocation_variables (outstanding, DPD,
   penalties), statuses, payments and tags. Add `data_type_conversion=true` for
   typed values and `audit_info=true` for created/updated metadata.
3. **updatePayment** — `PATCH /recovery/payments/{loan_id}?company_id=...&allocation_month=YYYY-M-01`
   to record a repayment. When recovery-amount bifurcation is enabled, split the
   amount using prefixed keys: `recovered_expected_emi`, `recovered_late_fee`,
   `recovered_penalty`.

## Rules
- **Tenancy:** `company_id` is required on every request — never omit it.
- **No idempotency key** is documented; do not blind-retry a `POST`. On a
  timeout, `getLoan` first to check whether the write landed before retrying.
- **Errors** use a custom envelope: `{ "message", "success": false, "output": { "errors": {...} } }`.
  A `401` ("Session expired, please login again") means re-run createAccessToken.
  See `errors/credgenics-problem-types.yml`.
- Conventions: `conventions/credgenics-conventions.yml`.
