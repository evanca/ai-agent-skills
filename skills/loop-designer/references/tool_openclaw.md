# Tool reference: OpenClaw — agent loop runtime

Source: docs.openclaw.ai/concepts/agent-loop (fetched 2026-07-02)

OpenClaw's loop: "a single, serialized run per session that emits lifecycle and stream events as the model thinks, calls tools, and streams output." Valuable here mostly as the reference design for the *runtime engineering* of loops: serialization, timeouts, liveness, hooks.

## Flow
1. `agent` RPC validates params, resolves session, persists metadata, returns `runId` immediately
2. `agentCommand` resolves model defaults, loads skill snapshots, invokes embedded runtime
3. `runEmbeddedAgent` serializes via per-session queues, resolves model/auth, streams events, enforces timeouts
4. Event bridging routes runtime events to tool / assistant / lifecycle streams

## Concurrency model (the part worth stealing)
- Runs serialize through **per-session lanes** + optional global queue → prevents tool/session races, keeps history consistent
- Queue modes for incoming messages: steer / followup / collect / interrupt
- Transcript writes behind a **file-based process-aware write lock** (default acquire timeout 60s; non-reentrant unless `allowReentrant: true`)
- Compaction/rewrites must take the same lock

## Timeout layering (set all three)
| Layer | Default | Note |
|---|---|---|
| `agent.wait` RPC | 30s | wait-only; doesn't stop the agent |
| Agent runtime | 172800s (48h) | whole-run ceiling |
| Model idle watchdog | provider-configurable | aborts when no chunks arrive; extend for slow local providers (Ollama) BEFORE raising whole-agent timeout |

Cron runs get isolated per-turn timeouts so stale child sessions can't block the lane.

## Liveness diagnostics (stalled-loop detection)
- `session.long_running` — active work reporting progress
- `session.stalled` — active work, no recent progress (abort threshold flips runs here)
- `session.stuck` — recoverable stale bookkeeping; releases the lane on recovery
- `stuckSessionWarnMs` warns; `stuckSessionAbortMs` (≥5 min and 3× warn) drains aborted runs so queued work resumes

## Hook interception points
`agent:bootstrap` (modify bootstrap files pre-prompt), `before_model_resolve`, `before_prompt_build`, `before_agent_reply` (claim/suppress turn), `before_tool_call` / `after_tool_call` (`{block: true}` is terminal), `tool_result_persist` (transform before transcript write), message lifecycle, `session_start`/`session_end`.

## Termination
Agent timeout (abort), AbortSignal (cancel), gateway disconnect / RPC timeout. Auto-compaction emits events and can trigger retry — buffers reset on retry to avoid duplicate output.

## Warnings
- Write locks are blocking: slow filesystems or concurrent writers delay everything
- Timeout stacking: idle watchdog capped by run timeout and provider timeout — tune all consistently
- Stalled-recovery backs off on repeats; persistently stuck sessions may need manual intervention
