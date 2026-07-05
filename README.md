# QA Manual Testing Portfolio — ShopSphere Checkout & Payment

**Author:** Mohamed Elaadly — ISTQB CTFL, CTAL-TAE, CT-PT, CT-AuT
**Type:** Manual QA methodology portfolio (documentation-driven, not a code/automation repo)
**License:** MIT

## Purpose of This Repository

Most public QA portfolios are automation-code repositories: a Selenium/Playwright suite, a few `it()` blocks, a CI badge. That's valuable, but it doesn't demonstrate the discipline that automation code sits on top of: **deciding what to test, why, in what order, and how to prove coverage and communicate what was found.**

This repository is a deliberately **documentation-first** artifact set. It simulates the full manual QA lifecycle a senior QA engineer would produce for a real, high-risk feature — from test planning through structured test design, executable test cases, requirements traceability, and defect reporting — without any automation code. It is meant to be read, not run.

## The Scenario

The feature under test is the **Checkout & Payment flow** of a fictional online store, **ShopSphere**: cart review → shipping address entry → shipping method selection → payment (card and digital wallet) → promo code application → order review → order confirmation.

This scenario was chosen deliberately, not arbitrarily:

- **Checkout/payment flows are the highest-risk surface in e-commerce and fintech.** They combine money movement, third-party integrations (payment gateways), sensitive PII, and multi-step stateful UI — exactly the kind of area where structured test design (not just "click around and see") pays off.
- It's a **generic, public-safe** scenario — no proprietary business logic, no real employer's product or data — while still being realistic enough that every requirement, test case, and defect below is internally consistent and plausible, not toy filler.
- It's directly relevant to the author's background working in **fintech QA**, where payment-flow correctness, defect-reporting quality, and requirements traceability are core, audited deliverables — not nice-to-haves.

## How the Artifacts Connect

The repository is structured so each artifact builds on the previous one, mirroring a real QA workflow:

```
Test Plan  →  Test Design Techniques  →  Test Cases  →  Traceability Matrix
   |                                          |
   |                                          v
   +------------------------------->   Bug Reports
   (scope, risk, entry/exit criteria)   (defects found while executing the test cases)
```

1. **[`01-test-plan/Test-Plan.md`](01-test-plan/Test-Plan.md)** — the IEEE-829-style plan defining scope, objectives, what's in/out, the multi-technique test approach, entry/exit criteria, environment, risks, and schedule. This is the "why are we testing this, and how will we know when we're done" document.
2. **[`02-test-design/Test-Design-Techniques.md`](02-test-design/Test-Design-Techniques.md)** — shows the ISTQB black-box technique work *before* test cases are written: Equivalence Partitioning tables (promo code format, card number format, postal code), Boundary Value Analysis tables (free-shipping threshold, quantity limits, CVV length), a Decision Table combining payment method × shipping country × promo validity, and three session-based Exploratory Testing charters targeting state/navigation risk the scripted cases can't fully cover.
3. **[`03-test-cases/`](03-test-cases/)** — 50 concrete, non-redundant test cases derived directly from the techniques above, covering every module of the flow plus security-lite checks. Provided as `Test-Cases.csv` (spreadsheet/TestRail-importable) and mirrored as `Test-Cases.md` (grouped by module, readable on GitHub).
4. **[`05-traceability/Requirements-Traceability-Matrix.md`](05-traceability/Requirements-Traceability-Matrix.md)** — maps 24 plausible requirements to the test cases that verify them, proving 100% requirement coverage and showing exactly where defects were found relative to the requirement they threatened.
5. **[`04-bug-reports/`](04-bug-reports/)** — six real-style defect reports (`BUG-001` through `BUG-006`), each one discovered by executing a specific test case or exploratory charter from the steps above. This is the payoff: the process didn't just produce paperwork, it found genuine, varied-severity, plausible bugs — a promo double-apply race condition, a postal-code validation gap, a currency/pricing mismatch between review and confirmation, a back-button duplicate-order bug, a CVV field paste-sanitization gap, and a double-submit race on the "Place Order" button.

## Repository Structure

```
qa-manual-testing/
├── README.md
├── LICENSE
├── .gitignore
├── 01-test-plan/
│   └── Test-Plan.md
├── 02-test-design/
│   └── Test-Design-Techniques.md
├── 03-test-cases/
│   ├── Test-Cases.csv
│   └── Test-Cases.md
├── 04-bug-reports/
│   ├── BUG-001.md   (promo code double-apply race condition — High)
│   ├── BUG-002.md   (invalid postal code format accepted — Medium)
│   ├── BUG-003.md   (currency total mismatch, review vs. confirmation — Critical)
│   ├── BUG-004.md   (back-button duplicate order submission — Critical)
│   ├── BUG-005.md   (CVV field accepts non-numeric paste input — Low)
│   └── BUG-006.md   (double-submit via "Place Order" click race — High)
└── 05-traceability/
    └── Requirements-Traceability-Matrix.md
```

## Why This Project (Skills Demonstrated)

- **Structured test design over ad-hoc clicking.** Every test case traces back to an explicit technique — Equivalence Partitioning, Boundary Value Analysis, Decision Tables, or a scoped Exploratory Testing charter — rather than an unstructured list of "things I thought to try." This is the ISTQB CTFL/CTAL black-box design methodology applied to a realistic feature, not a textbook exercise.
- **Risk-based planning.** The Test Plan explicitly identifies and mitigates risk (e.g., duplicate-submission race conditions, currency edge cases, mock-gateway fidelity) rather than treating "test everything equally" as a strategy.
- **Defect reporting quality.** Each bug report separates observed fact (Steps, Expected, Actual) from QA hypothesis (Root Cause / Notes) — a distinction that matters in real triage, where a QA engineer's job is to give developers a fast, credible lead without overclaiming root cause.
- **Requirements traceability as a coverage argument, not paperwork.** The RTM doesn't just list requirement IDs; it shows *where* defects clustered relative to requirements, tying the traceability exercise back to real findings.
- **Domain relevance to fintech/payments QA.** The scenario (checkout/payment, multi-currency handling, decline/timeout handling, duplicate-charge prevention) mirrors the risk profile of production payment systems, aligned with the author's ISTQB CTFL / CTAL-TAE / CT-PT / CT-AuT certifications and fintech QA background.

## Notes on Realism

ShopSphere, its data, and all six defects are fictional and constructed for this portfolio. They are written to be internally consistent (test data, expected results, and bug reports all agree with each other) and representative of real defect classes seen in checkout/payment systems, but no real company's product, data, or incident is represented.
