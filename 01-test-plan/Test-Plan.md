# Test Plan: ShopSphere — Checkout & Payment Flow

**Document ID:** TP-CHK-001
**Version:** 1.0
**Author:** Mohamed Elaadly (QA Engineer)
**Status:** Approved for execution
**Standard reference:** Structured per IEEE 829-1998 Test Plan outline

---

## 1. Introduction

ShopSphere is a fictional online retail platform. This test plan covers the **Checkout & Payment** flow, the sequence a shopper follows from a populated cart through to a confirmed order. This flow is the single highest-risk area of the product: it touches money movement, third-party payment processors, tax/shipping calculation, and customer PII (shipping address, contact details). Defects here have direct revenue impact (abandoned carts, duplicate charges, incorrect totals) and reputational/compliance impact (payment data handling, price accuracy regulations).

This document defines what will be tested, how, by whom, under what conditions, and what "done" means for the release candidate `ShopSphere Checkout v2.4`.

## 2. Scope

### 2.1 In Scope

The Checkout & Payment flow, defined as the following screens/steps in sequence:

1. **Cart Review** — line items, quantities, subtotal, applied promos, "Proceed to Checkout" entry point.
2. **Shipping Address Entry** — new address form, saved address selection, address book validation.
3. **Shipping Method Selection** — carrier/speed options, dynamic cost and ETA display, free-shipping threshold logic.
4. **Payment Method Entry** — credit/debit card form, digital wallet (ShopSphere Wallet) option, saved payment methods.
5. **Promo Code Application** — code entry, validation, discount calculation, stacking rules.
6. **Order Review** — final summary (items, shipping, tax, discount, grand total) prior to submission.
7. **Order Confirmation** — post-submission confirmation page, order number generation, confirmation email trigger (email content/delivery not tested here, only trigger).

### 2.2 Out of Scope

- Product browsing, search, and product detail pages (covered under a separate Catalog test plan).
- Post-purchase flows: order tracking, returns, refunds, customer support ticketing.
- Actual settlement with real payment processors (a sandboxed/mock payment gateway is used — see §7).
- Marketing/CRM triggers (abandoned cart emails, retargeting pixels).
- Native mobile app checkout (this cycle covers responsive web only; mobile app has its own test plan).
- Load/performance/stress testing (covered by a separate Performance Test Plan; this plan includes only light usability/timing observations).
- Full penetration testing (this plan includes "security-lite" functional checks only, not a formal pen test).

## 3. Objectives

- Verify the checkout flow correctly calculates and displays cart, shipping, tax, discount, and total amounts at every step, and that the final charged amount matches the displayed order review amount.
- Verify all input fields (address, card, promo code) enforce correct validation, including boundary and negative cases.
- Verify payment processing handles success, decline, timeout, and error paths gracefully without double-charging or losing order state.
- Verify promo code logic (validity window, usage limits, stacking rules) behaves per business rules.
- Verify the flow is usable and consistent across supported browsers/viewports.
- Identify functional, data-integrity, and security-lite defects before release, and provide traceability from requirements to test coverage.

## 4. Test Items

| Item | Version | Description |
|---|---|---|
| ShopSphere Web Checkout | v2.4 (release candidate) | Cart, address, shipping, payment, promo, review, confirmation screens |
| Checkout API (mocked backend) | v2.4-rc | Order calculation, tax, discount, order creation endpoints |
| Payment Gateway Sandbox | Mock Gateway v3 | Simulated card/wallet processor with configurable test outcomes (approve/decline/timeout) |
| Order Confirmation Service | v2.4-rc | Order number generation, confirmation page render, email trigger (trigger only) |

## 5. Features to Be Tested

- Cart line-item review (quantity edit, remove item, subtotal recalculation) as entry state into checkout.
- Shipping address form: required field validation, format validation (postal code, phone), saved address reuse, address editing mid-checkout.
- Shipping method selection: dynamic pricing, ETA display, free-shipping threshold behavior.
- Payment: card entry (number, expiry, CVV, cardholder name), card type detection, digital wallet flow, saved payment methods, decline/timeout handling.
- Promo code: valid/invalid/expired/already-used codes, single-use enforcement, stacking with other discounts, code removal.
- Order review: accuracy of all line items and totals, ability to navigate back and edit, currency display correctness.
- Order confirmation: order number uniqueness/format, summary accuracy against what was charged, confirmation trigger fired exactly once.
- Cross-cutting: browser/session back-button behavior, session timeout mid-checkout, double-submit prevention.

## 6. Features Not to Be Tested

- Warehouse/inventory allocation logic triggered after order creation.
- Tax engine's internal jurisdiction rules (tax rate correctness is assumed from a stubbed tax service; only that the returned tax value is correctly *displayed and summed* is tested).
- Third-party wallet provider's own authentication UI (only ShopSphere's integration points are tested).
- Accessibility conformance audit (WCAG) — noted as a risk/gap in §11, recommended as a follow-up cycle.

## 7. Test Approach

A risk-based, multi-technique manual testing approach is used, layered as follows:

| Technique | Purpose | Example application |
|---|---|---|
| **Functional / positive testing** | Confirm the flow works end-to-end for the "happy path" and its common variants | Complete checkout with saved card, complete checkout with wallet |
| **Negative testing** | Confirm invalid input and failure paths are handled and surfaced clearly | Invalid card number, expired promo code, decline response from gateway |
| **Boundary Value Analysis** | Probe edges of numeric/length constraints | Free-shipping threshold ($49.99 vs $50.00), max quantity per line item, card number length limits |
| **Equivalence Partitioning** | Reduce input space to representative classes | Promo code formats, postal code formats by class of validity |
| **Decision-table-based testing** | Validate combinational business rules | Payment method × shipping country × promo validity → accept/reject/adjust outcome |
| **Usability review** | Catch friction, ambiguous errors, inconsistent states | Error message clarity, field focus/scroll behavior, disabled-state feedback |
| **Security-lite testing** | Catch obvious data-handling and tampering risks without a full pen test | Client-side price tampering, SQLi-pattern strings in text fields, session timeout mid-checkout |
| **Cross-browser / responsive testing** | Confirm consistent behavior across supported environments | Chrome, Firefox, Safari, Edge; desktop + tablet + mobile-web viewport |
| **Exploratory testing (session-based)** | Uncover issues structured cases miss | Charters defined in `02-test-design/Test-Design-Techniques.md` |

Test design techniques are documented in detail, with worked examples, in `02-test-design/Test-Design-Techniques.md`. Concrete test cases derived from this approach are recorded in `03-test-cases/`.

## 8. Entry Criteria

- Checkout & Payment feature is code-complete and deployed to the QA/staging environment.
- Mock payment gateway is configured with test cards for approve, decline, insufficient-funds, and timeout scenarios.
- Test data set is available: seeded user accounts, saved addresses, saved payment methods, promo codes (valid, expired, used, invalid).
- Test Plan and Test Cases are reviewed and approved.
- No P1 (blocker) defects open against a prior smoke test of the build.

## 9. Exit Criteria

- 100% of P1 test cases executed, with 100% pass rate (no open P1/P2 defects).
- ≥ 95% of P2 test cases executed, with ≥ 95% pass rate.
- ≥ 90% of all planned test cases (P1–P3) executed.
- All identified defects are triaged; no open Critical or High severity defects remain unresolved or without an accepted risk sign-off.
- Requirements Traceability Matrix shows 100% of in-scope requirements mapped to at least one executed test case.
- Test summary report reviewed and sign-off obtained from QA Lead and Product Owner.

### 9.1 Suspension / Resumption Criteria

- Testing is suspended if a P1 defect blocks progress through a core checkout step (e.g., cannot reach Order Review from Payment).
- Testing resumes once a fix is deployed to the test environment and the blocking defect is verified closed.

## 10. Test Environment

| Component | Detail |
|---|---|
| Application environment | QA/Staging (`checkout-staging.shopsphere.example`) |
| Browsers | Chrome (latest, latest-1), Firefox (latest), Safari (latest), Edge (latest) |
| Viewports | Desktop (1920×1080, 1366×768), Tablet (768×1024), Mobile-web (390×844) |
| Payment gateway | Sandboxed mock gateway with configurable test card responses |
| Test accounts | 5 seeded shopper accounts with varying cart contents, saved addresses, and saved payment methods |
| Test data | Promo code set (valid, expired, single-use-consumed, invalid-format); card set (valid, expired, invalid Luhn, insufficient funds trigger) |
| Defect tracking | Markdown bug reports in `04-bug-reports/` (portfolio simulation of a Jira-style tracker) |
| Test management | CSV/Markdown test case repository in `03-test-cases/` (portfolio simulation of a TestRail-style tool) |

## 11. Roles & Responsibilities

| Role | Responsibility | Owner |
|---|---|---|
| QA Lead / Test Designer | Own test plan, test design, execution, defect reporting, traceability, sign-off recommendation | Mohamed Elaadly |
| Product Owner | Approve scope, clarify business rules (e.g., promo stacking, free-shipping threshold), review exit report | Product stakeholder (fictional) |
| Development Lead | Provide build, fix defects, support root-cause triage | Engineering stakeholder (fictional) |
| DevOps | Maintain QA/staging environment and mock payment gateway availability | Platform stakeholder (fictional) |

## 12. Risks & Mitigations

| # | Risk | Likelihood | Impact | Mitigation |
|---|---|---|---|---|
| R1 | Mock payment gateway behaves differently from the real processor (e.g., timeout semantics) | Medium | High | Document gateway assumptions explicitly; flag as a residual risk in the exit report; recommend a contract/integration test pass against a processor sandbox before production launch |
| R2 | Currency/locale edge cases (e.g., non-USD formatting) are under-represented in test data | Medium | Medium | Add explicit multi-currency test cases (see TC-CONF series); track as a known coverage gap otherwise |
| R3 | Race conditions around double-submit / back-button are hard to reproduce deterministically | High | High | Dedicate explicit test cases and an exploratory charter to this area; treat any reproduction as Critical regardless of frequency |
| R4 | Accessibility (WCAG) issues are out of formal scope but may exist | Medium | Medium | Log accessibility observations opportunistically during usability testing; recommend a dedicated a11y audit cycle |
| R5 | Test schedule compression could reduce cross-browser coverage | Medium | Medium | Prioritize Chrome + Safari (highest traffic share) if time is cut; document reduced coverage in exit report |
| R6 | Promo code business rules are ambiguous/underspecified for edge cases (e.g., promo + wallet stacking) | Low | Medium | Confirm rules with Product Owner before finalizing decision table; log assumptions in Test Design doc |

## 13. Schedule & Milestones

| Milestone | Target |
|---|---|
| Test Plan review & sign-off | Day 1 |
| Test design (EP/BVA/Decision Table/Charters) complete | Day 2–3 |
| Test case authoring complete | Day 3–4 |
| Test execution cycle 1 (full pass) | Day 5–7 |
| Defect triage & retest | Day 8–9 |
| Test execution cycle 2 (regression on fixes) | Day 9–10 |
| Exploratory testing sessions | Day 10 |
| Exit report & sign-off | Day 11 |

## 14. Deliverables

- `01-test-plan/Test-Plan.md` (this document)
- `02-test-design/Test-Design-Techniques.md` — EP, BVA, Decision Table, Exploratory Charters
- `03-test-cases/Test-Cases.csv` and `.md` — full test case repository
- `04-bug-reports/BUG-001.md` … `BUG-006.md` — defect reports
- `05-traceability/Requirements-Traceability-Matrix.md` — requirement-to-test-case coverage
- Test summary/exit report (verbal/sign-off equivalent captured in RTM coverage summary)

## 15. Approval

| Name | Role | Decision |
|---|---|---|
| Mohamed Elaadly | QA Lead | Approved |
| (Product Owner) | Product | Approved (simulated) |
| (Engineering Lead) | Development | Approved (simulated) |
