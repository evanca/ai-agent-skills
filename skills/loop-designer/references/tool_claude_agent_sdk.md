# The SDK agent loop (from code.claude.com/docs/en/agent-sdk/agent-loop)

The Agent SDK embeds Claude Code's autonomous loop in your own apps: Claude evaluates the prompt, calls tools, receives results, repeats until done. Standalone package — no CLI install needed.

## The cycle
1. **Receive prompt** — prompt + system prompt + tool definitions + history. SDK yields `SystemMessage` subtype `"init"` with session metadata.
2. **Evaluate and respond** — text, tool calls, or both → `AssistantMessage`.
3. **Execute tools** — SDK runs each requested tool; results feed back. Hooks can intercept/modify/block.
4. **Repeat** — each full cycle is one turn; continues until a response with no tool calls.
5. **Return result** — final `AssistantMessage`, then `ResultMessage` (final text, token usage, cost, session ID).

A **turn** = one round trip (Claude output with tool calls → tools execute → results feed back), without yielding control to your code.

## Message types
- **`SystemMessage`**: `"init"`, `"compact_boundary"` (after compaction), `"informational"`, `"worker_shutting_down"`
- **`AssistantMessage`**: after each Claude response (text + tool call blocks). TS wraps raw API message: content at `message.message.content`
- **`UserMessage`**: after each tool execution (tool results), and streamed mid-loop user inputs
- **`StreamEvent`**: only with partial messages enabled (`include_partial_messages` / `includePartialMessages`)
- **`ResultMessage`**: end of loop — final text, usage, cost, session ID. Iterate the stream to completion (trailing events like `prompt_suggestion` can arrive after it)

Python: check with `isinstance(message, ResultMessage)`. TypeScript: check `message.type === "result"`.

## Tools
Built-ins: Read/Edit/Write, Glob/Grep, Bash, WebSearch/WebFetch, ToolSearch, Agent/Skill/AskUserQuestion/TaskCreate/TaskUpdate. Plus MCP servers, custom tool handlers, project skills via setting sources.

**Permissions:** `allowed_tools` auto-approves listed tools; `disallowed_tools` blocks regardless; `permission_mode` covers the rest. Scope with rules like `"Bash(npm *)"`. Denied tool → Claude gets a rejection message and adapts.

**Parallel execution:** read-only tools run concurrently; state-mutating tools (Edit/Write/Bash) run sequentially. Custom tools default sequential — set `readOnlyHint` in annotations to allow parallel.

## Controlling the loop

| Option | Controls | Default |
|---|---|---|
| `max_turns` / `maxTurns` | max tool-use round trips | no limit |
| `max_budget_usd` / `maxBudgetUsd` | max spend before stopping | no limit |

Hitting a limit → `ResultMessage` subtype `error_max_turns` / `error_max_budget_usd`. **Setting a budget is a good default for production agents.**

**Effort** (`"low"`→`"max"`): reasoning depth per response; `"xhigh"` recommended for coding/agentic tasks on Fable 5, Opus 4.7+, Sonnet 5. Independent from extended thinking. Set per session or per subagent (`AgentDefinition.effort`).

**Permission modes:** `"default"` (approval callback; no callback = deny), `"acceptEdits"` (auto-approve edits + common fs commands), `"plan"` (explore without editing), `"dontAsk"` (pre-approved runs, rest denied), `"auto"` (TS only, model classifier), `"bypassPermissions"` (isolated environments only; not as root).

**Model:** unset → Claude Code default; pin explicitly (e.g. `claude-sonnet-5`) for cheaper/faster agents.

## Context window
Everything accumulates across turns: system prompt, tool definitions, history, tool outputs. Stable prefixes (system prompt, tool defs, CLAUDE.md) are prompt-cached. Large tool outputs are the big consumer.

**Automatic compaction** summarizes older history near the limit (emits `compact_boundary`). Early-conversation instructions may not survive — persistent rules belong in CLAUDE.md (re-injected every request). Customize: summarization-instructions section in CLAUDE.md (free-form header, matched on intent), `PreCompact` hook (e.g. archive transcript), or send `/compact` as a prompt.

**Efficiency:** subagents for subtasks (fresh context, only final response returns to parent); scope subagent `tools` to minimum; watch MCP schema costs (tool search defers them by default, but not on Vertex/non-first-party base URL); lower effort for routine agents.

## Sessions
Capture `ResultMessage.session_id` to resume (full context restored) or fork. Python `ClaudeSDKClient` handles session IDs automatically across calls.

## Result handling

| subtype | meaning | `result` field? |
|---|---|---|
| `success` | finished normally | yes |
| `error_max_turns` | hit maxTurns | no |
| `error_max_budget_usd` | hit budget | no |
| `error_during_execution` | API failure / cancel | no |
| `error_max_structured_output_retries` | structured output never validated | no |

All subtypes carry `total_cost_usd`, `usage`, `num_turns`, `session_id` (Python: cost/usage may be `None` — guard). `stop_reason`: `end_turn`, `max_tokens`, `refusal` — check for refusals explicitly.

## Hooks
Run in your process (no context cost); can short-circuit the loop.

| Hook | Fires | Uses |
|---|---|---|
| `PreToolUse` | before tool executes | validate inputs, block dangerous commands |
| `PostToolUse` | after tool returns | audit, side effects |
| `UserPromptSubmit` | prompt sent | inject context |
| `Stop` | agent finishes | validate result, save state |
| `SubagentStart`/`SubagentStop` | subagent lifecycle | track parallel results |
| `PreCompact` | before compaction | archive transcript |

## Canonical production agent (Python)
```python
import asyncio
from claude_agent_sdk import query, ClaudeAgentOptions, ResultMessage

async def run_agent():
    async for message in query(
        prompt="Find and fix the bug causing test failures in the auth module",
        options=ClaudeAgentOptions(
            allowed_tools=["Read", "Edit", "Bash", "Glob", "Grep"],  # auto-approved
            setting_sources=["project"],  # CLAUDE.md, skills, hooks from cwd
            max_turns=30,                 # prevent runaway sessions
            effort="high",
        ),
    ):
        if isinstance(message, ResultMessage):
            if message.subtype == "success":
                print(f"Done: {message.result}")
            elif message.subtype == "error_max_turns":
                print(f"Hit turn limit. Resume session {message.session_id}.")
            else:
                print(f"Stopped: {message.subtype}")
            if message.total_cost_usd is not None:
                print(f"Cost: ${message.total_cost_usd:.4f}")

asyncio.run(run_agent())
```
TypeScript mirrors this: `query({ prompt, options: { allowedTools, settingSources: ["project"], maxTurns, effort } })`, check `message.type === "result"`, session id from the init system message.
