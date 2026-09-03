# Sovereign CodeSWAT

CodeSWAT is a deterministic validation and control pipeline for local LLM code generation.

It turns a natural-language task into an explicit contract, generates code in controlled groups, and validates the result before assembly.

```text
Task → Planner → Contract → Generation → Audit → Repair → Accepted Code
