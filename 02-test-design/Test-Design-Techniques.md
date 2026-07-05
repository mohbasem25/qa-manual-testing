# Test Design Techniques — ShopSphere Checkout & Payment

**Document ID:** TD-CHK-001
**Related:** `01-test-plan/Test-Plan.md`, `03-test-cases/Test-Cases.csv`
**Purpose:** Demonstrate explicit application of ISTQB black-box test design techniques (Equivalence Partitioning, Boundary Value Analysis, Decision Tables, Exploratory/Session-Based Testing) to derive the test cases in `03-test-cases/`.

---

## 1. Equivalence Partitioning (EP)

EP reduces an input domain into partitions where all values are expected to be treated the same by the system, so one representative value per partition is tested.

### 1.1 Promo Code Input

**Business rule:** Promo codes are 6–10 alphanumeric characters, case-insensitive, and must exist in the active promotions table.

| Partition | Description | Representative Value | Expected Behavior | Valid/Invalid |
|---|---|---|---|---|
| EP1 | Valid, active, unused code | `SAVE10NOW` | Discount applied, confirmation message shown | Valid |
| EP2 | Valid format, but expired | `WINTER22` | Rejected with "This code has expired" | Invalid |
| EP3 | Valid format, but already used by this account | `WELCOME5` (single-use, previously redeemed) | Rejected with "This code has already been used" | Invalid |
| EP4 | Valid format, does not exist in system | `ZZZZZ99` | Rejected with "Invalid promo code" | Invalid |
| EP5 | Too short (< 6 chars) | `AB1` | Rejected client-side, "Code must be 6–10 characters" | Invalid |
| EP6 | Too long (> 10 chars) | `SAVEBIGMONEY2026` | Rejected client-side or truncated with error | Invalid |
| EP7 | Contains special characters | `SAVE-10%` | Rejected, "Only letters and numbers allowed" | Invalid |
| EP8 | Empty input, "Apply" clicked | `` (blank) | "Apply" disabled or "Enter a code" message; no crash | Invalid |
| EP9 | Valid code, lowercase entry | `save10now` | Treated case-insensitively, discount applied | Valid |

### 1.2 Card Number Length / Format

**Business rule:** Accepted card networks are Visa (16 digits), Mastercard (16 digits), Amex (15 digits). Field must accept digits only (spaces auto-stripped for display).

| Partition | Description | Representative Value | Expected Behavior | Valid/Invalid |
|---|---|---|---|---|
| EP1 | Valid Visa, passes Luhn check | `4111 1111 1111 1111` | Accepted, card type icon shows Visa | Valid |
| EP2 | Valid Amex, 15 digits, passes Luhn | `3782 822463 10005` | Accepted, card type icon shows Amex, CVV field expects 4 digits | Valid |
| EP3 | 16-digit number, fails Luhn check | `4111 1111 1111 1112` | Rejected, "Card number is invalid" | Invalid |
| EP4 | Too short (< 13 digits) | `4111 1111 111` | Rejected, "Card number is invalid" | Invalid |
| EP5 | Too long (> 19 digits) | `4111 1111 1111 1111 111` | Input blocked beyond max length, or rejected on submit | Invalid |
| EP6 | Contains letters | `4111 11AB 1111 1111` | Rejected client-side, non-numeric input blocked/stripped | Invalid |
| EP7 | Unsupported network (e.g., Diners Club prefix) | `3056 9309 0259 04` | Rejected, "Card type not supported" | Invalid |

### 1.3 Shipping Postal Code (US format used as baseline)

| Partition | Description | Representative Value | Expected Behavior | Valid/Invalid |
|---|---|---|---|---|
| EP1 | Valid 5-digit ZIP | `94105` | Accepted | Valid |
| EP2 | Valid ZIP+4 | `94105-1234` | Accepted | Valid |
| EP3 | Too few digits | `941` | Rejected, "Enter a valid ZIP code" | Invalid |
| EP4 | Contains letters | `9410A` | Rejected | Invalid |
| EP5 | Correct length, non-existent ZIP | `00000` | Accepted client-side (format valid); flagged only if backend ZIP-lookup validation is implemented — otherwise passes to shipping calc, which should not crash | Boundary/assumption — see Note below |

> **Note:** EP5 highlights a scope decision to confirm with the business: does the system validate ZIP *existence* or only *format*? Test cases assume format-only validation unless stated otherwise, and this assumption is logged as a risk in the Test Plan (§12, R6-adjacent).

---

## 2. Boundary Value Analysis (BVA)

BVA focuses on values at and around the edges of a valid range, where off-by-one defects are most likely.

### 2.1 Free Shipping Threshold

**Business rule:** Orders with merchandise subtotal ≥ $50.00 qualify for free standard shipping.

| Boundary | Value | Expected Result |
|---|---|---|
| Just below threshold | $49.98 | Standard shipping fee applies (e.g., $5.99); free shipping option not offered |
| At lower boundary − 0.01 | $49.99 | Standard shipping fee applies |
| Exactly at threshold | $50.00 | Free shipping option available and auto-selected/highlighted |
| Just above threshold | $50.01 | Free shipping option available |
| Well above threshold | $250.00 | Free shipping available; no other pricing side effects |

### 2.2 Line Item Quantity Limit

**Business rule:** Maximum quantity per line item is 10 units (inventory/fraud control).

| Boundary | Value | Expected Result |
|---|---|---|
| Minimum − 1 | 0 | Not selectable; stepper minimum enforced at 1, or item removed at 0 |
| Minimum | 1 | Accepted |
| Just below max | 9 | Accepted |
| Maximum | 10 | Accepted, stepper "+" control becomes disabled |
| Max + 1 | 11 (via manual input if field is editable) | Rejected, value clamped to 10 with inline message "Maximum quantity is 10" |
| Well above max | 999 | Rejected, same clamp/message behavior, no crash or silent overflow |

### 2.3 Promo Code Length (paired with EP §1.1)

| Boundary | Value | Length | Expected Result |
|---|---|---|---|
| Min − 1 | `SAVE1` | 5 | Rejected |
| Min | `SAVE10` | 6 | Accepted (format-valid; still subject to existence/expiry check) |
| Max | `SAVE10ANNIV` | 10 | Accepted (format-valid) |
| Max + 1 | `SAVE10ANNIVX` | 11 | Rejected |

### 2.4 CVV Field Length

**Business rule:** CVV is 3 digits for Visa/Mastercard, 4 digits for Amex.

| Boundary | Card Type | Value | Length | Expected Result |
|---|---|---|---|---|
| Min − 1 | Visa | `12` | 2 | Rejected |
| Min/Max (exact) | Visa | `123` | 3 | Accepted |
| Max + 1 | Visa | `1234` | 4 | Rejected/truncated for non-Amex |
| Min − 1 | Amex | `123` | 3 | Rejected |
| Min/Max (exact) | Amex | `1234` | 4 | Accepted |

---

## 3. Decision Table — Payment Method × Shipping Country × Promo Validity

**Business rules encoded:**
- ShopSphere Wallet is only available for domestic (US) shipping addresses (regulatory/fraud constraint).
- International orders (non-US) surcharge a flat $4.00 handling fee, shown at Order Review.
- An invalid/expired promo code never blocks checkout — it is simply not applied, and the customer is notified.
- A valid promo code combined with a wallet payment is allowed and stacks normally.

| Rule # | Payment Method | Shipping Country | Promo Code Validity | → Outcome: Promo Applied? | → Outcome: Payment Allowed? | → Outcome: Intl. Surcharge? | Notes |
|---|---|---|---|---|---|---|---|
| 1 | Card | US | Valid | Yes | Yes | No | Baseline domestic happy path |
| 2 | Card | US | Invalid/Expired | No (with notice) | Yes | No | Checkout proceeds without discount |
| 3 | Card | Non-US | Valid | Yes | Yes | Yes | Discount + surcharge both shown at Order Review |
| 4 | Card | Non-US | Invalid/Expired | No (with notice) | Yes | Yes | Surcharge still applies |
| 5 | Wallet | US | Valid | Yes | Yes | No | Wallet allowed domestically |
| 6 | Wallet | US | Invalid/Expired | No (with notice) | Yes | No | |
| 7 | Wallet | Non-US | Valid | N/A | **No** | N/A | Wallet option must be hidden/disabled for non-US address; if somehow submitted, must be rejected server-side with a clear error, not silently charged |
| 8 | Wallet | Non-US | Invalid/Expired | N/A | **No** | N/A | Same as Rule 7 — country restriction takes precedence over promo state |

**Design notes:**
- Rules 7–8 are the highest-value rows: they test a *combinational* constraint that simple EP/BVA on individual fields would never surface — the wallet+country restriction only manifests when both conditions are evaluated together.
- Test cases TC-PAY-014 and TC-PAY-015 (see `03-test-cases/`) implement Rules 7–8, both as a UI-level check (wallet option not selectable) and an API-tamper check (see security-lite cases) confirming server-side enforcement, not just client-side hiding.

---

## 4. Exploratory Testing Charters (Session-Based Test Management)

Exploratory sessions complement scripted cases by targeting areas where tester judgment, not a predefined script, is likely to surface issues — particularly state-management and multi-step-flow interactions.

### Charter 1 — Checkout State Resilience Under Navigation

- **Charter:** Explore the checkout flow's resilience to non-linear navigation (browser back/forward, tab duplication, page refresh, direct URL entry to a later step) to discover state-loss, duplicate-submission, or stale-data bugs.
- **Areas:** Cart → Address → Shipping → Payment → Review → Confirmation, plus browser chrome (back/forward/refresh/new-tab).
- **Timebox:** 60 minutes.
- **Notes / Observations Log:**
  - Does refreshing on Order Review re-fetch the cart, or show stale/cached totals?
  - Does pressing Back after a successful order confirmation resubmit payment if "Place Order" is pressed again? (Linked to BUG-004.)
  - Does opening the confirmation URL directly in a second tab (session still valid) show it as a read-only receipt, or does it allow interaction that could re-trigger side effects?
  - Does a promo code applied in one tab persist/conflict when the same cart is open in a second tab?

### Charter 2 — Payment Form Edge Behavior & Error Recovery

- **Charter:** Explore the payment form's behavior under rapid, unusual, or interrupted input to discover validation gaps, race conditions, and unclear error recovery paths.
- **Areas:** Card number/CVV/expiry fields, wallet selection toggle, "Place Order" button, gateway timeout/decline simulation.
- **Timebox:** 45 minutes.
- **Notes / Observations Log:**
  - Rapidly double-clicking "Place Order" — does it queue two submissions? (Linked to BUG-006.)
  - Pasting a card number with spaces, dashes, or copied from a PDF (invisible characters) — does validation still work?
  - Switching from Card to Wallet and back — do previously entered card details persist safely, or leak into the DOM after switching away?
  - Forcing a simulated gateway timeout — does the UI show a clear retry option, or does it hang/silently fail while a charge may have gone through?
  - Typing non-numeric characters into CVV rapidly, then pasting an emoji or long string — any client-side crash or console error?

### Charter 3 — Promo Code & Pricing Consistency Across the Funnel

- **Charter:** Explore whether displayed pricing (subtotal, discount, shipping, tax, total) remains internally consistent as the customer moves forward and backward across Cart → Promo → Shipping → Review → Confirmation, hunting for silent recalculation errors.
- **Areas:** Promo code entry/removal, shipping method changes after a promo is applied, currency/locale display, order confirmation summary vs. cart summary.
- **Timebox:** 45 minutes.
- **Notes / Observations Log:**
  - Apply a promo, then change shipping method — does the discount recalculate correctly if the promo is percentage-based vs. flat-amount?
  - Apply a promo, go back to Cart, change item quantity, return to Review — is the promo re-validated (e.g., minimum spend requirement) or does a stale discount persist? (Linked to BUG-001.)
  - Switch account locale/currency (if supported) mid-session — does Order Review and Confirmation show matching currency symbols and converted totals? (Linked to BUG-003.)
  - Remove a promo code after applying it — does the total correctly revert with no residual rounding artifact (e.g., $0.01 left over)?

---

## 5. Traceability Note

Every EP partition, BVA boundary, and decision table rule above is realized as one or more concrete test cases in `03-test-cases/Test-Cases.csv` (see the `Type` column for Positive/Negative/Boundary/Security/Usability classification) and mapped to requirements in `05-traceability/Requirements-Traceability-Matrix.md`.
