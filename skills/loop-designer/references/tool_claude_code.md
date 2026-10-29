# Loop types (from claude.com/blog/getting-started-with-loops)

Loops = "agents repeating cycles of work until a stop condition is met." Categorized by trigger, stop criteria, primitives, and task fit.

## Turn-based loops
- **Trigger:** user prompt. **Stop:** Claude determines completion or needs context.
- **Best for:** shorter tasks outside regular processes.
- Encode verification as skills: e.g. "Verify any UI change end-to-end before declaring it done" — start dev servers, interact with changes, check console errors, run performance audits.

## Goal-based loops (`/goal`)
- **Trigger:** manual real-time prompt. **Stop:** goal achieved or max turns reached.
- **Best for:** tasks with verifiable exit criteria.
- Example: `/goal get the homepage Lighthouse score to 90 or above, stop after 5 tries.`
- An evaluator model checks whether success criteria are met before allowing the agent to stop.

## Time-based loops (`/loop` and `/schedule`)
- **Trigger:** specified time intervals. **Stop:** user cancellation or work completion.
- **Best for:** recurring work or monitoring external systems.
- `/loop` runs locally on your machine; `/schedule` runs in the cloud for autonomous operation.
- Example: `/loop 5m check my PR, address review comments, and fix failing CI`

## Proactive loops
- **Trigger:** events or schedules without real-time human involvement. **Stop:** individual tasks exit when goals met; the routine runs until disabled.
- **Best for:** recurring, well-defined work streams.
- Compose primitives: `/schedule every hour: check #project-feedback for bug reports. /goal: don't stop until every report found this run is triaged, actioned, and responded to.`

## Managing code quality in loops
- Keep the codebase clean enough for Claude to follow.
- Encode verification as skills using quantitative checks.
- Make documentation easily accessible.
- Use a second agent for code review to reduce self-review bias.
- When issues arise, encode the fix systematically (skill/CLAUDE.md) so all future iterations improve.

## Token usage management
- Match primitives and models to task complexity.
- Define specific success and stop criteria — vague goals burn tokens.
- Pilot before large-scale runs.
- Use scripts for deterministic work instead of reasoning through steps.
- Match routine intervals to actual change frequency of the thing being watched.
- Review usage via `/usage`.

## Getting started heuristic
Identify bottleneck tasks and check: (1) can you write verification checks? (2) can you define completion clearly? (3) is there a scheduled work pattern? Then run the loop, observe where it stalls or over-reaches, and iterate.
