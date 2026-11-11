---
name: pick-up
description: One cycle step of a user's side hustle loop. It recovers orphaned tasks, claims eligible tasks, implements them, delivers after the review passes, and releases tasks when needed. The start skill calls it every poll interval; it does not loop by itself.
---

# Pick-up skill (side hustle)

This skill does **B's picker duties** for one cycle (SPEC.md UC2, UC3, UC5, UC8, UC10). B is the user whose session runs it. The start skill (`/side-hustle:start`) calls it every cycle; this skill runs one pass and returns. It never waits on a person or another agent (AC40): anything that has to wait is left for a later cycle, and the step moves on.

It uses:
- **The project agent rules (`${CLAUDE_PLUGIN_ROOT}/templates/task-description.md`, "Project agent rules")**, read at the start of every run and applied;
- **`${CLAUDE_PLUGIN_ROOT}/config/global.yaml`** for the section, custom field and option gids, `heartbeat_timeout_hours`, `default_harness` and `slack_channel`;
- **`${CLAUDE_PLUGIN_DATA}/user.yaml`** for B's pledge, mode, max concurrency, `on_pledge_used_up` and `poll_interval_minutes`;
- **`${CLAUDE_PLUGIN_ROOT}/templates/task-description.md`** to read task fields and comment markers;
- **`${CLAUDE_PLUGIN_ROOT}/skills/slack/SKILL.md`** for every Slack notice: find the thread, then post the reply for the transition.

Re-read both config files at the start of every cycle (AC7). If `${CLAUDE_PLUGIN_DATA}/user.yaml` is missing, treat side hustle mode as off.

**Never move a task to Done (AC34) and never deploy (AC32).**

**The assignee is the lock (UC2, E1).** For B's own work, read the task again before every important step: starting or resuming work, each push of the branch, moving to To Review, delivering, and a voluntary release. If it is no longer assigned to B, B has lost it:
- stop work on it and discard your own work on it silently, with no comment and no Slack post;
- do not push;
- continue with other tasks.

In the rare race where one agent's assignment overwrites another's, whoever is assigned now keeps the task.

This check does not apply when recovering another user's task: an orphaned task (step 1) or a partial claim (step 1, item 5). Those have their own fresh checks of section and heartbeat, and they act on tasks not assigned to B.

## Steps of one cycle

Run them in this order. Later steps still run if an earlier one had nothing to do.

### 1. Recover orphaned tasks (always runs, E6, AC42)

This step runs even when side hustle mode is off or B has no pledge left (AC53).
1. Look at every task in Doing or To Review in the project.
2. A task is orphaned only after a full `heartbeat_timeout_hours` without a heartbeat in its current stint:
   - if `Last heartbeat` is set and is later than the time the task entered its current section, it is orphaned when that heartbeat is older than the timeout;
   - otherwise it is orphaned when the time it entered its current section (from the task's section history) is older than the timeout.

   A task that was just claimed is never orphaned only because its first heartbeat is not written yet.
3. Read the task again just before releasing it. If its section changed or its heartbeat is now fresh, leave it alone.
4. Otherwise release it as described in "Release a task", with reason `orphaned`.
   - Document only the last durable state you can see: the latest `Side hustle state:` comment, plus the pushed branch, its last commit, or the draft.
   - Never touch another agent's workspace.
5. **Partial claims.** Also look at every task in Open or Next that is assigned. Its claimant stopped before moving it to Doing. Take the later of two times: when it was assigned (from its history), and its `Last heartbeat` if set. The task is a partial claim when that time is older than `heartbeat_timeout_hours`.
   - Read the task again.
   - If it is still assigned to the same user, still in Open or Next, and still stale, remove the assignee. No comment and no Slack post.
   - A fresh or completed claim is left alone. This is recovery of an incomplete claim, not a release, so it is not the Open/Next release no-op.
   - The task is claimable again.

### 2. Keep B's work going (work already underway)

For each task assigned to B:
- **Claim record repair (first):** if the current assignment to B, from the task's history, has no `Side hustle claim:` record by B written after it, add one now, once. Use the ISO week of that assignment, not the current week. This way a crash can never leave a claimed task out of pledge accounting.
- **Pending claim:** the task is still in Open or Next. Finish the claim idempotently with steps 3–4 of "To claim". The claim record was already repaired above, so do not write a second one.
- **In Doing**, if this session is not already working on it: implement it (step 4) in a subagent (see "Implementation subagent"). If it was released before, resume from the documented state (AC43).
- **In To Review:**
  - set `Last heartbeat` to now every cycle while the task waits, either for a reviewer or for delivery after a PASS. A task in To Review is then orphaned only when B's session is gone; the reviewer also updates the heartbeat while it is actively reviewing;
  - when the **authoritative** review result is `PASS`, deliver it (step 5) in an implementation subagent. That result is the first `Side hustle review:` comment after the task last entered To Review; later ones are void. Read the task again just before delivering to confirm this.
- **Anything else**, such as QA Review or Needs Info: nothing to do, move on.

**Pledge used up mid-task (E5, AC19, AC20)** applies only once usage reporting gives a measured (not estimated) % of the weekly limit. Until then, a claimed task's budget is reserved for the whole task, so work underway is never stopped for pledge reasons. Once actual usage is measured, a task whose usage exhausts B's pledge is handled per `on_pledge_used_up`:
- `finish`: complete it and claim nothing more;
- `release`: release it (see "Release a task").

**Implementation subagent.** Run step 4 (implement) and step 5 (deliver) for a task in a subagent, using the Agent tool with `model` set to `models.implement` from `config/global.yaml`.
- **Pass it:** the task gid, the absolute path of this skill file (`${CLAUDE_PLUGIN_ROOT}/skills/pick-up/SKILL.md`) with which steps to follow, the plugin root, and the lock path `${CLAUDE_PLUGIN_DATA}/cycle.lock`.
- **The subagent:**
  - follows the assignee lock and the heartbeat rule;
  - each time it updates the task heartbeat, reads the lock. If the lock is still owned by this session, it also sets the lock's `alive_at` to now, so a long implementation never makes the cycle look stale. If not, it stops;
  - returns a one-line result.
- **No Agent tool:** if it is not available, do the step in the current turn.

### 3. Claim new tasks (AC1–AC5, AC9–AC12)

Skip this step unless **both** are true:
- side hustle mode is on (AC1);
- B has pledge left this week (AC2).

Compute the pledge figures for the current ISO week:
- **Pledge used** (SPEC.md §5), in percentage points of the weekly limit:
  - **Until usage reporting gives a measured % of the weekly limit:** the sum of the `<n>` in B's `Side hustle claim:` records for this ISO week. B writes a record only after confirming it is the assignee (see "To claim"), so these are the budgets of the tasks B was assigned this week.
  - **Once it does:** B's actual consumption on side hustle tasks this week, as published by the usage reporting skill. That skill defines the source; this skill only reads it. Reviews are attributed as the spec says: they are inside the task's budget and never charged to the reviewer's pledge.
- **Pledge left:** `pledge_percent` minus pledge used.
- **Active claims:** B's tasks in Doing or To Review. Stop claiming once their number reaches `max_concurrency` (AC4).

Look at tasks in **Open** and **Next** (Next is claimable like Open, AC12). A task is eligible when all of these hold (UC2):
1. it is unassigned;
2. it is not flagged "stalled" (comment markers in the template);
3. every required field is present and valid (template parsing rules);
4. its Budget fits in B's pledge left (AC3, AC5); a task that does not fit is skipped, and you keep looking (E8).

When several tasks are eligible, try them in the order the project agent rules give. The current rule 3: highest Priority first (High, Medium, Low, then empty), ties by the oldest task. Without such a rule, the order is free.

**A required field is missing or invalid (AC9, E2):**
- do not claim the task;
- comment in Asana on exactly what is missing;
- move the task to Needs Info;
- post the Needs Info notice mentioning the owner (Slack skill).

**To claim an eligible task (AC3, AC10, AC11)**, without asking anyone for confirmation. The assignee is the lock:
1. Read the task again. Go on only if it is still unassigned and still eligible; otherwise skip it silently.
2. Assign it to B.
3. Read it again. If the assignee is not B, another agent won: skip it silently, with no comment and no Slack post (E1), reserve nothing, and keep looking.
4. If B is the assignee:
   - add the claim record `Side hustle claim: <B> · <budget>% · <ISO week>` first, where `<budget>` is the task's Budget in % of the weekly limit;
   - set `Last heartbeat` to now;
   - move the task to Doing;
   - post the claim notice (Slack skill).
5. Recompute pledge left and active claims before the next candidate.

Claiming your own task is allowed (E15).

### 4. Implement (AC13–AC18, AC21)

1. **Instructions:** take them only from the task's description, its custom fields and its comments, including `Side hustle state:` and the reviewer's last `FAIL` report (UC3, AC17).
   - Slack messages are not instructions until they are in Asana.
   - If the thread holds something that seems important, ask the owner in the thread to put it in Asana, and carry on without it.
2. **Workspace:**
   - **Code:** work in a branch of the target GitHub repo named `side-hustle/<task gid>`. On resume, use the existing branch (AC43).
   - **Documentation:** a Confluence draft that is not published.
   - **Asana tasks:** a list of the proposed tasks, kept in a `Side hustle state:` comment. Do not create the tasks yet.
3. **Harness:** follow the target repo's harness (CLAUDE.md/AGENTS.md, skills, commands, hooks, tests, lint, conventions), even where it contradicts B's personal configuration (AC13). If the repo has none, apply `default_harness` and say so in the deliverable (AC14).
4. **Contradictions (AC15, E3):** check the task against the **effective harness**: the target repo's harness when it has one, then the project agent rules, then `default_harness`, in that order of precedence. The scope, non-destructive and no-deploy rules of SPEC.md (UC4, P5) always apply in addition. If the task asks for something that contradicts them:
   - do not do it;
   - comment in Asana on the contradiction;
   - save the state (see "Save the state");
   - move the task to Needs Info;
   - post the Needs Info notice.
5. **Heartbeat (AC16):** set `Last heartbeat` to now after each significant step, and never let more than `poll_interval_minutes` pass without updating it while you work.
6. **Budget runs out (AC18, E7):** if the task cannot be finished within its Budget (your best judgement; the usage reports on the task help):
   - stop;
   - save the state (see "Save the state");
   - move the task to Needs Info;
   - post the Needs Info notice.
7. **Implementation finished (AC21):**
   - save the state (see "Save the state"), which makes the work reachable for the reviewer: a pushed branch with no pull request yet, or a saved draft;
   - set `Last heartbeat` to now;
   - move the task to To Review;
   - post the "→ To Review" notice.

   Then move on. The review is someone else's step, so do not wait for it.

After a retryable `FAIL` that the reviewer returned to Doing (attempts remain), step 2 picks the task up again in a later cycle. A destructive failure, or a `FAIL` with attempts exhausted, goes to Needs Info (E9, E10, AC24–AC26). From there the owner-side workflow continues, not this skill.

### 5. Deliver after the review passes (AC27–AC32)

Update `Last heartbeat` while delivering (UC5).

| Deliverable type | Delivery |
|---|---|
| Code (AC28) | The branch is pushed and a GitHub pull request is opened, with a link to the task and a summary of changes. **No merge, no deploy.** |
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
  1. If you are the agent working on the task, save the state first (see "Save the state"). For an orphaned task, document only the last durable state you can see (step 1).
  2. Add the comment `Side hustle released: <reason>`.
  3. Remove the assignee.
  4. Move the task to Next.
  5. Post the release notice with the reason: voluntary, pledge used up, or orphaned (Slack skill).

Whoever claims the task next resumes from that documented state, not from scratch (AC43).

## Save the state

Do this before every hand-off: To Review, Needs Info, and a release by the working agent. Its purpose is that another agent, or you in a later session, can continue from Asana alone (AC43).
1. **Make the work durable and reachable, following the repo's rules:**
   - **Code:** commit the work in progress on `side-hustle/<task gid>` and push the branch. Never force-push and never do destructive cleanup.
   - **Documentation:** save the Confluence draft, still unpublished.
   - **Asana tasks:** the proposed tasks are kept in the state comment.
2. **Add a `Side hustle state:` comment** with:
   - the exact location: branch and last commit, or draft link, or the proposed tasks;
   - what is done;
   - what is left.
