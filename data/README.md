# QuoteGuard Dataset

This directory contains the synthetic procurement quotation dataset used to develop and evaluate **QuoteGuard**, an AI Procurement Quotation Assessment Agent.

The dataset was created specifically for this project to test whether the agent can correctly assess supplier quotations under normal, exceptional, incomplete, erroneous, and invalid procurement scenarios.

---

## Dataset Purpose

The dataset is designed to evaluate QuoteGuard's ability to:

- Validate supplier quotation information
- Verify financial calculations
- Detect missing or inconsistent information
- Identify commercial and procurement-policy exceptions
- Detect invalid or unacceptable quotations
- Select the appropriate procurement decision
- Trigger the correct downstream procurement action
- Avoid unsafe approvals

The dataset contains both standard quotations and deliberately constructed edge cases.

---

## Dataset Generation

The quotation dataset was generated using:

`Quotation_data_generator.ipynb`

The generator creates structured supplier quotations representing different procurement situations.

The generated cases are then used by:

`Procurement_Agent.ipynb`

for agent execution and evaluation.

The evaluation ground truth is kept separate from the information exposed to the agent. QuoteGuard receives only the quotation itself and must independently determine the appropriate decision using its tools and procurement policy.

---

## Dataset Structure

Each evaluation case contains three main components:

```json
{
  "eval_id": "PQE-2026-001",

  "quotation": {
    "supplier_name": "...",
    "quotation_number": "...",
    "quotation_date": "...",
    "valid_until": "...",
    "currency": "SGD",
    "items": [],
    "subtotal": 0,
    "discount_percent": 0,
    "discount_amount": 0,
    "tax_percent": 9,
    "tax_amount": 0,
    "shipping_cost": 0,
    "additional_charges": 0,
    "total_amount": 0,
    "delivery_terms": "...",
    "warranty": "...",
    "payment_terms": "..."
  },

  "expected": {
    "decision": "...",
    "issue": "...",
    "next_action": "...",
    "severity": "..."
  },

  "tags": []
}
```

### `eval_id`

Unique identifier for each evaluation case.

Example:

```text
PQE-2026-001
```

### `quotation`

Contains the supplier quotation information available to QuoteGuard.

Depending on the test case, quotations may also contain fields such as:

- `document_type`
- `special_terms`
- `procurement_request`
- `extraction_status`
- `extraction_note`

These fields support scenarios such as invalid documents, cancellation conditions, procurement-scope mismatches, and unreadable quotations.

### `expected`

Contains the ground-truth result used by the evaluation harness.

These values are **not exposed to the agent during execution**.

The expected output contains:

| Field | Purpose |
|---|---|
| `decision` | Expected procurement disposition |
| `issue` | Expected primary issue |
| `next_action` | Expected downstream action |
| `severity` | Expected risk severity |

### `tags`

Descriptive metadata identifying the scenario being tested.

Examples include standard quotations, calculation errors, commercial exceptions, missing information, invalid documents, and other procurement edge cases.

---

## Decision Classes

QuoteGuard uses four procurement decisions:

| Decision | Meaning | Required Action |
|---|---|---|
| `APPROVE` | Quotation satisfies required checks | `CREATE_DRAFT_PO` |
| `REVIEW` | Human procurement judgement is required | `SUPERVISOR_REVIEW` |
| `REQUEST_CLARIFICATION` | Supplier correction or clarification is required | `CONTACT_SUPPLIER` |
| `REJECT` | Quotation should not proceed | `STOP_PROCESSING` |

---

## Evaluation Dataset

The benchmark contains **50 evaluation cases**.

They cover four major classes:

- `APPROVE`
- `REVIEW`
- `REQUEST_CLARIFICATION`
- `REJECT`

The cases intentionally include both normal procurement quotations and difficult scenarios designed to test the agent's decision boundaries.

Examples of conditions represented in the dataset include:

- Correct quotations
- Calculation inconsistencies
- Missing required fields
- Non-standard payment terms
- Short warranty periods
- Large procurement quantities
- High-value quotations
- Cancellation/restocking conditions
- Invalid document types
- Severely expired quotations
- Future-dated quotations
- Unreadable documents
- Procurement-scope mismatches

---

## Ground-Truth Separation

A key design requirement is preventing evaluation leakage.

The quotation retrieval tool returns only:

```text
quotation
```

It does **not** expose:

```text
expected
tags
```

to the agent.

Therefore, QuoteGuard must derive its decision from the quotation, validation tools, and procurement policy rather than accessing the expected answer.

This separation allows the dataset to function as a genuine evaluation benchmark rather than a lookup task.

---

## How the Dataset Is Used

The evaluation pipeline is:

```text
Dataset Case
     │
     ▼
Quotation Only
     │
     ▼
QuoteGuard Agent
     │
     ├── get_quotation()
     ├── validate_calculations()
     ├── check_quotation_validity()
     └── check_procurement_policy()
     │
     ▼
Procurement Decision
     │
     ▼
Guardrails
     │
     ▼
Compare Against Ground Truth
     │
     ▼
Evaluation Metrics
```

The resulting agent output is compared against the `expected` block.

---

## Metrics Evaluated

The dataset supports measurement of:

- Decision accuracy
- Issue accuracy
- Next-action accuracy
- Severity accuracy
- Complete task success
- Failure rate
- False approvals
- Guardrail-triggered executions
- Tool usage
- Agent turns
- Latency
- Token consumption
- API cost

**Complete task success** requires the relevant structured output fields to match the expected result, making it stricter than decision accuracy alone.

---

## Important Note

This is a **synthetic academic evaluation dataset**.

Supplier names, quotations, prices, commercial conditions, and procurement scenarios were created for experimentation and evaluation. They should not be interpreted as real supplier quotations or real procurement records.
