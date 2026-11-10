---
name: side-hustle-dashboard
description: Show the Side Hustle dashboard. It reads the Side Hustle Asana project and renders the usage, flow, quality, people and optimization blocks with names, plus informational insights. Read-only. Use when anyone on the team asks for the side hustle dashboard or metrics.
---

# Side Hustle dashboard (`/side-hustle-dashboard`)

Any team member runs this skill to see the dashboard (SPEC.md §11, AC46–AC51). It reads Asana and renders the result in the Claude session, with names. There is no server and no database.

**It is read-only.** It never writes to Asana, Slack or the config, and never changes a task, setting, budget or pledge (AC49).

## Inputs

- **`config/global.yaml`:** `asana_project` (project, sections and custom fields), `usage.ledger_task`, `heartbeat_timeout_hours` and `default_budget_per_size`.
- **Every task in the project except the ledger task:** read its name, creator, assignee, section and assignment history, custom fields (Budget, Estimate, Last heartbeat) and comments. The markers are defined in `templates/task-description.md`.
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
  - **orphaned (now):** the task is in Doing or To Review, and is orphaned under pick-up step 1's full rule for its current stint. The reference time is its `Last heartbeat` if that is later than its entry into the current section, otherwise the entry time, and it must be older than `heartbeat_timeout_hours`. Only currently orphaned tasks are flagged.
  - **Orphan releases (history):** a separate, dated list of this week's `Side hustle released: orphaned` events. They never flag a task that was later recovered or completed.

### Quality

- **First-pass approval:** the % of Done tasks that never went QA Review → Next.
- **Rejections:** the number of QA Review → Next moves, overall and per task.
- **Reviewer failures:** the authoritative `Side hustle review: FAIL` results, by outcome (retry, attempts exhausted, destructive) and by failed check (compliance, scope, non-destructive, harness), with each one's one-line reason.

### People (AC51)

A leaderboard, all time and this week:
- **publishes:** tasks created;
- **picks up:** `Side hustle claim:` records;
- **completes:** tasks that reached Done, credited to the picker who delivered them. That is the author of the latest `Side hustle claim:` record written before the task's last move to QA Review. If there is no such record, use the assignee at the moment of the Done move, from the task's assignment history. If neither is available, count the task as "unknown". Never fall back to the current assignee.

### Optimization (AC48)

- **Completed tasks:** for each Done task, show these, all as % of the weekly limit:
  - actual: the sum of the `~<pct>%` in all its usage lines, all users and phases;
  - Budget, and actual minus Budget;
  - the Estimate as its size (S/M/L) **and** its baseline, which is `default_budget_per_size` for that size in `config/global.yaml`, labelled "estimate baseline"; also actual minus that baseline;
  - for a task with no Estimate, show `none` with no baseline. Never treat the Budget as the Estimate.
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
