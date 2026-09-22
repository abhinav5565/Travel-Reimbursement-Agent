# Travel Reimbursement Approval Agent

## Overview

This project implements a lightweight AI-assisted agent for evaluating employee travel reimbursement claims against a defined travel reimbursement policy.

The system evaluates:

- Expense eligibility
- Per-diem and category limits
- Receipt requirements
- Approval thresholds
- Submission timeliness
- Policy exceptions and ambiguous cases

It returns one of four decisions:

- `APPROVE`
- `PARTIAL_APPROVE`
- `REJECT`
- `MANUAL_REVIEW`

The notebook evaluates all five sample claims provided in the case study and produces the required structured JSON output.

---

## Architecture

```text
                    Travel Claim
                         |
                         v
                Claim Validation
                         |
                         v
              +---------------------+
              |   Groq LLM Agent    |
              |                     |
              +----------+----------+
                         |
                    Tool Calling
                         |
        +----------------+----------------+
        |                |                |
        v                v                v
 Policy Lookup     Receipt Check     Limit Checker
        |                |                |
        +----------------+----------------+
                         |
             +-----------+-----------+
             |                       |
             v                       v
      Approval Threshold       Timeliness Check
             |                       |
             +-----------+-----------+
                         |
                         v
              Deterministic Decision
                    + Guardrails
                         |
                         v
                 Output Validation
                         |
                         v
                Structured JSON
                         |
                         v
                    Dashboard
```

### Design principle

The LLM is used for **agentic orchestration and tool selection**, while policy-critical calculations and validation are handled deterministically in Python.

This separation reduces the risk of LLM hallucination or arithmetic errors affecting reimbursement decisions.

---

## Tools

| Tool | Purpose |
|---|---|
| `policy_lookup` | Retrieves relevant policy rules and stable `POL-*` references |
| `receipt_completeness_check` | Determines whether required receipts are attached |
| `limit_checker` | Applies meal, lodging, and ground-transport limits |
| `approval_threshold_check` | Determines the applicable approval tier |
| `timeliness_check` | Checks the 30-day submission requirement |
| `output_validator` | Validates the final structured decision |

---

## Decision Logic

### APPROVE

A claim is approved when all items are eligible, required receipts are available, no applicable limits are exceeded, the claim is within approval authority, and it is submitted within the required window.

### PARTIAL_APPROVE

Used when the claim is valid but one or more amounts exceed an applicable policy cap. The reimbursable amount is limited to the applicable cap and the excess is deducted.

### REJECT

Used when all claimed items are ineligible and there is nothing reimbursable.

### MANUAL_REVIEW

Used instead of forcing an automated decision for ambiguity, policy exceptions, missing required receipts, high-value claims, late submissions, or conflicting information.

---

## Sample Claims

The notebook evaluates the five claims supplied in the assignment:

| Claim | Scenario | Expected Decision |
|---|---|---|
| `CLM-001` | 2-day industry conference | `APPROVE` |
| `CLM-002` | Weekend hotel stay with spa/minibar | `REJECT` |
| `CLM-003` | Client site visit with lodging over limit | `PARTIAL_APPROVE` |
| `CLM-004` | International vendor negotiation | `MANUAL_REVIEW` |
| `CLM-005` | Client dinner with missing receipt | `MANUAL_REVIEW` |

Expected reimbursement summary:

| Claim | Approved | Deducted | Decision |
|---|---:|---:|---|
| `CLM-001` | $1,110 | $0 | `APPROVE` |
| `CLM-002` | $0 | $380 | `REJECT` |
| `CLM-003` | $840 | $100 | `PARTIAL_APPROVE` |
| `CLM-004` | Manual review | Manual review | `MANUAL_REVIEW` |
| `CLM-005` | Manual review | Manual review | `MANUAL_REVIEW` |

For Manual Review cases, the notebook does not force a final reimbursement amount because the exception requires human resolution.

---

## Running the Notebook

### 1. Clone the repository

```bash
git clone <repository-url>
cd travel-reimbursement-agent
```

### 2. Install dependencies

```bash
pip install pandas matplotlib openai
```

### 3. Open the notebook

Open:

```text
abhinavkumar.ipynb
```

using Jupyter Notebook, JupyterLab, or VS Code.

### 4. Run the notebook

The core deterministic evaluation can run without an LLM.

The notebook also contains an **optional Groq LLM / Agentic Flow** section.

If you want to demonstrate the agentic workflow, run that optional setup cell.

---

## Groq Configuration

The optional agentic flow uses Groq through its OpenAI-compatible API.


Then restart the Jupyter kernel and run the notebook.

### Option B — Notebook prompt

If `GROQ_API_KEY` is not present, the optional setup cell prompts for the key securely using `getpass`.


---

## Agentic Flow

When the optional Groq flow is enabled:

```text
Claim
  |
  v
Groq Agent
  |
  +--> policy_lookup
  |
  +--> receipt_completeness_check
  |
  +--> limit_checker
  |
  +--> approval_threshold_check
  |
  +--> timeliness_check
  |
  v
Tool Results
  |
  v
Agent Reasoning
  |
  v
Deterministic Policy Guardrails
  |
  v
Validated Decision
```

The model can select and invoke relevant tools, while Python functions perform policy-critical calculations.

---

## Output Format

The final notebook cell produces a JSON array containing one object per claim.

Each object contains exactly:

```text
claim_id
decision
approved_amount
deducted_amount
missing_docs
policy_refs
confidence
explanation
tools_used
```

---

## Dashboard

The notebook includes a minimal data-driven dashboard generated from the actual evaluation results, including decision breakdown and approved/deducted amounts.

---

## Reliability & Guardrails

Key safeguards include:

1. Policy rules use stable `POL-*` identifiers.
2. Limit calculations are deterministic.
3. Receipt requirements are checked programmatically.
4. Approval thresholds are evaluated programmatically.
5. Late claims are routed to Manual Review.
6. Policy exceptions are routed to Manual Review.
7. Final output is schema-validated.
8. The system can fall back to deterministic evaluation if the optional LLM flow is unavailable.

---

## Assumptions and Limitations

This is a lightweight case-study prototype rather than a production enterprise reimbursement platform.

### Assumptions

- The provided policy is the authoritative policy source.
- The five supplied claims are the evaluation dataset.
- Claim data is already structured.
- Currency is USD.
- Receipt attachment status is supplied with each claim.

### Limitations

- No real employee or company data is used.
- No external expense-management or HR system is integrated.
- Receipt images are not OCR-processed.
- The policy is represented as structured mock data rather than a production document-retrieval system.
- Authentication, authorization, persistence, and enterprise audit storage are outside the scope of this prototype.

### Potential Improvements

- Document/OCR-based receipt extraction
- Enterprise policy RAG
- Persistent audit storage
- Human approval workflow
- Duplicate-claim detection
- Authentication and authorization
- Monitoring and evaluation
- Model/prompt versioning
- MCP-based external tool integrations

---


The notebook is the primary deliverable and contains the complete runnable implementation.

---

