---
name: side-hustle-reviewer
description: One cycle step of a user's side hustle loop. It independently reviews tasks in To Review with the four checks of UC4, records the result in Asana, and moves the task on. start-side-hustle calls it every poll interval; it does not loop by itself.
---

# Reviewer skill (side hustle)

This skill does **R's reviewer duties** for one cycle (SPEC.md UC4, UC10, E9, E10, E21, AC22–AC27). R is the user whose session runs it. It runs one pass and returns, and it never waits on anyone (AC40).

It uses:
- **`config/global.yaml`** for the section and custom field gids, `max_reviewer_attempts`, `heartbeat_timeout_hours` and `default_harness`;
- **`config/user.yaml`** for R's side hustle mode and `poll_interval_minutes`;
- **`templates/task-description.md`** for task fields and comment markers;
- **`skills/slack/SKILL.md`** for notices.

Re-read both config files every cycle. **Gate:**
- run only while side hustle mode is on;
- R needs no pledge left, because reviews do not consume R's pledge; their cost is inside the task's budget (SPEC.md §5, AC53).

**Never deliver, merge, deploy or move a task to Done.** Delivery is the picker's job (pick-up skill, step 5).

## 1. Pick a task to review

Look at tasks in **To Review**. R may review any of them, including a task R implemented (E21), as long as the review runs in an independent context (step 3).

The **review window** of a task starts when it last entered To Review. Skip the task if, in this window, either:
- a `Side hustle review:` result already exists, because the review is done;
- or a `Side hustle review started: <reviewer>` comment younger than `heartbeat_timeout_hours` exists, because another review is in progress. The task's `Last heartbeat` cannot tell this, since the picker also refreshes it while the task waits. An older start marker is abandoned, and the task can be reviewed again.

To take a task:
1. Read it again.
2. Add `Side hustle review started: <R>`.
3. Set `Last heartbeat` to now.

Review one task at a time.

## 2. Keep the review valid (lock check)

Before each important step (starting the checks, posting the result, moving the task), read the task again. Abandon the review silently, with no comment and no Slack post, when any of these is true:
- the task is no longer in To Review;
- its assignee changed;
- a `Side hustle review:` result now exists in this review window.

Only the **first** result in a review window counts. While reviewing, set `Last heartbeat` at least every `poll_interval_minutes`, so the task is not treated as orphaned (pick-up step 1, AC16).

## 3. Review in an independent context (AC22)

Run the checks in a **fresh agent context**, never in the context that implemented the task. This applies even when R implemented it.

Give that context only the following.
- **The task:**
  - its description and custom fields (template v1);
  - the owner's comments: instructions and clarifications;
  - earlier `Side hustle review:` reports.
- **The deliverable**, located through the latest `Side hustle state:` comment:
  - Code: the branch `side-hustle/<task gid>` against its base;
  - Documentation: the Confluence draft;
  - Asana tasks: the proposed tasks.

Use only the location from the state comment; ignore any explanation or reasoning the implementer wrote there. Do not give the context the implementer's session, its notes or reasoning, or any Slack messages.

For code, check out the branch in a clean workspace of your own, and never push to it.

## 4. The four checks (UC4, AC23)

1. **Compliance:** every acceptance criterion is met. List each criterion with pass or fail and the evidence.
2. **Scope:** nothing listed under "Out of scope / do not touch" was changed, and there are no changes unrelated to the task.
3. **Non-destructive:** the deliverable does none of these, unless the task explicitly requested the removal of files, tests or functionality:
   - no data deletion;
   - no irreversible migration;
   - no force-push;
   - no permission or secret change;
   - no removal of files, tests or functionality.
4. **Harness:** the **effective harness** passes: the target repo's harness (tests, lint, conventions) when it has one, otherwise `default_harness`. If the default harness applies, the deliverable must say so (AC14).

**Failure classes:**
- If check 3 fails, the failure is **destructive**.
- Any other failed check is **non-destructive**.

## 5. Record the result and move the task (AC23–AC27)

Do the lock check (step 2) first. Then add **one** Asana comment. Its first line is `Side hustle review: PASS` or `Side hustle review: FAIL`. The comment contains:
- the outcome line: `Outcome: pass`, `Outcome: retry (attempt <n> of <max>)`, `Outcome: attempts exhausted`, or `Outcome: destructive`;
- the result of each of the four checks, with its evidence;
- for a FAIL, what has to change.

**Counting attempts:**
- `<max>` is `max_reviewer_attempts`.
- The returns to Doing used so far are the `Outcome: retry` reports since the task's latest `Side hustle claim:` record. A new claimer starts with fresh attempts.

Then apply the outcome that matches the result:

| Result | Outcome | Then |
|---|---|---|
| All four checks pass (AC27) | `pass` | Leave the task in To Review. The picker's next cycle delivers it and moves it to QA Review (pick-up step 2 and step 5). No section change, so no Slack notice. |
| Non-destructive failure, fewer than `<max>` returns used (AC24, E9) | `retry (attempt n of max)` | Move the task to Doing; it keeps its assignee. Post the "To Review → Doing" notice with attempt `<n>` of `<max>` (Slack skill). The picker resumes from the report. |
| Non-destructive failure, `<max>` returns already used (AC25) | `attempts exhausted` | Move the task to Needs Info. Post the Needs Info notice mentioning the owner. |
| Destructive failure (AC26, E10) | `destructive` | Move the task to Needs Info immediately, with no retries. Post the Needs Info notice mentioning the owner. |

After a Needs Info outcome, the owner-side workflow continues (fill-in skill). The deliverable stays where the state comment says. The reviewer changes nothing in it.

Then return; the next task waits for the next cycle.
