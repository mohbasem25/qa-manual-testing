# UI Test Cases — NexaPay Merchant Portal

**Platform under test:** the NexaPay Merchant Portal — a fictional web dashboard where merchants log in to view transactions, issue refunds, review settlement/payout reports, and manage API keys for their integration with the (fictional) NexaPay payment gateway. Also covers the **NexaPay Wallet** consumer app — the wallet balance/top-up, P2P transfer, bill payment, and wallet transaction history screens.

**Total test cases:** 33
**Columns:** TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type

## Login & Authentication

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-LOGIN-001 | Successful login with valid credentials and 2FA | Merchant account exists with email/password and 2FA (authenticator app) enrolled. | 1. Enter valid email/password. 2. Enter the correct 6-digit 2FA code. 3. Click 'Sign In'. | Email: ops@merchant-demo.test; valid TOTP code | User is authenticated and lands on the Dashboard; session token is issued; 2FA code cannot be reused (single-use). | P1 | Positive |
| UI-LOGIN-002 | Login is rejected with an incorrect 2FA code | Merchant has entered correct email/password. | 1. Enter a valid password. 2. Enter an incorrect 6-digit code. 3. Click 'Verify'. | 2FA code: '000000' (incorrect) | Login is rejected; inline error 'Invalid verification code'; user remains on the 2FA challenge screen; attempt is counted toward lockout threshold. | P1 | Negative |
| UI-LOGIN-003 | Account locks out after repeated failed login attempts | Merchant account is in a normal, unlocked state. | 1. Attempt login with an incorrect password 5 times in a row. 2. Attempt a 6th login with the correct password. | Password: wrong x5, then correct | Account is temporarily locked after the configured threshold; the 6th (correct) attempt is also rejected with a 'temporarily locked, try again later' message. | P2 | Negative |

## Dashboard & Transaction List

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-DASH-001 | Transaction list displays recent transactions on load | Merchant has at least 10 transactions in the last 30 days. | 1. Log in. 2. Navigate to Transactions tab. | N/A | List loads showing transaction ID, date, amount, currency, status, and masked card; sorted newest-first by default. | P1 | Positive |
| UI-DASH-002 | Filtering transactions by date range | Merchant has transactions spanning several months. | 1. Open the date-range filter. 2. Select a specific start and end date. 3. Apply filter. | Range: 2026-06-01 to 2026-06-30 | Only transactions with a created date inside the selected range are shown; result count updates; filter chip is visible and removable. | P2 | Positive |
| UI-DASH-003 | Filtering transactions by status | Merchant has transactions in AUTHORIZED, CAPTURED, REFUNDED, and DECLINED states. | 1. Open the status filter. 2. Select 'Refunded' only. | Status: REFUNDED | Only REFUNDED transactions are displayed; other statuses are excluded; filter can be cleared to restore the full list. | P2 | Positive |
| UI-DASH-004 | Filtering transactions by currency | Merchant processes transactions in multiple currencies (AED, USD, EUR). | 1. Open the currency filter. 2. Select 'USD'. | Currency: USD | Only USD-denominated transactions are shown; amounts display with the correct currency symbol/code. | P3 | Positive |
| UI-DASH-005 | Combining multiple filters narrows results correctly | Merchant has a mixed transaction history. | 1. Apply a date range. 2. Additionally apply a status filter. 3. Additionally apply a currency filter. | Range + status=CAPTURED + currency=AED | Filters are applied cumulatively (AND logic); result set matches all three conditions simultaneously; clearing one filter re-widens results correctly. | P2 | Positive |

## Transaction Detail & Refund

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-DET-001 | Transaction detail view shows full lifecycle history | A transaction has been authorized, captured, and partially refunded. | 1. Click a transaction row to open its detail view. | N/A | Detail page shows a timeline of state changes with timestamps (Authorized → Captured → Partially Refunded), original amount, and remaining refundable balance. | P2 | Positive |
| UI-DET-002 | Initiating a full refund from the transaction detail view | Transaction is in `CAPTURED` state with no prior refunds. | 1. Open transaction detail. 2. Click 'Refund'. 3. Select 'Full amount'. 4. Confirm. | Captured amount: AED 250.00 | Refund is submitted; status updates to `REFUNDED`; a confirmation toast/message is shown; refund appears in the transaction timeline. | P1 | Positive |
| UI-DET-003 | Initiating a partial refund with amount validation | Transaction is `CAPTURED`, amount AED 250.00. | 1. Click 'Refund'. 2. Enter a partial amount. 3. Confirm. | Refund amount: AED 100.00 | Refund succeeds; status updates to `PARTIALLY_REFUNDED`; remaining refundable balance shown as AED 150.00. | P1 | Positive |
| UI-DET-004 | Refund amount exceeding the refundable balance is blocked in the UI | Transaction is `CAPTURED`, AED 250.00, with AED 150.00 already refunded. | 1. Click 'Refund'. 2. Enter an amount greater than the remaining AED 100.00 balance. | Refund amount entered: AED 150.00 | 'Refund' button is disabled or an inline validation error appears ('Amount exceeds refundable balance') before submission; no request is sent. | P1 | Boundary |
| UI-DET-005 | Double-clicking 'Confirm Refund' does not create two refunds | Transaction is eligible for a full refund. | 1. Click 'Refund', enter amount, click 'Confirm'. 2. Immediately click 'Confirm' again before the page responds. | Refund amount: AED 250.00 | Button is disabled after first click / a loading state is shown; only one refund request is submitted and processed. | P1 | Negative |

## Settlement / Payout Reports

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-SET-001 | Viewing the settlement/payout report for a completed period | Merchant has at least one completed settlement period. | 1. Navigate to 'Settlements' tab. 2. Select a past period. | Period: June 2026 | Report shows gross transaction volume, fees deducted, refunds, and net payout amount; totals reconcile (gross − fees − refunds = net payout). | P1 | Positive |
| UI-SET-002 | Exporting a settlement report as CSV | A settlement report is displayed on screen. | 1. Click 'Export CSV'. 2. Open the downloaded file. | N/A | CSV downloads successfully; column headers and row totals match the on-screen report exactly. | P3 | Positive |

## API Key Management

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-KEY-001 | Generating a new API key | Merchant is logged in with admin privileges; navigates to API Keys screen. | 1. Click 'Generate New Key'. 2. Name the key. 3. Confirm. | Key label: 'Production Backend' | New key is generated and shown in full exactly once; a warning states it will not be shown again; key appears in the list afterward only in masked form. | P1 | Positive |
| UI-KEY-002 | Revoking an existing API key | At least one active API key exists. | 1. Click 'Revoke' next to a key. 2. Confirm the action in the dialog. | Key: 'Production Backend' | Key status changes to 'Revoked'; any API calls using that key subsequently fail with an authentication error (cross-checked against API-ERR-001). | P1 | Positive |
| UI-KEY-003 | Revoking an API key requires explicit confirmation | An active API key exists. | 1. Click 'Revoke'. 2. Observe the confirmation dialog. 3. Click 'Cancel'. | N/A | Key is NOT revoked after cancelling; dialog clearly warns that revocation is immediate and irreversible. | P2 | Negative |

## Multi-Currency Display & Responsiveness

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-DISP-001 | Amounts display with correct currency symbol and decimal formatting | Merchant has transactions in AED, USD, and a zero-decimal currency. | 1. View the transaction list containing mixed currencies. | AED 1,250.50; USD 340.00 | Each amount shows the correct symbol/code and decimal precision for its currency (e.g., zero-decimal currencies show no decimal places); thousands separators render correctly. | P2 | Positive |
| UI-DISP-002 | Portal renders correctly across supported browsers and a tablet viewport | Test environment has Chrome, Firefox, Safari, and a tablet-width (768px) viewport available. | 1. Load the Dashboard in each browser/viewport. 2. Check layout, navigation, and table readability. | N/A | No broken layout, overlapping elements, or horizontal scroll issues; core navigation and transaction table remain usable at tablet width. | P3 | Positive |

## Role-Based UI Permissions

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-PERM-001 | Support-role user sees masked card data only, not full PAN | A user account is assigned the 'Support' role (read-only, limited PII visibility). | 1. Log in as a Support-role user. 2. Open a transaction detail view. | Card on file: **** 1111 | Only masked card (last 4 digits) and brand are visible; no UI control or hidden field exposes the full PAN; API responses backing the view are also masked. | P1 | Security |
| UI-PERM-002 | Support-role user cannot access API Key Management or issue refunds | Support-role user is logged in. | 1. Attempt to navigate directly to the API Keys URL. 2. Attempt to open a transaction and locate a 'Refund' control. | N/A | Navigation is blocked with a 'not authorized' page or the menu item is absent; 'Refund' action is not present/is disabled for this role. | P1 | Security |

## NexaPay Wallet — Balance & Top-Up

The **NexaPay Wallet** consumer app is a separate front end from the Merchant Portal above, used by end users to hold a balance, top it up, send P2P transfers, and pay bills.

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-WAL-001 | Wallet balance displays correctly on the Wallet home screen | Consumer is logged in and has a non-zero wallet balance. | 1. Log in to the NexaPay Wallet app. 2. Land on the Wallet home screen. | Balance: AED 340.50 | Current available balance is displayed prominently, correctly formatted with currency symbol and two-decimal precision; balance matches the backend ledger total. | P1 | Positive |
| UI-WAL-002 | Wallet top-up flow: selecting a funding source and entering an amount | Consumer has at least one linked bank card and a bank-transfer option available. | 1. Tap 'Top Up'. 2. Select a funding source (linked card or bank transfer). 3. Enter an amount. 4. Confirm. | Amount: AED 200.00; source: linked card | Confirmation screen appears before submission; amount below a configured minimum is blocked with inline validation; a valid amount proceeds to submission. | P1 | Positive |
| UI-WAL-003 | Wallet top-up confirmation screen shows the updated balance | A top-up of AED 200.00 has just completed successfully. | 1. Complete a top-up. 2. Observe the success/confirmation screen. 3. Return to the Wallet home screen. | Amount: AED 200.00 | Success screen shows the amount credited and a transaction reference; Wallet home screen balance reflects the new total without requiring a manual refresh. | P1 | Positive |

## NexaPay Wallet — P2P Transfer

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-P2P-001 | P2P transfer: recipient lookup by phone number or username | Consumer wants to send money to another NexaPay user. | 1. Tap 'Send Money'. 2. Enter a recipient's phone number or username. 3. Observe the lookup result. | Recipient: +9715xxxxxxx | Matching recipient's masked name/profile is shown for confirmation before any amount is entered; an unknown phone number/username shows a clear 'no NexaPay user found' state. | P1 | Positive |
| UI-P2P-002 | P2P transfer confirmation screen before submission | A valid recipient has been found and an amount entered. | 1. Enter a transfer amount. 2. Tap 'Continue'. 3. Review the confirmation screen. | Amount: AED 50.00 | Confirmation screen clearly shows recipient name, amount, and any fee (if applicable) before the user can tap the final 'Send' action; amount and recipient cannot be silently altered after this screen. | P1 | Positive |
| UI-P2P-003 | P2P transfer receipt/success screen with transaction reference | A P2P transfer has just completed successfully. | 1. Confirm and send a transfer. 2. Observe the success/receipt screen. | Amount: AED 50.00 | Receipt screen shows amount, recipient, timestamp, and a unique transaction reference; receipt is also retrievable later from transaction history. | P1 | Positive |
| UI-P2P-004 | P2P transfer is blocked in the UI when the amount exceeds available balance | Wallet balance is AED 30.00. | 1. Tap 'Send Money'. 2. Select a valid recipient. 3. Enter an amount greater than the available balance. | Amount entered: AED 50.00 | 'Continue'/'Send' action is disabled or an inline validation error appears ('Insufficient balance') before submission; no transfer request is sent. | P1 | Boundary |

## NexaPay Wallet — Bill Payment

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-BILL-001 | Bill payment: biller selection from a category list | Consumer wants to pay a utility or telecom bill. | 1. Tap 'Pay Bills'. 2. Browse or search biller categories (e.g., Electricity, Telecom). 3. Select a specific biller. | Category: Telecom; Biller: sample telecom provider | Biller categories and billers load correctly; selecting a biller proceeds to the reference-entry screen for that biller. | P2 | Positive |
| UI-BILL-002 | Bill payment: biller reference entry validation | A biller has been selected that requires a fixed-format account/reference number. | 1. Enter a biller reference in an invalid format. 2. Attempt to continue. | Reference entered: 'ABC' (invalid format) | Inline validation error is shown before submission ('Enter a valid account/reference number'); 'Continue' remains disabled until a validly formatted reference is entered. | P1 | Negative |
| UI-BILL-003 | Bill payment confirmation and success screen | A valid biller reference and amount have been entered. | 1. Review the confirmation screen showing biller, reference, and amount. 2. Confirm payment. 3. Observe the success screen and status. | Amount: AED 150.00 | Confirmation screen requires explicit confirmation before charging the wallet; success screen shows a `Pending`/`Success` status consistent with the backend, with a transaction reference. | P1 | Positive |

## NexaPay Wallet — Transaction History

| TC ID | Title | Preconditions | Steps | Test Data | Expected Result | Priority | Type |
|---|---|---|---|---|---|---|---|
| UI-HIST-001 | Transaction history filtering by wallet activity type | Wallet history contains a mix of top-ups, P2P transfers, and bill payments. | 1. Navigate to 'Transaction History'. 2. Apply a filter for 'P2P Transfers' only. 3. Switch the filter to 'Bill Payments' only. | Filter: P2P Transfers, then Bill Payments | Each filter correctly narrows the list to only the selected activity type; each entry shows type-appropriate details (recipient for P2P, biller for bill payment); filters can be cleared to restore the full combined history. | P2 | Positive |

## Notes

Screen names, field labels, and roles above are generic and illustrative of a typical PSP merchant dashboard and consumer wallet app; they are not modeled on any specific real vendor's UI.
