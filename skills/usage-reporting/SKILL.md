---
name: usage-reporting
description: One cycle step of a user's side hustle loop. It reports the usage of this cycle to Asana per person, model, task and phase, and the user's pledge for the week, so the dashboard can read it. The start skill calls it at the end of each cycle; it does not loop by itself.
---

# Usage reporting skill (side hustle)

This skill makes every agent report its usage (SPEC.md §11, AC47, AC48), so that the dashboard can attribute it. B is the user whose session runs it. It runs at the end of each cycle (start skill step 6), in one pass, and never waits on anyone.

**Asana is the store.** Every report is an Asana comment, the same as every other side hustle record. There is no new infrastructure, and every user and the dashboard can read it.

## During the cycle: the segment log

The start skill keeps a short log in the session while the cycle runs. Each time the cycle starts working on a task, it notes:
- the time;
- the task gid;
- the phase:
  - `implementation`: pick-up implementing or delivering;
  - `review`: the reviewer;
  - `communication`: the Slack skill, fill-in, or pick-up notices that do not implement.

The segment ends when the next one starts, or when the step ends. Work that touches no task, such as polling or reading config, is not attributed to any task and is not reported.

## 1. Measure each segment

**Tokens (measured).**
- **Sources:** the current session's Claude Code transcript (`~/.claude/projects/<project>/<session id>.jsonl`), plus the transcripts of any **subagents** the cycle started, which Claude Code stores separately for that session. That includes the reviewer's independent context. Each assistant record carries its `model`, its token `usage`, and a response id (`message.id`).
- **Count each response once:** one response can appear as several records, for example one per content or tool block. Group the records by `message.id` and take that response's usage once, from its last (completed) record.
- **Attribute it:**
  - a main-session response belongs to the segment its timestamp falls in;
  - a subagent's responses all belong to the segment that started that subagent, for example the reviewer's `review` segment.

  Only numbers are read, never the subagent's content, so no implementer reasoning reaches the reviewer and no review content leaves it.
- **Count per segment:** for each `(task, phase, model)`, sum `input_tokens + output_tokens + cache_creation_input_tokens + cache_read_input_tokens` over those responses.
- **If a transcript cannot be read**, the session's or a subagent's: estimate that part as its characters ÷ 4 and mark the line `tokens: estimate`. If not even that is possible, mark it `tokens: unavailable`. Never report missing usage as a measured zero.

**Cost (estimate).** `cost_usd = tokens / 1,000,000 × price_usd_per_mtok[model]`, with the rate from `${CLAUDE_PLUGIN_ROOT}/config/global.yaml` `usage.price_usd_per_mtok`. Models without a rate report `cost: n/a`. A subscription is not billed per token, so the cost is an API-equivalent figure for comparison only.

**% of the weekly limit (estimate).** Claude Code does not expose the weekly limit as a number, so this is `100 × tokens / usage.weekly_limit_tokens_estimate`.

## 2. Report per task (AC47)

On each task worked on in this cycle, add **one** comment, with one line per `(phase, model)`:

```
Side hustle usage: <B> · <ISO week> · <cycle start time>
<phase> · <model> · <tokens> tokens (<measured|estimate|unavailable>) · ~$<cost> · ~<pct>% of weekly limit (estimate)
```

Before posting, read the task's comments. Never post twice for the same cycle start time, so the report is idempotent.

## 3. Report the pledge (AC48)

Keep B's pledge on the ledger task, `usage.ledger_task` in `${CLAUDE_PLUGIN_ROOT}/config/global.yaml`. It is a completed system task in Done that every other skill ignores.
- **When:** at the first cycle of a week, and whenever `pledge_percent` changes.
- **What:** add `Side hustle pledge: <B> · <pledge_percent>% · <ISO week>`.

If `ledger_task` is empty, skip this step and say so once in the session.

## Task events

The dashboard derives task events from what Asana already holds, so this skill reports nothing extra for them:
- **Published:** task creation.
- **Picked up:** the `Side hustle claim:` record and the assignee change.
- **Section changes:** the section history.
- **Completed:** moved to Done.
- **Released, stalled, review results:** their markers in `${CLAUDE_PLUGIN_ROOT}/templates/task-description.md`.

## Pledge used stays on reserved budgets

These reports measure tokens, but their % of the weekly limit is an **estimate**, because the real limit is not known. Pledge used therefore stays the sum of reserved budgets (claim records), as in pick-up. The switch to actual consumption happens only once usage is reported as a measured % of the weekly limit.
