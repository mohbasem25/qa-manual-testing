# UI Test Cases — NexaPay Merchant Portal

**Platform under test:** the NexaPay Merchant Portal — a fictional web dashboard where merchants log in to view transactions, issue refunds, review settlement/payout reports, and manage API keys for their integration with the (fictional) NexaPay payment gateway.

**Total test cases:** 20
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

## Notes

Screen names, field labels, and roles above are generic and illustrative of a typical PSP merchant dashboard; they are not modeled on any specific real vendor's UI.
