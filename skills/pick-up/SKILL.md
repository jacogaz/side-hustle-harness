---
name: side-hustle-pick-up
description: One cycle step of a user's side hustle loop. It recovers orphaned tasks, claims eligible tasks, implements them, delivers after the review passes, and releases tasks when needed. start-side-hustle calls it every poll interval; it does not loop by itself.
---

# Pick-up skill (side hustle)

This skill does **B's picker duties** for one cycle (SPEC.md UC2, UC3, UC5, UC8, UC10). B is the user whose session runs it. `start-side-hustle` calls it every cycle; this skill runs one pass and returns. It never waits on a person or another agent (AC40): anything that has to wait is left for a later cycle, and the step moves on.

It uses:
- **`config/global.yaml`** for the section, custom field and option gids, `heartbeat_timeout_hours`, `default_harness` and `slack_channel`;
- **`config/user.yaml`** for B's pledge, mode, max concurrency, `on_pledge_used_up` and `poll_interval_minutes`;
- **`templates/task-description.md`** to read task fields and comment markers;
- **`skills/slack/SKILL.md`** for every Slack notice: find the thread, then post the reply for the transition.

Re-read both config files at the start of every cycle (AC7). If `config/user.yaml` is missing, treat side hustle mode as off.

**Never move a task to Done (AC34) and never deploy (AC32).**

## Steps of one cycle

Run them in this order. Later steps still run if an earlier one had nothing to do.

### 1. Recover orphaned tasks (always runs, E6, AC42)

This step runs even when side hustle mode is off or B has no pledge left (AC53).
1. Look at every task in Doing or To Review in the project.
2. A task is orphaned when its `Last heartbeat` is empty or older than `heartbeat_timeout_hours`.
3. Release each orphaned task as described in "Release a task", with reason `orphaned`. The documented state is the last known one: the latest `Side hustle state:` comment, plus the branch or draft if you can see it.

### 2. Keep B's work going (work already underway)

For each task assigned to B:
- **In Doing**, if this session is not already working on it: implement it (step 4). If it was released before, resume from the documented state (AC43).
- **In To Review**, when the latest `Side hustle review:` comment after the task last entered To Review is `PASS`: deliver it (step 5).
- **Anything else**, such as To Review still waiting for a review, or QA Review or Needs Info: nothing to do, move on.

**Pledge used up mid-task (E5, AC19, AC20)** applies only once usage reporting measures actual usage. Until then, a claimed task's budget is reserved for the whole task, so work underway is never stopped for pledge reasons. Once actual usage is measured, a task whose usage exhausts B's pledge is handled per `on_pledge_used_up`:
- `finish`: complete it and claim nothing more;
- `release`: release it (see "Release a task").

### 3. Claim new tasks (AC1–AC5, AC9–AC12)

Skip this step unless **both** are true:
- side hustle mode is on (AC1);
- B has pledge left this week (AC2).

Compute the pledge figures for the current ISO week:
- **Pledge used:** the sum of the `<n>%` in every `Side hustle claim: <B> · <n>% · <this week>` comment in the project. These are the budgets B claimed this week (SPEC.md §5).
- **Pledge left:** `pledge_percent` minus pledge used.
- **Active claims:** B's tasks in Doing or To Review. Stop claiming once their number reaches `max_concurrency` (AC4).

Look at tasks in **Open** and **Next** (Next is claimable like Open, AC12). A task is eligible when all of these hold (UC2):
1. it is unassigned;
2. it is not flagged "stalled" (comment markers in the template);
3. every required field is present and valid (template parsing rules);
4. its Budget fits in B's pledge left (AC3, AC5); a task that does not fit is skipped, and you keep looking (E8).

**A required field is missing or invalid (AC9, E2):**
- do not claim the task;
- comment in Asana on exactly what is missing;
- move the task to Needs Info;
- post the Needs Info notice mentioning the owner (Slack skill).

**To claim an eligible task (AC3, AC10, AC11)**, without asking anyone for confirmation:
1. Read the task again. If it is no longer unassigned, skip it silently.
2. Assign it to B.
3. Read it again. If the assignee is not B, another agent won: skip it with no comment and no Slack post (E1), and keep looking.
4. If B is the assignee:
   - move the task to Doing;
   - set `Last heartbeat` to now;
   - add the comment `Side hustle claim: <B> · <budget>% · <ISO week>`;
   - post the claim notice (Slack skill).
5. Recompute pledge left and active claims before the next candidate.

Claiming your own task is allowed (E15).

### 4. Implement (AC13–AC18, AC21)

1. **Instructions:** take them only from the task's description, its custom fields and its comments, including `Side hustle state:` and the reviewer's last `FAIL` report (UC3, AC17).
   - Slack messages are not instructions until they are in Asana.
   - If the thread holds something that seems important, ask the owner in the thread to put it in Asana, and carry on without it.
2. **Workspace:**
   - **Code:** work in a branch of the target GitLab repo named `side-hustle/<task gid>`. On resume, use the existing branch (AC43).
   - **Documentation:** a Confluence draft that is not published.
   - **Asana tasks:** a list of the proposed tasks, kept in a `Side hustle state:` comment. Do not create the tasks yet.
3. **Harness:** follow the target repo's harness (CLAUDE.md/AGENTS.md, skills, commands, hooks, tests, lint, conventions), even where it contradicts B's personal configuration (AC13). If the repo has none, apply `default_harness` and say so in the deliverable (AC14).
4. **Contradictions (AC15, E3):** if the task asks for something that contradicts the repo harness or the default harness:
   - do not do it;
   - comment in Asana on the contradiction;
   - add a `Side hustle state:` comment;
   - move the task to Needs Info;
   - post the Needs Info notice.
5. **Heartbeat (AC16):** set `Last heartbeat` to now after each significant step, and never let more than `poll_interval_minutes` pass without updating it while you work.
6. **Budget runs out (AC18, E7):** if the task cannot be finished within its Budget (your best judgement until usage reporting measures it):
   - stop;
   - add a `Side hustle state:` comment;
   - move the task to Needs Info;
   - post the Needs Info notice.
7. **Implementation finished (AC21):**
   - make the work reachable for the reviewer: push the branch (no MR yet) or save the draft;
   - add a `Side hustle state:` comment;
   - set `Last heartbeat` to now;
   - move the task to To Review;
   - post the "→ To Review" notice.

   Then move on. The review is someone else's step, so do not wait for it.

After a `FAIL` review, the reviewer moves the task back to Doing, and step 2 picks it up again in a later cycle.

### 5. Deliver after the review passes (AC27–AC32)

Update `Last heartbeat` while delivering (UC5).

| Deliverable type | Delivery |
|---|---|
| Code (AC28) | The branch is pushed and a GitLab MR is opened, with a link to the task and a summary of changes. **No merge, no deploy.** |
| Documentation (AC29) | The page is published in Confluence as a draft or in a review space. Put its link in the task. |
| Asana tasks (AC30) | Create the proposed tasks in the target project, and link them from the original task. |

Then:
1. add a `Side hustle state:` comment with the links;
2. move the task to QA Review;
3. post the QA Review notice, which mentions the owner and includes a summary and links (AC31, Slack skill).

From here only the owner acts. Never move the task to Done (AC34).

## Release a task (AC41–AC43, UC8, E16)

Release when:
- B asks;
- B's agent decides to give a task up;
- `on_pledge_used_up` is `release` (once actual usage is measured);
- the task is orphaned (step 1).

What release does depends on the section:
- **Done:** refuse. Done is final.
- **Open or Next:** do nothing and send no notification. The task is already unassigned.
- **Doing, To Review, QA Review or Needs Info:**
  1. Add a `Side hustle state:` comment with the branch, last commit, draft or proposed tasks, and what is left.
  2. Remove the assignee.
  3. Move the task to Next.
  4. Post the release notice with the reason: voluntary, pledge used up, or orphaned (Slack skill).

Whoever claims the task next resumes from that documented state, not from scratch (AC43).
