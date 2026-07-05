# Database Test Cases — NexaPay Ledger & Merchant Data Store

**Platform under test:** the data layer behind NexaPay (fictional payment gateway) — a relational ledger holding `merchants`, `orders`, `transactions`, and `refunds` tables, plus a webhook delivery/event log. It also covers the ledger behind **NexaPay Wallet**, NexaPay's consumer digital-wallet product (wallet balance ledger, P2P transfer atomicity, bill-payment idempotency, transfer limits). These test cases focus on data integrity, consistency, and financial correctness rather than UI or API behavior.

**Total test cases:** 34
**Columns:** TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type

## Transaction State Machine Integrity

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-STATE-001 | State machine | A transaction cannot jump from CREATED directly to SETTLED | Seed a transaction row in `CREATED` state. | Attempt a direct update setting `status='SETTLED'` bypassing the application's state-transition service (or query the state-transition log for this row). | Write is rejected by a DB constraint/trigger, or if performed out-of-band, an audit alert/inconsistency is detectable; valid path is CREATED → AUTHORIZED → CAPTURED → SETTLED. | P1 | Negative |
| DB-STATE-002 | State machine | State transition history is fully recorded | A transaction progresses through its normal lifecycle via the application. | Query `transaction_state_history` for the transaction ID. | Every state change (CREATED, AUTHORIZED, CAPTURED, SETTLED) has a corresponding row with timestamp and actor; no gaps in the sequence. | P1 | Positive |
| DB-STATE-003 | State machine | Terminal states (VOIDED, REFUNDED, DECLINED) cannot be transitioned further | Seed a transaction in `REFUNDED` state. | Attempt to update it to `CAPTURED`. | Update is rejected by a check constraint or application-level guard; row remains `REFUNDED`. | P2 | Negative |

## Referential Integrity

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-REF-001 | Referential integrity | Every transaction references a valid, existing order | Database contains transactions created through normal flows. | `SELECT * FROM transactions t LEFT JOIN orders o ON t.order_id = o.id WHERE o.id IS NULL;` | Query returns zero rows (no orphaned transactions). | P1 | Positive |
| DB-REF-002 | Referential integrity | Every order references a valid, existing merchant | Database contains orders across multiple merchants. | `SELECT * FROM orders o LEFT JOIN merchants m ON o.merchant_id = m.id WHERE m.id IS NULL;` | Query returns zero rows. | P1 | Positive |
| DB-REF-003 | Referential integrity | A refund cannot exist without a parent transaction | Attempt to insert a refund row with a non-existent `transaction_id`. | `INSERT INTO refunds (transaction_id, amount) VALUES ('nonexistent-id', 500);` | Insert fails due to a foreign-key constraint violation; no orphaned refund is persisted. | P1 | Negative |
| DB-REF-004 | Referential integrity | Deleting a merchant is blocked while active orders/transactions exist | Merchant has at least one non-terminal-state order. | Attempt `DELETE FROM merchants WHERE id = ?` for that merchant. | Delete is rejected by a foreign-key restrict/cascade policy, or the merchant is soft-deleted instead of hard-deleted, preserving transaction history. | P2 | Negative |

## Double-Entry Ledger Reconciliation

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-LED-001 | Ledger reconciliation | Sum of debits equals sum of credits across the ledger | Ledger contains a representative sample of captures, refunds, and fee entries for a settlement period. | `SELECT SUM(debit) - SUM(credit) FROM ledger_entries WHERE period = ?;` | Result is exactly 0.00 for a balanced period; any non-zero delta indicates a reconciliation defect. | P1 | Positive |
| DB-LED-002 | Ledger reconciliation | Every capture produces a matching pair of ledger entries | A payment is captured for 5000 (merchant currency minor units). | Query `ledger_entries` for the transaction ID. | Exactly one debit entry and one matching credit entry exist, both referencing the same transaction ID and summing to zero net effect on the control account. | P1 | Positive |
| DB-LED-003 | Ledger reconciliation | A refund produces a reversing ledger entry, not a deletion of the original | A previously captured transaction is refunded in full. | Query `ledger_entries` for the transaction ID after refund. | Original capture entries remain untouched (immutable audit trail); new reversing entries are appended; net balance for the transaction is zero. | P1 | Positive |

## Idempotent Write Behavior

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-IDEM-001 | Idempotency | Retried webhook event does not create a duplicate transaction record | A `payment.captured` webhook event is processed, then redelivered with the same `event_id`. | Query `transactions` and `processed_events` tables after both deliveries. | `processed_events` shows the `event_id` recorded once; only one transaction/state-update is applied; the second delivery is a no-op. | P1 | Idempotency |
| DB-IDEM-002 | Idempotency | Idempotency key table correctly stores the original response for replay | A payment request is created with `Idempotency-Key: idem-100`. | Query `idempotency_keys` table for `idem-100`. | Row exists with the stored response payload/hash and an expiry; a retried request with the same key returns the stored result rather than re-executing business logic. | P2 | Positive |

## PII / PCI Data Handling

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-PII-001 | PCI data masking | Raw card number (PAN) is never stored in any table | A payment has been processed with a full card number. | Scan `transactions`, `payment_methods`, and any logging/staging tables for 16-digit numeric patterns matching the input PAN. | No table contains the raw PAN; only a token reference and masked last-4 digits (e.g., `**** 1111`) are present. | P1 | Security |
| DB-PII-002 | PCI data masking | CVV is never persisted anywhere, even transiently | A payment has been processed including CVV entry. | Scan `transactions`, `payment_methods`, staging tables, and audit logs for the CVV value used. | CVV value is not found in any persisted table or log; CVV is used only in-memory/in-transit per PCI-DSS guidance. | P1 | Security |
| DB-PII-003 | PCI data masking | Card brand and expiry are stored, but full expiry is not exposed unmasked in exports | A merchant requests a transaction export/report. | Query the export generation logic/table for card-related columns. | Export contains masked PAN and brand only; full expiry date, if stored, is not included in merchant-facing exports. | P2 | Security |

## Currency & Amount Precision

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-AMT-001 | Amount precision | Monetary columns use integer minor units, not floating point | Inspect schema definition for `transactions.amount` and `ledger_entries.debit/credit`. | `\d+ transactions` (or equivalent schema introspection). | Columns are defined as integer/decimal fixed-precision types (e.g., `BIGINT` minor units or `NUMERIC(19,4)`), never `FLOAT`/`DOUBLE`. | P1 | Positive |
| DB-AMT-002 | Amount precision | Repeated partial captures/refunds do not accumulate rounding drift | A transaction of 10000 is partially captured three times (3334 + 3333 + 3333). | Sum the three capture amounts and compare to the original authorized amount. | Sum equals exactly 10000 with no residual fractional unit lost or gained; ledger balances to zero. | P1 | Boundary |
| DB-AMT-003 | Amount precision | Currency minor-unit exponent is respected for zero-decimal currencies | A payment is created in a zero-decimal currency (e.g., JPY-style, no minor units). | Insert/query a transaction amount for such a currency. | Amount is stored and interpreted in whole units, not divided by 100; no accidental 100x underscale/overscale defect. | P2 | Boundary |

## Audit Trail / Soft Delete

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-AUD-001 | Audit trail | Financial records are never hard-deleted | Attempt to delete a settled transaction row directly. | `DELETE FROM transactions WHERE id = ?` for a SETTLED transaction. | Delete is blocked at the DB or application layer; records use a `deleted_at`/`is_active` soft-delete flag instead, preserving history for audit/reconciliation. | P1 | Negative |
| DB-AUD-002 | Audit trail | Every mutating change to a transaction is captured in an audit log | A transaction is captured, then partially refunded. | Query `audit_log` for the transaction ID. | Each state-changing action has a corresponding audit entry with old value, new value, actor, and timestamp. | P2 | Positive |

## Performance / Query Behavior at Scale

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-PERF-001 | Query performance | Merchant statement query performs acceptably on a high-volume transactions table | `transactions` table seeded with several million rows across many merchants. | Run the merchant-statement query filtering by `merchant_id` and date range; inspect the query plan. | Query plan shows use of a composite index on `(merchant_id, created_at)`; execution time stays within an agreed threshold (e.g., sub-second) rather than a full table scan. | P2 | Positive |
| DB-PERF-002 | Query performance | Pagination on the transaction list uses keyset/cursor pagination, not high-offset OFFSET scans | Table has more than 500,000 rows for a merchant. | Compare query plan/timing for `OFFSET 400000 LIMIT 50` versus a cursor-based equivalent. | Cursor-based approach is used in production paths; offset-based deep pagination is avoided or flagged as a known performance risk. | P3 | Positive |

## Orphaned Records

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-ORPH-001 | Orphaned records | No refund rows exist without a corresponding transaction | Run after a batch of refund operations, including simulated partial failures. | `SELECT * FROM refunds r LEFT JOIN transactions t ON r.transaction_id = t.id WHERE t.id IS NULL;` | Query returns zero rows. | P1 | Negative |
| DB-ORPH-002 | Orphaned records | No webhook event log entries reference a non-existent merchant | Webhook events are logged per merchant subscription. | `SELECT * FROM webhook_events w LEFT JOIN merchants m ON w.merchant_id = m.id WHERE m.id IS NULL;` | Query returns zero rows. | P2 | Negative |

## NexaPay Wallet — Balance Ledger Consistency

These test cases extend the ledger-integrity approach above to **NexaPay Wallet**, the consumer digital-wallet product, covering top-up, P2P transfer, and bill-payment activity against the wallet's own ledger tables (`wallet_balances`, `wallet_ledger_entries`).

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-WBAL-001 | Wallet ledger | Wallet balance ledger reflects the exact top-up amount as a credit entry | A wallet top-up of 20000 minor units has been confirmed successful. | `SELECT * FROM wallet_ledger_entries WHERE reference_id = ? AND entry_type='TOPUP_CREDIT';` | Exactly one credit entry of 20000 exists, referencing the top-up ID; no other wallet ledger rows are affected. | P1 | Positive |
| DB-WBAL-002 | Wallet ledger | Wallet balance ledger reflects the exact bill-payment amount as a debit entry | A bill payment of 15000 minor units has completed successfully. | `SELECT * FROM wallet_ledger_entries WHERE reference_id = ? AND entry_type='BILL_DEBIT';` | Exactly one debit entry of 15000 exists, referencing the bill payment ID. | P1 | Positive |
| DB-WBAL-003 | Wallet ledger | Sum of wallet ledger entries reconciles to the displayed wallet balance | Wallet has a mix of top-up, P2P, and bill-payment entries over its history. | `SELECT SUM(credit) - SUM(debit) FROM wallet_ledger_entries WHERE wallet_id = ?;` | Result exactly equals the `wallet_balances.available_balance` value for that wallet; any drift indicates a reconciliation defect. | P1 | Positive |

## NexaPay Wallet — P2P Transfer Atomicity

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-P2P-001 | Transfer atomicity | P2P transfer debit and credit entries are written atomically | A P2P transfer of 5000 between wallet A and wallet B is submitted. | Query `wallet_ledger_entries` for both wallets immediately after the transfer response, and inspect the transaction/commit boundary in the application logs. | Both the sender debit and recipient credit are committed within a single database transaction; either both rows exist or neither does — no window where only one side is visible. | P1 | Positive |
| DB-P2P-002 | Transfer atomicity | A failed P2P transfer leaves no partial debit/credit state | The recipient wallet is simulated as unavailable/frozen mid-transfer, forcing a failure after the sender-side debit would otherwise be applied. | Query `wallet_ledger_entries` and `wallet_balances` for both wallets after the failure. | Neither wallet shows a debit or credit for the failed attempt; sender balance is unchanged; the transfer record is marked `FAILED`, not left `IN_PROGRESS`. | P1 | Negative |
| DB-P2P-003 | Transfer atomicity | Concurrent P2P transfers from the same wallet cannot overdraw the balance | Wallet has a balance of 6000; two concurrent transfer requests of 5000 each are submitted at the same time. | Inspect row-level locking behavior (e.g., `SELECT ... FOR UPDATE` on `wallet_balances`) and the final ledger state. | Exactly one transfer succeeds; the second is rejected with `INSUFFICIENT_BALANCE`; the wallet balance never goes negative. | P1 | Boundary |

## NexaPay Wallet — Bill Payment Idempotency

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-BILL-001 | Idempotency | Retried bill-payment webhook does not create a duplicate debit | A biller-confirmation webhook for a bill payment is processed, then redelivered with the same `event_id`. | Query `processed_events` and `wallet_ledger_entries` after both deliveries. | `processed_events` records the `event_id` once; only one debit entry exists for the bill payment; the second delivery is a no-op. | P1 | Idempotency |
| DB-BILL-002 | Idempotency | Bill payment provider transaction reference is unique per payment | Two distinct bill payments are submitted, each expected to receive its own provider reference. | `SELECT provider_reference, COUNT(*) FROM bill_payments GROUP BY provider_reference HAVING COUNT(*) > 1;` | Query returns zero rows; a unique constraint on `provider_reference` prevents accidental duplicate recording of the same underlying payment. | P2 | Positive |

## NexaPay Wallet — Transfer Limit Enforcement

| TC ID | Area | Title | Precondition / Setup | Steps / Query Intent | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| DB-LIM-001 | Limit enforcement | Daily transfer limit is enforced via an aggregate query at the data layer, not app logic alone | Wallet has transferred 9000 today against a configured daily limit of 10000. | Run the daily-aggregate check the application relies on: `SELECT SUM(amount) FROM wallet_transfers WHERE sender_wallet_id = ? AND created_at::date = CURRENT_DATE;` before authorizing a new transfer. | Aggregate correctly reflects 9000; a new transfer of 2000 is blocked before any ledger write occurs, confirming the limit check reads committed data, not a stale in-memory counter. | P2 | Positive |
| DB-LIM-002 | Limit enforcement | Per-transaction transfer limit constraint rejects an oversized single transfer at insert time | A single transfer of 15000 is attempted against a per-transaction limit of 10000. | Attempt the insert/stored-procedure call that backs `POST /v1/wallet/transfers`. | Insert is rejected by a check constraint or application-level guard before any ledger rows are written; no orphaned `IN_PROGRESS` transfer row remains. | P2 | Boundary |

## Notes

Table and column names above (`transactions`, `ledger_entries`, `idempotency_keys`, `wallet_ledger_entries`, `wallet_balances`, `wallet_transfers`, `bill_payments`, etc.) are illustrative and generic to relational payment-ledger design; they are not drawn from any specific real vendor's schema.
