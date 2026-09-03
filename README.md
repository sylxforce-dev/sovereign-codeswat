# Sovereign CodeSWAT

CodeSWAT is a deterministic validation and control pipeline for local LLM code generation.

It turns a natural-language task into an explicit contract, generates code in controlled groups, and validates the result before assembly.

```text
Task → Planner → Contract → Generation → Audit → Repair → Accepted Code
```

---

## Technical Validation Report

For the full architectural breakdown, 7B baseline comparisons, policy hardening details, and runtime telemetry, see the primary report:

👉 **[Read the Full Validation Report (CodeSWAT.md)](./CodeSWAT.md)**

---

### Pipeline Summary

* **Raw 7B vs. CodeSWAT**: Eliminates arbitrary scope creep, unrequested imports, and runtime `NameError` drift.
* **Three-Tier Deterministic Gates**: Structural AST audits, semantic namespace tracking, and contract-level invariant verification.
* **Pipeline Hardening**: Policy improvements reduced convergence time from ~60s down to ~30s with **0 retries**.
