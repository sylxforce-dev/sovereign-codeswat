# CodeSWAT — 7B Code Generation Validation Report

## Overview

CodeSWAT was developed around a fundamental engineering observation: a small local coding model can generate usable syntax, but its statistical weights and pre-trained patterns cannot ensure that the generated code strictly honors an exact, formal specification.

The goal of CodeSWAT is **not to make the model perfect**. The goal is to place a deterministic control layer around the model so that generated code is converted into an explicit contract, generated in controlled groups, statically validated, rejected upon boundary violations, and repaired through targeted retries.

This development phase was conducted using:
* **Qwen2.5.1-Coder-7B-Instruct**
* **Q6_K_L GGUF**
* **NVIDIA RTX 5060 Ti 8 GB**
* Fully GPU-resident local inference

The identical 7B checkpoint was evaluated across both baseline unconstrained generation and CodeSWAT contract-governed runs.

---

# 1. Baseline vs. CodeSWAT — Why the Pipeline is Necessary

A raw, unconstrained 7B model cannot be trusted in production environments without an external verification layer.

### Raw / Default 7B Generation (Unconstrained)
* **Execution**: Zero-shot direct task completion.
* **Latency**: Fast generation (~2.78 s generation, 7.65 s total).
* **Scope Creep**: Arbitrarily invents unrequested imports (e.g. `HTTPException`, `re`) and injects unrequested business rules (e.g. `len(username) < 3` validation, regex checks).
* **Namespace Drift**: Prone to generating references to undefined schemas and symbols, resulting in unhandled runtime `NameError` exceptions.
* **Verdict**: Produces valid Python syntax, but repeatedly fails formal contract adherence and architectural boundaries.

### CodeSWAT Governed Pipeline
* **Contract Enforcement**: The Planner compiles the task into a rigid, machine-readable JSON specification before execution.
* **Bounded Context**: Code is generated in isolated logical groups (`models`, `dependencies`, `endpoints`), reducing reasoning surface area and eliminating token drift.
* **Deterministic Auditing**: AST structural verification, semantic namespace validation, and contract compliance gates block non-compliant code before assembly.
* **Self-Healing**: Detected errors trigger targeted remediation loops rather than pipeline termination.

| Metric / Dimension | Raw Default 7B | CodeSWAT Pipeline |
|---|---|---|
| Contract Adherence | Unenforced (interprets prompt freely) | Deterministic (JSON specification) |
| Namespace Integrity | Prone to missing symbols (`NameError`) | Statically verified before assembly |
| Architectural Scope | Injects unrequested modules/rules | Strict whitelist of symbols/imports |
| Recovery Mechanism | None (fails at runtime) | Targeted AST/semantic retries |

---

# 2. Baseline — Raw 7B Generation

The first test used the model directly with the task description.

```text
Task
 ↓
7B Model
 ↓
Code
```

The model generated functionally reasonable FastAPI code, but it also started making implementation decisions that were not requested.

Examples included:

```python
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
import uuid
import re
```

and additional validation such as:

```python
if len(user.username) < 3:
    raise HTTPException(...)
```

and:

```python
if not re.match(...):
    raise HTTPException(...)
```

The specification stated what the implementation should contain, but there was no deterministic mechanism forcing the model to stay inside that implementation boundary.

### Baseline Measurement

| Metric | Raw 7B |
|---|---:|
| Model | Qwen2.5.1-Coder-7B-Instruct |
| Quantization | Q6_K_L |
| Generation Mode | Zero-shot |
| Planning | No |
| Contract | No |
| Group Generation | No |
| Generation Time | **2.78 s** |
| Total Execution Window | **7.65 s** |

This established the baseline: **The 7B model can produce working code, but working code is not the same thing as contract-compliant code.**

---

# 3. CodeSWAT Architecture

CodeSWAT introduces an explicit intermediate contract.

```text
Human Task
    ↓
Planner
    ↓
JSON Contract
    ↓
Grouped Generation
    ↓
AST / Architecture Audit
    ↓
Semantic Audit
    ↓
Contract Validation
    ↓
Assembly
    ↓
Full Audit
    ↓
Accepted Code
```

The Planner translates natural language into unambiguous machine-readable constraints: required imports, endpoint routes, schemas, forbidden constructs, dependency providers, and architectural rules. The generation objective shifts from speculative design to explicit implementation under strict contractual constraints.

---

# 4. Group-Based Generation

Code is not generated as one large uncontrolled completion. It is divided into logical groups:

```text
models
dependencies
endpoints
```

Each group receives its own implementation context and is audited before being accepted. Partitioning minimizes the active reasoning window for the 7B parameter footprint, preventing systemic hallucination across multi-layer dependencies.

---

# 5. Semantic Error Recovery Test

During loyalty service generation, the endpoint generator initially produced code referencing `EarnRequest` and `RedeemRequest` without defining those symbols.

The deterministic semantic audit detected the missing symbols (`NameError`), rejected the group, and performed a targeted retry:

```text
Endpoint generation
       ↓
❌ Undefined symbols
       ↓
Semantic Audit
       ↓
Retry
       ↓
Corrected generation
       ↓
✅ Green Light
```

The remediated generation added:

```python
class EarnRequest(BaseModel):
    points: int = Field(..., gt=0)

class RedeemRequest(BaseModel):
    points: int = Field(..., gt=0)
```

The experiment demonstrates that the model is allowed to make mistakes, but the pipeline does not have to accept them.

---

# 6. Behavioural Contract Test

The loyalty service required: `Balance must never drop below zero.`

An unconstrained integer input could allow negative values (`points = -10`). The contract enforced strict Pydantic boundary validation (`points: int = Field(..., gt=0)`) and guarded redemption state updates before modification:

```python
if customer_id not in self.balances or self.balances[customer_id] < points:
    return False

self.balances[customer_id] -= points
return True
```

The verification pipeline must guarantee state and behavioral invariants, not merely Python syntax correctness.

---

# 7. Singleton and State Management

Mutable application state was required to reside inside `class LoyaltyStore:`.

The model generated a singleton implementation using `__new__`:

```python
class LoyaltyStore:
    _instance = None

    def __new__(cls):
        if cls._instance is None:
            cls._instance = super().__new__(cls)
            cls._instance.balances = {}
        return cls._instance
```

Invoking `LoyaltyStore()` does not create a new object when `__new__` returns the existing instance. The validator must enforce contract semantics rather than arbitrary coding style heuristics.

---

# 8. Group Boundary and Namespace Validation

Auditing individual groups in isolation leads to false positives. Downstream groups legitimately rely on upstream symbols (`LoyaltyStore`, `get_loyalty_store`, `EarnRequest`, `RedeemRequest`, `app`).

CodeSWAT resolves this through a continuous symbol manifest tracking accepted symbols, group-owned symbols, and previous group namespaces.

---

# 9. Assembly Validation

Duplicate structural group delimiters emitted during generation:

```text
# === GROUP: dependencies ===
...
# === GROUP: dependencies ===
```

represent an assembly integrity failure. CodeSWAT deterministically requires `START marker == 1` and `END marker == 1` per group, rejecting anomalies prior to final assembly.

---

# 10. Three Levels of Deterministic Validation

* **Level 1 — AST / Syntax**: Valid syntax, imports, function/class definitions, forbidden calls, duplicate definitions, and markers.
* **Level 2 — Semantic Validation**: Undefined symbols, missing dependencies, invalid namespace references, cross-group resolution.
* **Level 3 — Contract Validation**: Request/response schema adherence, forbidden constructs, out-of-scope libraries, and state invariants.

---

# 11. Pipeline Evolution & Benchmark Runs

The comparison between Run A and Run B demonstrates the direct impact of **pipeline policy hardening**. The underlying model remained identical; the control scaffolding was refined.

### Run B: Customer Loyalty Points Service (Early Pipeline Baseline)
* **Status**: Legacy pipeline run before schema policy hardening.
* **Task Profile**: Stateful integer arithmetic with balance invariants (`balance >= 0`).
* **Execution Telemetry**:
  * Planner: 60.73 s
  * models: 4.51 s
  * dependencies: 1.34 s
  * endpoints: 5.21 s
  * Full audit: 0.00 s
* **Policies Enforced**: Singleton state, independent customer balances, Pydantic input limits (`Field(..., gt=0)`).
* **Known Gap (Duplicate Validation)**: Generator reimplemented manual defensive checks (`if points < 0: raise ValueError`) inside the class method despite Pydantic already enforcing it at the gateway.
* **Verdict**: **PASS with Dead Code**. Valid and safe, but highlighted redundant logic not yet pruned by static AST checks.

### Run A: In-Memory User Session Tracker (Hardened Pipeline)
* **Status**: Post-hardening run with explicit Pydantic subclass policies and strict namespace ordering.
* **Task Profile**: Explicit Pydantic subclass requirement, singleton state.
* **Execution Telemetry**:
  * Planner: 17.93 s
  * models: 6.69 s
  * dependencies: 1.94 s
  * endpoints: 3.74 s
  * Full audit: 0.00 s (Total execution: ~30 s)
* **Policies Enforced**: AST rejection of raw `BaseModel`, singleton registry via `__new__`, full FastAPI `Depends()` injection.
* **Verdict**: **PASS (0 retries)**. Convergence time dropped from ~60s down to ~30s with zero generation retries required.

---

# 12. Addendum — Pipeline-Level Bugs Found During Contract Hardening

Contract hardening revealed three specific pipeline-level bugs distinct from ordinary syntax errors:

1. **Group Ordering Inversion**: A dependency provider's target group could be scheduled after the endpoint group that required it, triggering downstream resolution failures during sequential assembly.
2. **Group Rule-Count Overflow**: Injecting multiple architectural guardrails into a single module pushed rule counts beyond threshold limits, causing planner validation failures instead of balanced partitioning.
3. **Redundant Validation Branching**: The code generator duplicated network-layer validation inside business logic methods. This dead code highlights the necessity for cross-layer AST semantic reasoning between Pydantic boundary definitions and internal function blocks.

All three findings point to a single conclusion: **the contract layer itself requires the same deterministic rigor as the code it constrains.** CodeSWAT's long-term reliability depends equally on Planner-side schema validation and Generator-side AST auditing.

---

# 13. Final Principle

> **The goal is not to trust the model more.**
> **The goal is to give the system more control over the model.**

The model generates. **CodeSWAT decides what is allowed to pass.**

---

### Artifacts & Telemetry References

The raw pipeline execution traces referenced in this validation report are preserved as immutable benchmarks:

### Artifacts & Telemetry References

The raw pipeline execution traces referenced in this validation report are preserved as immutable benchmarks:

* **[Run A Telemetry Trace](Logs/Logs_1.md)**: In-Memory User Session Tracker trace (~30s, 0 retries).
* **[Run B Telemetry Trace](Logs/Logs_2.md)**: Customer Loyalty Points Service trace and duplicate validation analysis (~60s).
