# Requirements Traceability Matrix — ShopSphere Checkout & Payment

**Document ID:** RTM-CHK-001
**Related:** `01-test-plan/Test-Plan.md`, `03-test-cases/Test-Cases.csv`, `04-bug-reports/`

## Purpose

This matrix maps each functional/business requirement for the Checkout & Payment feature to the test case(s) that exercise it, demonstrating coverage and enabling impact analysis (e.g., "if requirement REQ-009 changes, which test cases must be re-run?"). Requirement IDs below are written for this portfolio project to represent a plausible, realistic requirements set for this feature.

## Requirements → Test Case Coverage

| Req ID | Requirement Description | Covered By (TC IDs) | Coverage Status |
|---|---|---|---|
| REQ-001 | The system shall display all cart line items with correct name, unit price, quantity, and line total, and an accurate subtotal, before checkout begins. | TC-CART-001, TC-CART-002, TC-CART-003 | Covered |
| REQ-002 | The system shall restrict line-item quantity to a maximum of 10 units per SKU. | TC-CART-004 | Covered |
| REQ-003 | The system shall prevent a shopper from proceeding to checkout with an empty cart. | TC-CART-005 | Covered |
| REQ-004 | The system shall require Name, Address Line 1, City, State/Region, Postal Code, Phone, and Country for a valid shipping address, and shall block submission if any required field is missing. | TC-ADDR-001, TC-ADDR-002 | Covered |
| REQ-005 | The system shall validate postal code format according to the selected country's format rules before accepting a shipping address. | TC-ADDR-003, TC-ADDR-006 | Covered (defect found — BUG-002) |
| REQ-006 | The system shall validate phone number format, accepting only digits, spaces, dashes, parentheses, and an optional leading "+". | TC-ADDR-004 | Covered |
| REQ-007 | The system shall allow a returning shopper to select a previously saved address instead of re-entering details. | TC-ADDR-005 | Covered |
| REQ-008 | The system shall sanitize all free-text address input to prevent injection or script execution when the data is stored or later displayed. | TC-ADDR-007, TC-SEC-002, TC-SEC-004 | Covered |
| REQ-009 | The system shall offer free standard shipping when the merchandise subtotal is greater than or equal to $50.00, and shall charge standard shipping fees below that threshold. | TC-SHIP-001, TC-SHIP-002 | Covered |
| REQ-010 | The system shall recalculate the shipping cost, delivery estimate, and order total immediately when the shopper changes the selected shipping method. | TC-SHIP-003, TC-SHIP-005 | Covered |
| REQ-011 | The system shall apply a flat $4.00 international handling surcharge for shipping addresses outside the United States, displayed as a distinct line item. | TC-SHIP-004 | Covered |
| REQ-012 | The system shall restrict ShopSphere Wallet as a payment method to shipping addresses within the United States, enforced both in the UI and at the API level. | TC-PAY-007, TC-PAY-008, TC-PAY-012 | Covered (defect risk validated — no defect found; server-side enforcement confirmed present) |
| REQ-013 | The system shall validate card number, expiry date, and CVV format/length according to the detected card network (Visa/Mastercard: 16 digits, 3-digit CVV; Amex: 15 digits, 4-digit CVV) before submitting to the payment gateway. | TC-PAY-001, TC-PAY-002, TC-PAY-003, TC-PAY-004, TC-PAY-006 | Covered (defect found — BUG-005) |
| REQ-014 | The system shall present a clear, non-technical error message and preserve entered (non-sensitive) checkout data when the payment gateway declines a transaction (CVV mismatch, insufficient funds) or times out. | TC-PAY-005, TC-PAY-009, TC-PAY-010 | Covered |
| REQ-015 | The system shall allow a shopper to reuse a saved, tokenized payment method without re-displaying the full card number. | TC-PAY-011 | Covered |
| REQ-016 | The system shall validate promo codes for format, existence, expiry, and per-account single-use status, and shall never apply the same single-use code twice for one account. | TC-PROMO-001 through TC-PROMO-006 | Covered (defect found — BUG-001) |
| REQ-017 | The system shall recalculate and display an accurate order total (subtotal − discount + shipping + surcharge + tax) at Order Review, matching the sum of all displayed component lines. | TC-REV-001, TC-SHIP-005 | Covered |
| REQ-018 | The system shall allow the shopper to return to and edit any prior checkout step from Order Review without losing previously entered valid data in other steps. | TC-REV-002 | Covered |
| REQ-019 | The system shall display the order total and currency at Confirmation identically to what was shown and accepted at Order Review, and shall charge exactly that amount. | TC-REV-003, TC-CONF-005 | Covered (defect found — BUG-003) |
| REQ-020 | The system shall require explicit Terms & Conditions acceptance before an order can be submitted. | TC-REV-005 | Covered |
| REQ-021 | The system shall prevent duplicate order submission from a single checkout attempt, including rapid double-click, browser back navigation, and page refresh after successful submission. | TC-REV-004, TC-CONF-003, TC-CONF-004 | Covered (defects found — BUG-004, BUG-006) |
| REQ-022 | The system shall generate a unique order number for every successfully placed order and trigger exactly one confirmation email per order. | TC-CONF-001, TC-CONF-002 | Covered |
| REQ-023 | The system shall independently validate and compute all pricing (item prices, totals) server-side, ignoring or rejecting any client-supplied pricing values in the checkout submission request. | TC-SEC-001 | Covered |
| REQ-024 | The system shall handle session expiration during checkout by prompting re-authentication rather than allowing submission on an invalid session, while preserving cart contents where possible. | TC-SEC-003 | Covered |

## Coverage Summary

- **Total requirements:** 24
- **Requirements with at least one mapped test case:** 24 (100%)
- **Requirements with an associated open defect found during testing:** REQ-005 (BUG-002), REQ-013 (BUG-005), REQ-016 (BUG-001), REQ-019 (BUG-003), REQ-021 (BUG-004, BUG-006) — 5 of 24 requirements (~21%) surfaced at least one defect, concentrated in the payment/promo/order-finalization areas, consistent with the risk assessment in the Test Plan (§12) that identified duplicate-submission and pricing-consistency as the highest-risk areas.
- **Requirements with zero test coverage:** None. This satisfies the Test Plan exit criterion (§9): "Requirements Traceability Matrix shows 100% of in-scope requirements mapped to at least one executed test case."

## How to Use This Matrix

- **Forward traceability** (requirement → tests): confirms every requirement has verification coverage, and shows where coverage is concentrated vs. thin (e.g., REQ-016 and REQ-021 each have several test cases because they are decision-table-driven, high-risk areas).
- **Backward traceability** (test → requirement): every test case in `03-test-cases/` can be traced back to the requirement(s) it was designed to verify, which supports impact analysis when a requirement changes (re-run only the mapped subset) and supports auditing "why does this test case exist?"
- **Defect correlation**: cross-referencing the "associated open defect" column against `04-bug-reports/` shows this RTM is not just a paperwork exercise — it is the mechanism that demonstrates test cases found real, releasable-risk defects.
