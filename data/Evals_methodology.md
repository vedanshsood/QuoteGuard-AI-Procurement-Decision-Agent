# QuoteGuard Evaluation Methodology

## Overview

QuoteGuard is evaluated using a purpose-built benchmark of **50 synthetic supplier quotation scenarios**.

The evaluation set was designed around the actual responsibility of the agent: determining whether a supplier quotation can proceed through a procurement workflow based on document validity, arithmetic correctness, commercial conditions, procurement policy, and safety constraints.

Rather than evaluating only whether the model produces a plausible natural-language response, each case defines an expected structured procurement outcome. This allows the system to measure whether the agent reaches the correct operational decision.

---

## Evaluation Objective

The primary question behind the evaluation design was:

> Can QuoteGuard reliably distinguish between quotations that can proceed automatically, quotations requiring human judgement, quotations requiring supplier clarification, and quotations that must not proceed?

The benchmark therefore evaluates both normal procurement activity and failure conditions.

Each evaluation case contains:

- A synthetic supplier quotation
- An expected procurement decision
- An expected primary issue
- An expected severity
- An expected downstream action
- Scenario tags describing the condition being tested

The expected answer is used only by the evaluation harness and is never provided to the agent.

---

# Decision Taxonomy

The benchmark is organized around QuoteGuard's four possible procurement decisions.

| Decision | Interpretation | Next Action |
|---|---|---|
| `APPROVE` | Quotation passes the required checks | `CREATE_DRAFT_PO` |
| `REVIEW` | Quotation contains a commercial condition requiring human judgement | `SUPERVISOR_REVIEW` |
| `REQUEST_CLARIFICATION` | Supplier must correct, complete, or clarify information | `CONTACT_SUPPLIER` |
| `REJECT` | Document or quotation must not continue through the workflow | `STOP_PROCESSING` |

This taxonomy was intentionally designed to separate different types of procurement problems instead of treating every abnormal quotation as a rejection.

---

# Evaluation Set Design

The 50 cases were divided into four major groups.

## 1. APPROVE Cases

These cases represent quotations that should be able to proceed through the standard procurement workflow.

They are used to test whether QuoteGuard can recognize valid quotations without unnecessarily escalating them.

The cases include variations in:

- Supplier
- Product
- Quantity
- Quotation value
- Delivery terms
- Warranty
- Payment terms
- Quotation validity period
- Discounts
- Tax
- Shipping charges

The purpose of these variations is to prevent the agent from learning a single fixed representation of an acceptable quotation.

A successful APPROVE case requires the relevant calculation, validity, and procurement-policy checks to pass.

---

## 2. REVIEW Cases

REVIEW scenarios represent quotations that may still be commercially legitimate but contain conditions requiring human procurement judgement.

These scenarios were included because procurement decisions are not always binary.

Examples include:

- High-value procurement
- Unusually large quantities
- Non-standard payment conditions
- High upfront payment requirements
- Full advance payment
- Below-standard warranty
- Commercial cancellation or restocking conditions
- Other policy exceptions

The expected result is:

```text
REVIEW
→ SUPERVISOR_REVIEW
```

These cases test whether the agent avoids both extremes:

- automatically approving a commercial exception, or
- incorrectly rejecting a potentially legitimate quotation.

This category was particularly useful during development because several early false approvals occurred in REVIEW scenarios.

---

## 3. REQUEST_CLARIFICATION Cases

These scenarios contain information that is incomplete, inconsistent, ambiguous, or apparently incorrect but potentially correctable by the supplier.

Examples include:

- Line-total mismatch
- Subtotal mismatch
- Incorrect tax calculation
- Incorrect discount calculation
- Missing warranty
- Missing currency
- Missing required quotation information

The expected result is:

```text
REQUEST_CLARIFICATION
→ CONTACT_SUPPLIER
```

These scenarios test whether QuoteGuard can distinguish a correctable quotation problem from a terminal rejection.

For example, an arithmetic inconsistency should generally result in supplier clarification rather than automatic rejection.

---

## 4. REJECT Cases

The REJECT group contains cases where the submitted quotation or document should not continue through the procurement workflow.

These scenarios test the safety boundary of the system.

Examples include:

- Invalid document types
- Unrelated documents
- Severely expired quotations
- Invalid future-dated quotations
- Unreadable documents
- Procurement-scope mismatch

The expected result is:

```text
REJECT
→ STOP_PROCESSING
```

These cases are especially important because incorrectly approving one represents a more serious failure than unnecessarily escalating a valid quotation.

---

# Ground-Truth Design

Each evaluation case contains an expected structured result.

Conceptually:

```json
{
  "expected": {
    "decision": "REVIEW",
    "issue": "HIGH_UPFRONT_PAYMENT",
    "severity": "HIGH",
    "next_action": "SUPERVISOR_REVIEW"
  }
}
```

The ground truth defines four separate dimensions.

### Decision

The overall procurement disposition:

```text
APPROVE
REVIEW
REQUEST_CLARIFICATION
REJECT
```

### Issue

The canonical primary issue expected from the quotation.

Examples:

```text
NONE
SUBTOTAL_MISMATCH
HIGH_UPFRONT_PAYMENT
MISSING_CURRENCY
UNREADABLE_DOCUMENT
PROCUREMENT_SCOPE_MISMATCH
```

Using canonical issue codes allows evaluation to go beyond natural-language similarity.

### Severity

The expected severity of the identified issue:

```text
NONE
LOW
MEDIUM
HIGH
CRITICAL
```

### Next Action

The expected operational consequence of the decision:

```text
CREATE_DRAFT_PO
SUPERVISOR_REVIEW
CONTACT_SUPPLIER
STOP_PROCESSING
```

---

# Evaluation Leakage Prevention

A deliberate separation exists between **agent-visible quotation data** and **evaluation ground truth**.

The complete evaluation case may contain:

```text
quotation
expected
tags
```

However, QuoteGuard's `get_quotation()` tool exposes only the quotation.

The agent therefore cannot access:

```text
expected
tags
```

during execution.

The expected result is accessed only after agent execution by the evaluation harness.

This prevents the benchmark from becoming a lookup task and ensures that the agent must derive its answer from:

```text
Quotation
     +
Validation Tools
     +
Procurement Policy
     +
Agent Reasoning
```

---

# Tool-Based Evaluation

QuoteGuard does not make its decision solely from the raw quotation.

The evaluation tests the complete agent workflow:

```text
Evaluation Case
      │
      ▼
get_quotation()
      │
      ▼
Supplier Quotation
      │
      ├───────────────┐
      ▼               ▼
validate_         check_quotation_
calculations()    validity()
      │               │
      └───────┬───────┘
              ▼
     check_procurement_
          policy()
              │
              ▼
       Agent Decision
              │
              ▼
        Guardrails
              │
              ▼
submit_procurement_decision()
              │
              ▼
 Compare with Ground Truth
```

This evaluates the agent as a complete system rather than testing only the underlying LLM.

---

# Why Multiple Output Metrics Are Used

Decision accuracy alone is insufficient for evaluating a procurement agent.

For example, an agent might correctly return:

```text
REVIEW
```

but identify the wrong issue or severity.

The evaluation therefore measures several dimensions independently.

## Decision Accuracy

Measures whether the agent selected the expected procurement decision.

```text
Actual Decision == Expected Decision
```

## Issue Accuracy

Measures whether the agent identified the expected primary issue.

```text
Actual Issue == Expected Issue
```

## Next-Action Accuracy

Measures whether the resulting procurement action matches the expected workflow action.

```text
Actual Action == Expected Action
```

## Severity Accuracy

Measures whether the issue was assigned the expected risk severity.

```text
Actual Severity == Expected Severity
```

## Complete Task Success

This is the strictest evaluation metric.

A task is considered completely successful only when the required structured output matches the expected result across the evaluated dimensions.

This prevents a correct high-level decision from hiding errors elsewhere in the procurement output.

---

# False Approval Analysis

False approvals were treated as a dedicated evaluation concern.

A false approval occurs when:

```text
Expected Decision != APPROVE
Actual Decision   == APPROVE
```

For example:

```text
Expected: REVIEW
Actual:   APPROVE
```

or:

```text
Expected: REJECT
Actual:   APPROVE
```

These failures are important because they allow a quotation requiring intervention to incorrectly continue toward purchase-order preparation.

During development, false-approval analysis helped identify missing or insufficient policy checks involving scenarios such as:

- unusually large quantities,
- cancellation/restocking conditions,
- procurement-scope mismatches.

The corresponding validation and policy logic was then strengthened.

---

# Evaluation-Driven Development

The evaluation suite was not used only as a final benchmark.

It was also used iteratively during system development.

The development cycle was:

```text
Design Evaluation Cases
        ↓
Run QuoteGuard
        ↓
Inspect Failures
        ↓
Inspect Tool Evidence
        ↓
Identify Root Cause
        ↓
Modify Policy / Tools / Guardrails
        ↓
Re-run Evaluation
```

For example, diagnostic analysis distinguished between:

- an LLM choosing the wrong decision despite correct evidence,
- a validation tool failing to detect an issue,
- a procurement-policy rule being absent,
- inconsistent canonical issue naming,
- incorrect severity assignment,
- guardrail/evidence inconsistencies.

This distinction was important because improving the prompt alone would not fix failures caused by missing deterministic evidence.

---

# Guardrail Evaluation

The benchmark also exercises QuoteGuard's guardrail layer.

Guardrails verify constraints such as:

- decision-to-action consistency,
- evidence requirements for approval,
- consistency between tool evidence and the submitted disposition,
- allowed categorical outputs,
- purchase-order autonomy boundaries.

An `APPROVE` decision therefore requires supporting validation evidence rather than relying solely on the model's judgement.

QuoteGuard may prepare:

```text
CREATE_DRAFT_PO
```

but is not authorized to autonomously issue a real purchase order.

---

# PDF Evaluation Extension

The structured benchmark is complemented by a PDF quotation workflow.

A supplier quotation PDF is:

```text
PDF
 ↓
Text Extraction
 ↓
Structured Quotation Extraction
 ↓
QuoteGuard Input
 ↓
Validation Tools
 ↓
Procurement Policy
 ↓
Agent Decision
 ↓
Guardrails
 ↓
Final Procurement Assessment
```

This tests whether the same agent architecture can operate on a more realistic document-based input rather than only pre-structured benchmark cases.

The PDF test is kept separate from the 50-case benchmark so that uploaded documents do not modify the benchmark dataset or its ground truth.

---

# Limitations of the Evaluation Set

The benchmark is intentionally controlled and synthetic.

This provides reproducibility and allows individual procurement conditions to be tested precisely, but it also introduces limitations.

The 50 cases do not represent the complete diversity of real enterprise procurement.

Real quotations may contain:

- complex tables,
- multiple currencies,
- inconsistent terminology,
- scanned documents,
- handwritten information,
- unusual tax structures,
- supplier-specific contract language,
- multi-page terms and conditions,
- multiple interacting commercial exceptions.

The current evaluation should therefore be interpreted as a controlled benchmark of QuoteGuard's decision architecture rather than proof of production-level procurement reliability.

A production evaluation would require a larger and more diverse dataset containing anonymized real-world procurement documents and additional adversarial scenarios.

---

# Evaluation Philosophy

The evaluation methodology was designed around three principles:

**1. Evaluate operational behaviour, not just language quality.**

The important question is whether QuoteGuard takes the correct procurement action, not whether its response merely sounds reasonable.

**2. Make unsafe decisions measurable.**

False approvals, invalid actions, missing evidence, and guardrail violations are explicitly observable.

**3. Diagnose the complete agent system.**

Failures may originate from the LLM, tools, policy definitions, extraction pipeline, or guardrails. Evaluation therefore considers the complete workflow rather than treating every failure as a model error.

This approach allowed the benchmark to serve both as a measurement framework and as a mechanism for systematically improving QuoteGuard.
