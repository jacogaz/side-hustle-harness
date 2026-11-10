---
name: side-hustle-dashboard
description: Show the Side Hustle dashboard. It reads the Side Hustle Asana project and renders the usage, flow, quality, people and optimization blocks with names, plus informational insights. Read-only. Use when anyone on the team asks for the side hustle dashboard or metrics.
---

# Side Hustle dashboard (`/side-hustle-dashboard`)

Any team member runs this skill to see the dashboard (SPEC.md §11, AC46–AC51). It reads Asana and renders the result in the Claude session, with names. There is no server and no database.

**It is read-only.** It never writes to Asana, Slack or the config, and never changes a task, setting, budget or pledge (AC49).

## Inputs

- **`config/global.yaml`:** `asana_project` (project, sections and custom fields), `usage.ledger_task` and `heartbeat_timeout_hours`.
- **Every task in the project except the ledger task:** read its name, creator, assignee, section history, custom fields (Budget, Estimate, Last heartbeat) and comments. The markers are defined in `templates/task-description.md`.
- **The ledger task:** read only its `Side hustle pledge:` comments.
- **Week:** the current ISO week by default. The user may name another week.
- **People:** shown by their Asana names.

## Blocks

### Usage (week)

Sources are the `Side hustle usage:` comments of the week, and the ledger's `Side hustle pledge:` records for the week.
- **Tokens and cost**, one table each for: per person, per model, per task, and per phase (implementation, review, communication). Mark cost as an estimate, and show `n/a` costs as such.
- **Pledge per user**, one row each, in % of the weekly limit:
  - pledge: the user's latest pledge record of the week;
  - pledge used (reserved): the sum of the user's `Side hustle claim:` records of the week;
  - actual (estimate): the sum of the `~<pct>%` in the user's usage lines of the week, without `review` lines, because reviews belong to the task's budget, not the reviewer's pledge.

### Flow

- **Tasks per section:** the current counts.
- **Average time in each section:** from the section history, using finished stints only.
- **Total Open → Done time:** for each Done task, from creation to its last move to Done. Show the average and the per-task value.
- **Flags (AC50):**
  - **stalled:** the latest stalled marker is `Side hustle stalled`;
  - **orphaned:** in Doing or To Review with no heartbeat for `heartbeat_timeout_hours` (the same rule as pick-up step 1), or released this week with `Side hustle released: orphaned`.

### Quality

- **First-pass approval:** the % of Done tasks that never went QA Review → Next.
- **Rejections:** the number of QA Review → Next moves, overall and per task.
- **Reviewer failures:** the authoritative `Side hustle review: FAIL` results, by outcome (retry, attempts exhausted, destructive) and by failed check (compliance, scope, non-destructive, harness), with each one's one-line reason.

### People (AC51)

A leaderboard, all time and this week:
- **publishes:** tasks created;
- **picks up:** `Side hustle claim:` records;
- **completes:** tasks that reached Done, counted for the user assigned when the task reached Done, who is the picker who delivered it.

### Optimization (AC48)

- **Completed tasks:** for each Done task, actual consumption against Budget and Estimate. Actual is the sum of the `~<pct>%` in all its usage lines, all users and phases. Also show the difference, actual minus Budget.
- **Per model:** the cost, and the success rate. The success rate is the % of tasks whose implementation used that model and were approved by the owner on the first pass.
- **Bottlenecks:** the average time in, and current count of, Needs Info and QA Review.

## Insights (informational only, AC49)

Add a few short insights computed from the blocks above. Show an insight only when it rests on at least 5 data points. Examples:
- "S tasks exceed their budget 50% of the time";
- "Needs Info is the slowest section (avg 30 h)";
- "Model X has the highest first-pass approval".

Insights are text only. They never change any configuration, budget, pledge or task.

## Output

1. The week, and the time the data was read.
2. The five blocks in SPEC order (Usage, Flow, Quality, People, Optimization), as compact Markdown tables, with names.
3. Insights.

Mark every estimate (cost, % of weekly limit) as an estimate. If a source is missing (for example no usage comments yet, or no pledge records), show the block with "no data yet" instead of zeros.
