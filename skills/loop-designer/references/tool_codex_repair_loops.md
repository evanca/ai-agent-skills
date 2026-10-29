# Tool reference: OpenAI Codex — iterative repair loops

Source: developers.openai.com/cookbook/examples/codex/build_iterative_repair_loops_with_codex (fetched 2026-07-02)

Closed-loop pattern: produce → validate → feed structured findings back → improve. Codex runs headless via CLI with JSON-schema-constrained output per phase. The canonical fixture repairs Jupyter notebooks with stale API references.

## The three-phase design

| Phase | Does | Does NOT | Output |
|---|---|---|---|
| **Review** | Inspect artifacts, return structured findings with severity | Execute or edit anything | Categorized findings |
| **Repair** | Apply the *smallest useful edits* addressing findings + validation delta, on artifact copies | Wholesale rewrites | Change summary + artifact paths |
| **Validate** | Execute the artifact, evaluate against criteria | — | Pass/fail verdict + remaining delta |

Validation failure is "structured feedback instead of a dead end" — the delta becomes the next repair's instructions.

## Structured handoffs
Machine-readable outputs between phases (JSON schemas per phase). Enables debugging and adaptation without reconstructing runs from diffs/logs.

## Termination conditions
- Validation passes completely
- Max iteration attempts reached
- **Stagnation detection: remaining delta stops changing between iterations**
- Human review required (ambiguous validation signals)

Intentional staging is legitimate: "continue to iteration N" delta = planned progression, not failure.

## Verification evidence types
- **Execution testing** — run the artifact; runtime failures, missing imports, stale paths
- **Structural validation** — artifact integrity (e.g. cell count preservation)
- **Pattern matching** — deprecated constructs, modernized API usage
- **Rubric scoring** — independent evaluation per discrete criterion

## Best practices
- Explicit business rules + issue taxonomy upfront (define "good" before looping)
- Full audit trail per iteration: findings, changes, validation results, remaining delta
- Evidence-driven repairs: ground in observed failures, not anticipated problems
- Staged complexity: multiple passes beat demanding full resolution in one attempt
- Independent cases run in parallel with shared validation logic

## Warnings
- Validation quality bounds loop quality — poor criteria perpetuate errors across iterations
- Iteration caps prevent infinite loops but can leave artifacts mid-state
- Unfocused repair prompts risk perpetual incompleteness
- Human review stays necessary for ambiguous signals

## Applicable beyond code
Docs maintenance, regulatory content (required language/citations intact), support knowledge vs current product behavior, code modernization against test suites.
