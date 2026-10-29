---
name: loop-designer
description: Use when the user wants to automate a recurring task, monitor something continuously ("check my PR every 5 minutes", "watch CI", "triage feedback hourly"), build a retry-until-green repair loop, schedule autonomous agent work, design a produce-validate-fix pipeline, or debug a loop that stalls, over-reaches, or burns tokens; trigger on /goal, /loop, /schedule, cron-like agent automation, "keep doing X until Y", iterative repair, self-healing pipelines, or agent-loop questions for any runtime including Claude Code, Claude Agent SDK, OpenAI Codex, and OpenClaw.
---

# Loop Designer

Design agent loops tool-agnostically, then bind them to a concrete runtime. A loop is an agent repeating cycles of work until a stop condition is met — the design questions are identical whether it runs as a Claude Code slash command, an SDK program, a Codex CLI pipeline, or an OpenClaw session. Design first, pick the tool last.

## The design sheet (fill these seven, in order)

Walk the user through these; a loop with a blank in any row is not ready to run.

**1. Trigger** — what starts an iteration?
- Manual prompt (one-shot with retries) · Goal (run until verified) · Clock (interval/cron) · Event (webhook, message, file change)
- For clock triggers: match the interval to the *actual change frequency* of the watched thing. Polling faster than reality changes burns budget for nothing.

**2. Work shape** — what happens inside one iteration?
- The strongest pattern for artifact-improving loops is **three phases with structured handoffs** (from the Codex cookbook): **Review** (find issues, touch nothing) → **Repair** (smallest useful edit addressing findings + last validation delta) → **Validate** (execute and score, emit pass/fail + remaining delta). Each phase emits machine-readable output (JSON) so the next phase — and the audit trail — never depends on parsing prose.
- Use scripts for deterministic steps instead of model reasoning; save the model for judgment.

**3. Verification** — how does the loop know work is good?
This is the whole game: **validation quality bounds loop quality**; weak criteria perpetuate errors across iterations. Prefer evidence in this order: execution (run it), structural checks (integrity), pattern checks (linting/greps), rubric scoring (independent evaluator). Separate the evaluator from the doer — a second agent or an evaluator model checking exit criteria reduces self-review bias. If you can't write the check, sharpen the goal before looping.

**4. Stop conditions** — every loop needs at least three:
- **Success**: verification passes
- **Cap**: max iterations AND max budget/spend — caps prevent infinite loops but can leave artifacts mid-state, so define what "mid-state" means for cleanup
- **Stagnation**: the remaining delta stops changing between iterations → stop and escalate, don't grind
- **Escalation**: ambiguous validation signals go to a human, not another iteration

**5. Budget & efficiency**
- Set spend/turn caps as a default, not an exception. Match model/effort to task complexity per phase (cheap for mechanical steps, strong for judgment). Pilot on one case before fanning out. Independent cases can run in parallel with shared validation logic.

**6. Concurrency & state** (matters as soon as anything persists or two runs can overlap)
- Serialize runs per session/resource (lanes/queues) to prevent races; protect shared transcripts/state with locks — and know your lock timeout behavior.
- Layer timeouts: per-call idle watchdog < per-iteration timeout < whole-run ceiling; tune all consistently or the tightest one silently dominates.
- Long-lived loops need context discipline: fresh contexts (subagents) per subtask, persistent rules stored outside the transcript (they must survive compaction/summarization), archive full transcripts before compaction if auditability matters.

**7. Observability**
- Keep an audit trail per iteration: findings, changes, validation result, remaining delta — reviewable without re-reading artifacts.
- Detect liveness, not just completion: distinguish *long-running* (progressing) from *stalled* (active, no progress) from *stuck* (needs recovery); warn early, abort late.
- When a loop misbehaves, encode the fix into the loop's rules/skills so every future iteration inherits it — don't just nudge the current run.

## Binding to a tool

Once the sheet is filled, open the matching reference and translate. **The references/ directory is extensible — when the user adopts a new runtime, research it and add a `tool_<name>.md` so this skill stays current.**

| Runtime | Reference | Use when |
|---|---|---|
| Claude Code (interactive/CLI) | `references/tool_claude_code.md` | Turn-based prompts, `/goal`, `/loop <interval>`, `/schedule` cloud routines, composed proactive loops |
| Claude Agent SDK (Python/TS) | `references/tool_claude_agent_sdk.md` | Embedding the loop in your own app: query options, max_turns/max_budget_usd, permission modes, hooks, compaction, session resume |
| OpenAI Codex CLI | `references/tool_codex_repair_loops.md` | Headless review→repair→validate pipelines with JSON-schema outputs per phase |
| OpenClaw | `references/tool_openclaw.md` | Session-serialized runtime: lanes/queues, write locks, layered timeouts, liveness diagnostics, plugin hooks |

Cross-tool cheat map for the sheet's rows: **caps** = Claude Code `/goal ... stop after N tries` · SDK `max_turns`/`max_budget_usd` · Codex iteration caps · OpenClaw timeouts. **Verification** = Claude verification skills / evaluator model · Codex validate phase schema · your own test suite everywhere. **Stagnation** = compare the validation delta across iterations (implement yourself on every runtime except where noted). **Liveness** = OpenClaw diagnostics natively; elsewhere, emit per-iteration heartbeats and alert on silence.

## Deliverables

- **Loop design doc** — the filled seven-row sheet with the verification checks written out
- **Runnable binding** — the exact command (`/loop 5m ...`, `/goal ... stop after 5 tries`), SDK code with limits and result handling, or phase prompts + JSON schemas for a repair pipeline
- **Loop debugging** — diagnose against the sheet: vague exit criteria? interval mismatched to change frequency? stagnation unhandled? context rot from compaction? evaluator = doer? missing caps?
- **Runtime comparison** — when the user hasn't picked a tool, recommend one based on where the loop must live (their terminal vs cloud vs their own app vs CI)
