# API Test Cases — NexaPay Payment Gateway

**Platform under test:** NexaPay — a fictional merchant-acquiring / payment gateway platform (invented for this portfolio) that merchants integrate with to accept card and digital-wallet payments, including tokenized recurring charges, refunds, and settlement reporting. NexaPay is not a real product and does not represent any employer's actual system.

**Total test cases:** 28
**Columns:** TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type

## Authorization & Capture

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-AUTH-001 | POST /v1/payments | Create a payment authorization with a valid card | Merchant has an active API key with `payments:write` scope. | amount=5000, currency=AED, card=4111111111111111, exp=12/2027, cvv=123 | 201 Created; response includes `payment_id`, `status=AUTHORIZED`, and masked card (`**** 1111`); no raw PAN in response. | P1 | Positive |
| API-AUTH-002 | POST /v1/payments | Authorization fails gracefully for a declined test card | Sandbox decline-trigger card is configured. | card=4000000000000002 (decline trigger), amount=2500 | 200/402 with `status=DECLINED` and a `decline_reason` code; no payment record is left in an ambiguous state. | P1 | Negative |
| API-AUTH-003 | POST /v1/payments/{id}/capture | Full capture of a previously authorized payment | Payment exists with `status=AUTHORIZED`, amount=5000. | capture amount=5000 (full) | 200 OK; `status=CAPTURED`; captured_amount equals authorized amount; funds move to settlement pipeline. | P1 | Positive |
| API-AUTH-004 | POST /v1/payments/{id}/capture | Partial capture of an authorized payment | Payment exists with `status=AUTHORIZED`, amount=5000. | capture amount=2000 (partial) | 200 OK; `status=PARTIALLY_CAPTURED`; remaining authorized balance (3000) is released or held per policy; response reflects `captured_amount=2000`. | P1 | Positive |
| API-AUTH-005 | POST /v1/payments/{id}/capture | Capture amount exceeding the authorized amount is rejected | Payment exists with `status=AUTHORIZED`, amount=5000. | capture amount=6000 | 422 Unprocessable Entity; error code `AMOUNT_EXCEEDS_AUTHORIZATION`; no capture is recorded. | P1 | Negative |
| API-AUTH-006 | POST /v1/payments/{id}/capture | Capturing an already-captured payment is rejected | Payment exists with `status=CAPTURED`. | capture amount=5000 | 409 Conflict; error code `INVALID_STATE_TRANSITION`; original capture record is unchanged. | P2 | Negative |
| API-AUTH-007 | POST /v1/payments/{id}/void | Void an authorized (uncaptured) payment | Payment exists with `status=AUTHORIZED`. | void reason="merchant_cancelled" | 200 OK; `status=VOIDED`; held funds are released back to the cardholder's available balance. | P2 | Positive |
| API-AUTH-008 | POST /v1/payments/{id}/void | Void is rejected once a payment has been captured | Payment exists with `status=CAPTURED`. | void reason="merchant_cancelled" | 409 Conflict; error message directs the caller to use `/refund` instead; no state change. | P2 | Negative |

## Refunds

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-REF-001 | POST /v1/payments/{id}/refund | Full refund of a captured payment | Payment exists with `status=CAPTURED`, amount=5000. | refund amount=5000 (full) | 200 OK; `status=REFUNDED`; a `refund_id` is returned; refunded_amount equals captured amount. | P1 | Positive |
| API-REF-002 | POST /v1/payments/{id}/refund | Partial refund of a captured payment | Payment exists with `status=CAPTURED`, amount=5000. | refund amount=1500 (partial) | 200 OK; `status=PARTIALLY_REFUNDED`; refundable balance decreases to 3500; multiple partial refunds are allowed up to the captured total. | P1 | Positive |
| API-REF-003 | POST /v1/payments/{id}/refund | Refund exceeding the remaining refundable balance is rejected | Payment previously partially refunded 3500 of 5000. | refund amount=2000 (only 1500 remains refundable) | 422 Unprocessable Entity; error code `REFUND_EXCEEDS_BALANCE`; no refund record created. | P1 | Boundary |
| API-REF-004 | POST /v1/payments/{id}/refund | Refund on a payment with no capture (AUTHORIZED only) is rejected | Payment exists with `status=AUTHORIZED`. | refund amount=5000 | 409 Conflict; error code `INVALID_STATE_TRANSITION`; guidance to void instead. | P2 | Negative |

## Recurring / Tokenized Payments

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-TOK-001 | POST /v1/payment-methods | Tokenize a card for future use | Merchant has `tokens:write` scope; valid card details supplied. | card=4111111111111111, exp=12/2027 | 201 Created; response returns a non-reversible `token_id`; raw PAN is never echoed back or persisted in plaintext logs. | P1 | Positive |
| API-TOK-002 | POST /v1/payments | Charge a saved token for a recurring subscription payment | A valid, non-expired `token_id` exists for the merchant's customer. | token_id=tok_abc123, amount=9900, currency=AED, recurring=true | 201 Created; `status=CAPTURED` (or AUTHORIZED per flow); no card entry fields required in the request. | P1 | Positive |
| API-TOK-003 | POST /v1/payments | Charging an expired or revoked token is rejected | Token was previously deleted via DELETE /v1/payment-methods/{id}. | token_id=tok_abc123 (revoked) | 410 Gone or 422; error code `TOKEN_INVALID`; no charge attempt reaches the card network. | P1 | Negative |
| API-TOK-004 | DELETE /v1/payment-methods/{id} | Revoke a stored token | Token exists and is active. | token_id=tok_abc123 | 204 No Content; subsequent charge attempts using this token fail with `TOKEN_INVALID`. | P2 | Positive |

## Webhooks / Notifications

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-WHK-001 | Webhook delivery | Successful webhook delivery on payment status change | Merchant has a registered, reachable webhook endpoint. | Event: `payment.captured` | Webhook is delivered within the SLA window; payload includes event type, payment_id, and a signature header; merchant endpoint returns 2xx. | P1 | Positive |
| API-WHK-002 | Webhook delivery | Webhook retry on merchant endpoint failure | Merchant endpoint is configured to return 500 on first two attempts. | Event: `payment.captured` | NexaPay retries with backoff (e.g., up to 5 attempts over an increasing interval); delivery is marked `failed` only after retries are exhausted; retries are logged with attempt count and timestamps. | P2 | Negative |
| API-WHK-003 | Webhook delivery | Duplicate webhook delivery is safely ignorable by design (at-least-once semantics) | A webhook for the same event is redelivered due to a delayed ack. | Event: `payment.captured`, same `event_id` twice | Payload includes a stable `event_id` so the merchant can de-duplicate; NexaPay documentation/behavior confirms at-least-once (not exactly-once) delivery. | P2 | Positive |

## Merchant Onboarding

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-MER-001 | POST /v1/merchants | Create a new merchant application | Caller has platform-level onboarding credentials. | legal_name="Acme Retail LLC", business_type="retail", country=AE | 201 Created; `merchant_id` returned; `kyb_status=PENDING`. | P1 | Positive |
| API-MER-002 | GET /v1/merchants/{id}/kyb-status | Retrieve KYB (Know-Your-Business) verification status | Merchant application exists and has been submitted for review. | merchant_id=mer_789 | 200 OK; returns one of `PENDING`, `APPROVED`, `REJECTED`, `ADDITIONAL_INFO_REQUIRED` with a `last_updated` timestamp. | P2 | Positive |
| API-MER-003 | POST /v1/payments | Payment creation is blocked for a merchant whose KYB is not yet approved | Merchant exists with `kyb_status=PENDING`. | merchant_id=mer_789, amount=1000 | 403 Forbidden; error code `MERCHANT_NOT_ACTIVATED`; no payment record created. | P1 | Negative |

## Idempotency

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-IDEM-001 | POST /v1/payments | Duplicate request with the same idempotency key returns the original result | An initial payment request succeeded with `Idempotency-Key: idem-001`. | Identical payload resent with `Idempotency-Key: idem-001` | 200/201 with the same `payment_id` and status as the original call; no second charge is created against the card. | P1 | Idempotency |
| API-IDEM-002 | POST /v1/payments | Same idempotency key with a different payload is rejected | An initial payment request succeeded with `Idempotency-Key: idem-002`, amount=5000. | Same key, amount changed to 7500 | 422 Unprocessable Entity; error code `IDEMPOTENCY_KEY_CONFLICT`; original payment is untouched. | P2 | Idempotency |
| API-IDEM-003 | POST /v1/payments | Concurrent duplicate requests with the same idempotency key create only one charge | Two requests with the identical key and payload are fired near-simultaneously. | Idempotency-Key: idem-003, amount=3000 (x2 concurrent) | Exactly one `payment_id` is created; the second response returns the same result as the first (no race-condition double charge). | P1 | Idempotency |

## Transaction List / Pagination

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-LIST-001 | GET /v1/payments | List transactions with default pagination | Merchant has more than one page of transactions (e.g., 120 records, default page size 50). | No query params | 200 OK; returns first 50 records plus a `next_cursor`; total count metadata present. | P2 | Positive |
| API-LIST-002 | GET /v1/payments | Filter transactions by status and date range | Merchant has a mix of AUTHORIZED, CAPTURED, and REFUNDED transactions across several days. | status=CAPTURED&from=2026-06-01&to=2026-06-30 | Only CAPTURED transactions within the date range are returned; results are sorted consistently (e.g., newest first) across pages. | P2 | Positive |
| API-LIST-003 | GET /v1/payments | Requesting a page beyond the last cursor returns an empty result, not an error | Merchant has exactly 2 pages of results. | cursor=<value-past-last-page> | 200 OK with an empty `data` array and `next_cursor=null`; no 4xx/5xx error. | P3 | Boundary |

## Error Handling / Security-lite

| TC ID | Endpoint / Method | Title | Preconditions | Request Data (brief) | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| API-ERR-001 | POST /v1/payments | Invalid API key is rejected | No valid API key supplied, or a revoked key is used. | Authorization: Bearer invalid_key | 401 Unauthorized; error code `INVALID_API_KEY`; no details about why the key is invalid beyond a generic message (avoids key-guessing hints). | P1 | Negative |
| API-ERR-002 | POST /v1/payments | Malformed JSON payload is rejected with a clear error | Well-formed API key; payload is syntactically invalid JSON. | `{ "amount": 5000, "currency": }` | 400 Bad Request; error code `MALFORMED_REQUEST`; no server stack trace leaked in the response body. | P2 | Negative |
| API-ERR-003 | POST /v1/payments | Unsupported currency code is rejected | Valid API key and payload otherwise complete. | currency="XXX" (not a supported ISO 4217 code) | 422 Unprocessable Entity; error code `UNSUPPORTED_CURRENCY`; lists supported currencies or a documentation link. | P2 | Negative |
| API-ERR-004 | POST /v1/merchants | API key with insufficient scope is rejected | API key only has `payments:read` scope, not `merchants:write`. | Attempt to create a merchant with a payments-scoped key | 403 Forbidden; error code `INSUFFICIENT_SCOPE`; required scope is named in the error to aid integration debugging. | P2 | Security-lite |

## Notes

All card numbers above are well-known publicly documented test/sandbox values used across the payments industry for non-production testing; none reference a live account. NexaPay, its endpoints, and field names are invented for this portfolio and are illustrative rather than a literal API specification.
