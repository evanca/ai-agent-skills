# AI Agent Skills for Claude Code, Codex, Cursor, Antigravity, and Other AI Coding Agents

**Agent skills your coding agent loads by itself: design agent loops that stop when they should, test an app AI's scope boundaries, and find out why a Claude prompt cache keeps missing.**

A skill is a folder with a `SKILL.md` file. Your agent reads the description, decides a task matches, and pulls in the guidance. You don't paste anything into a rules file or remember to `@`-mention a doc.

These skills aren't tied to a language or framework. For Flutter and Dart, see [flutter-ai-rules](https://github.com/evanca/flutter-ai-rules).

## ⚡ Quick start

```sh
npx skills add evanca/ai-agent-skills
```

The [Skills CLI](https://www.npmjs.com/package/skills) discovers the packages under [`skills/`](./skills) and installs them for supported agents. It needs **Node 22.20 or newer**. Every other install route below works on any Node version, or none at all.

Browse before installing, or take just one:

```sh
npx skills add evanca/ai-agent-skills --list
npx skills add evanca/ai-agent-skills --skill loop-designer
```

### Install as a plugin

This repo is also a plugin marketplace, so agents that support plugins can install the whole set and keep it updated in place. The manifests live at `.claude-plugin/`, `.codex-plugin/`, `.cursor-plugin/`, `.agents/plugins/`, and `plugin.json`, all pointing at the same [`skills/`](./skills) directory.

**Claude Code** — [plugin marketplaces](https://code.claude.com/docs/en/plugin-marketplaces):

```sh
/plugin marketplace add evanca/ai-agent-skills
/plugin install ai-agent-skills@ai-agent-skills
```

Skills then load as `ai-agent-skills:loop-designer`, `ai-agent-skills:cache-diagnostics`, and `ai-agent-skills:ai-scope-qa`.

**Codex** — [plugins](https://developers.openai.com/codex/plugins/build):

```sh
codex plugin marketplace add evanca/ai-agent-skills
codex plugin install ai-agent-skills@ai-agent-skills
```

**Cursor** — [plugins](https://cursor.com/docs/plugins): in Cursor, run `/add-plugin`, or add the repo as a team marketplace under **Dashboard → Settings → Plugins → Import from Repo**:

```
https://github.com/evanca/ai-agent-skills
```

**Antigravity** — [plugins](https://antigravity.google/docs/plugins):

```sh
git clone --depth 1 https://github.com/evanca/ai-agent-skills.git
agy plugin install ./ai-agent-skills
```

**Windsurf / Devin** has no plugin manifest, but Cascade already scans `.agents/skills/` and `~/.agents/skills/`, so copying a skill folder there is the install path.

Prefer to do it by hand? Copy or symlink any skill folder into your agent's skills directory: `.claude/skills/`, `.cursor/skills/`, `.codex/skills/`, `.agents/skills/` (Antigravity, Codex, Windsurf), `.windsurf/skills/`, or the global equivalent such as `~/.claude/skills/`.

## 🧠 What's in the box

| Skill | Loads when you're… |
|---|---|
| [`loop-designer`](./skills/loop-designer) | Automating a recurring task, building a retry-until-green loop, or debugging a loop that stalls or burns tokens |
| [`cache-diagnostics`](./skills/cache-diagnostics) | Finding out why a Claude prompt cache keeps missing or a turn costs more than expected |
| [`ai-scope-qa`](./skills/ai-scope-qa) | Testing an app AI's scope boundaries, false refusals, topic drift, or prompt injection |

### loop-designer

Designs an agent loop before it picks a tool. A seven-row sheet covers the trigger, the work inside one iteration, verification, stop conditions (success, caps, stagnation, escalation), budget, concurrency and state, and observability. The filled sheet then binds to a runtime:

- **Claude Code** — `/loop`, `/goal`, and `/schedule` cloud routines
- **Claude Agent SDK** — `max_turns`, `max_budget_usd`, hooks, compaction, session resume
- **OpenAI Codex CLI** — headless review → repair → validate pipelines with JSON-schema output per phase
- **OpenClaw** — serialized sessions, lanes and queues, layered timeouts, liveness diagnostics

It also debugs loops that already misbehave: vague exit criteria, an interval faster than the watched thing changes, unhandled stagnation, or an evaluator that is also the doer.

### cache-diagnostics

Prompt caching only works when a request starts byte-for-byte identical to a recent one. Reorder a tool, interpolate a timestamp into a system prompt, or edit an earlier message, and `usage.cache_read_input_tokens` drops to zero with nothing saying why.

The skill uses Anthropic's `cache-diagnosis-2026-04-07` beta, which compares two consecutive requests and reports where they first diverged. It covers the six `cache_miss_reason` types and the fix for each, three response states that look like findings and aren't, and the limits that make a result untrustworthy.

It bundles [`diagnose_cache.py`](./skills/cache-diagnostics/scripts/diagnose_cache.py), a probe that sends the same prefix twice with your own system prompt and tools:

```sh
python3 skills/cache-diagnostics/scripts/diagnose_cache.py --system-file prompts/system.txt --tools-file tools.json
```

Install `anthropic` and set credentials first: `ANTHROPIC_API_KEY`, or an `ant auth login` profile. Cache diagnostics runs on the Claude API only, not on Claude Platform on AWS, Amazon Bedrock, Google Cloud, or Microsoft Foundry.

### ai-scope-qa

Builds contextual conversation tests for an app's AI: legitimate tasks, ambiguous requests, ordinary off-topic input, and deliberate prompt injection. It checks false refusals alongside missed refusals and separates response wording from effects on state, access, quota, and stored data.

Includes a [prompt checklist](./skills/ai-scope-qa/references/prompt-checklist.md), [case design and evaluation guidance](./skills/ai-scope-qa/references/case-design.md), and [primary references](./skills/ai-scope-qa/references/sources.md). Expectations come from the app's own policy and supported actions.

## 📚 Sources and credits

Each skill condenses the documentation of the tools it covers. The vendors own that behavior, and their documentation is the authority: where a page and a skill disagree, trust the page. No vendor has reviewed or endorsed this repository.

**loop-designer** — [Getting started with loops](https://claude.com/blog/getting-started-with-loops) · [Agent SDK agent loop](https://code.claude.com/docs/en/agent-sdk/agent-loop) · [Build iterative repair loops with Codex](https://developers.openai.com/cookbook/examples/codex/build_iterative_repair_loops_with_codex) · [OpenClaw agent loop](https://docs.openclaw.ai/concepts/agent-loop)

**cache-diagnostics** — [Cache diagnostics](https://platform.claude.com/docs/en/build-with-claude/cache-diagnostics) · [Prompt caching](https://platform.claude.com/docs/en/build-with-claude/prompt-caching)

**ai-scope-qa** — [Out-of-scope prediction (Larson et al., 2019)](https://aclanthology.org/D19-1131/) · [CheckList behavioral testing (Ribeiro et al., 2020)](https://aclanthology.org/2020.acl-main.442/) · [Dialogflow CX test cases](https://docs.cloud.google.com/dialogflow/cx/docs/concept/test-case) · [OWASP LLM Prompt Injection Prevention](https://cheatsheetseries.owasp.org/cheatsheets/LLM_Prompt_Injection_Prevention_Cheat_Sheet.html). See the skill's [source and license record](./skills/ai-scope-qa/references/sources.md).

## 🛠️ Contributing

1. Fork this repository.
2. Add or change a skill in [`skills/`](./skills). Write its `description` as one line that starts with `Use when` and lists only trigger conditions.
3. Link the documentation the skill draws on, and open a pull request explaining the change.

## 📄 License

[MIT](./LICENSE). The license covers this repository's own text and scripts. The documentation the skills condense stays its owners'.
