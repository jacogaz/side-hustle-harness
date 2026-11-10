# Side Hustle: functional spec (PoC)

> What we are building, not how. No technology or architecture decisions.

## 1. Summary

Some team members run short on tokens, time or focus. Others have Claude tokens left unused in their current window. **Side Hustle** lets the first group publish tasks in Asana, and the **agents** of the second group pick them up, implement them, review them and deliver them **asynchronously and autonomously**. The task **owner** always reviews and approves the final result. Each user, owner or picker, runs **one skill once** (`start-side-hustle`), and from then on the whole flow runs by itself.

A **dashboard** shows team usage (tokens, models, tasks published, picked up and completed, leaderboard) and insights for optimizing the workflow.

## 2. Goals

1. Put the team's unused token capacity to work on real backlog items.
2. Make publishing a task cheap for the owner, and let someone else execute it without needing the owner's live context.
3. Make sure the workflow **never blocks** on a specific person or agent.
4. Make sure every delivery follows the target project's rules and is non-destructive.
5. Make usage and flow visible so they can be optimized.

## 3. Non-negotiable principles

| # | Principle |
|---|---|
| P1 | **Asana is the single source of truth** for instructions. Everything needed to continue a task lives in the Asana task (description and comments). Slack is for notifying and talking, never the only source of instructions. |
| P2 | **Everything is asynchronous.** No agent waits on a human or another agent; while waiting, it keeps working on other tasks. Once started, side hustle runs with no further human action. |
| P3 | **The workflow never blocks.** Any task can change assignee. Orphaned or stalled tasks are released automatically. |
| P4 | **The repository's harness wins.** Work is always executed under the target project's rules, never under rules that contradict them. |
| P5 | **No deploy without human review.** |
| P6 | **Only the owner** approves the deliverable and moves the task to Done. |

## 4. Actors

| Actor | Description |
|---|---|
| **Owner (User A)** | Overloaded person who publishes the task. Has the context, reviews in QA Review and marks Done. Runs `start-side-hustle` once so that A's owner agent runs. |
| **Owner agent** | A's agent, run by A's side hustle cycle (UC10) whatever A's side hustle mode or threshold. Fills in missing information on A's tasks that are in Needs Info, and clears the "stalled" flag once A has completed it. |
| **Picker (User B)** | Person with spare token capacity who runs `start-side-hustle` once, which turns on side hustle mode. |
| **Picker agent** | B's agent. Polls, claims, implements and delivers. |
| **Reviewer agent** | Independent agent (context separate from the implementer) that checks the deliverable against the task. |
| **Slack agent** | The agents' ability to post and listen in the side hustle channel. |
| **Team** | The whole team; sees the dashboard. |

Once a user runs `start-side-hustle`, the recurring cycle in that user's session plays that user's picker agent, reviewer agent and owner agent roles (UC10).

## 5. Per-user configuration

| Parameter | Description |
|---|---|
| Side hustle mode (on/off) | Opt-in that lets this user's agent pick up and review tasks. `start-side-hustle` turns it on. Owner agent duties run whether it is on or off. |
| Usage window | Window over which consumption is measured (e.g. weekly or 5h limit). |
| Usage threshold | % of the window below which the agent may pick up tasks. |
| Max concurrency | Number of tasks claimed at the same time. |
| On crossing the threshold mid-task | `finish` or `release`. |
| Poll interval | Time between two cycles of the user's side hustle loop (UC10). |

## 6. Global configuration (side hustle)

| Parameter | Description |
|---|---|
| Max reviewer attempts | Number of Doing ↔ To Review loops before moving to Needs Info. |
| Heartbeat timeout | Hours without a heartbeat after which a task in Doing or To Review is considered orphaned. |
| Needs Info timeout | Hours without an answer before a single reminder to the owner; if the reminder also gets no answer within it, the task is flagged "stalled" and released to Next by the owner agent or the owner. |
| Default budget per size | Default tokens for S/M/L that the create-task skill applies when the owner gives an estimate but no budget. |
| Default harness | Minimal shared guidelines applied when the repo defines no harness. |
| Slack channel | Shared side hustle channel. |

## 7. Task template (Asana)

| Field | Required | Notes |
|---|---|---|
| Goal | Yes | What needs to be achieved. |
| Deliverable type | Yes | Code / Documentation / Asana tasks. |
| Target resource | Yes | GitLab repo, Confluence space or Asana project. |
| Acceptance criteria | Yes | Verifiable list. |
| Out of scope / do not touch | Yes | What must not be modified. |
| Context / links | No | Specs, MRs, threads. |
| Priority | No | High / Medium / Low. |
| Estimate | No | S / M / L. |
| Token budget | Yes | If the owner gives none, the create-task skill sets it when publishing: the default for the estimate, or, with no estimate either, a budget it determines. |
| Last heartbeat | System | Updated by the agent currently working on the task while it is in Doing or To Review (the picker agent implementing or delivering, the reviewer agent reviewing). There is no heartbeat once the task is in QA Review; it starts again if the task returns to Doing. |

## 8. Lifecycle (Asana sections)

| Section | Meaning | Eligible to pick up? |
|---|---|---|
| **Open** | Published and ready | Yes |
| **Doing** | An agent is implementing; assigned to it | No |
| **To Review** | The reviewer agent is checking | No |
| **QA Review** | The owner is reviewing the deliverable | No |
| **Next** | The owner rejected it and left a comment in Asana on how to proceed, or it was released with its state documented | Yes, unless flagged "stalled" |
| **Needs Info** | Information is missing or a decision is needed; waiting for the owner agent or the owner | No |
| **Done** | Approved by the owner | No |

### Transitions

| From → To | Who | Trigger |
|---|---|---|
| — → Open | Owner (or the create-task skill) | Task created with all required fields |
| Open/Next → Doing | Picker agent | Claim: assigns the task to itself and moves it |
| Open/Next → Needs Info | Picker agent | Required fields are missing, ambiguous or contradict the repo harness |
| Doing → To Review | Picker agent | Implementation finished |
| To Review → Doing | Reviewer agent | A non-destructive failure is found and attempts remain |
| To Review → Needs Info | Reviewer agent | A destructive failure is found (the non-destructive check fails), or attempts are exhausted |
| To Review → QA Review | Picker agent | All checks pass; the picker agent delivers (UC5) and the owner is notified |
| QA Review → Next | Owner | Rejection with an Asana comment on how to proceed |
| QA Review → Done | Owner | Approval (and merge, final publication or deploy done by a human) |
| Doing → Needs Info | Picker agent | Budget exhausted before reaching To Review, or the task contradicts the repo harness |
| Doing/To Review/QA Review/Needs Info → Next | Picker / picker agent | Voluntary release |
| Doing/To Review → Next | Picker agent / any agent | Threshold crossed with `release` policy, or orphaned task |
| Needs Info → Open | Owner agent / owner | Information completed in Asana |
| Needs Info → Next | Owner agent / owner | Stalled: the single reminder got no answer within the Needs Info timeout |

When a task moves to Open or Next, it is **unassigned**. The assignment is what locks it for everyone else.

## 9. Use cases

### UC1: Publish a task
A (or A's agent, using the create-task skill) creates the task in the **Side Hustle** project with the full template and leaves it in Open. The Slack agent opens a thread for the task in the side hustle channel.

### UC2: Poll and claim
With side hustle mode on and consumption below the threshold, B's agent looks for tasks in Open or Next that meet **all** of the following:
- they are unassigned;
- they are not flagged "stalled";
- all required fields are filled in;
- their budget fits within the margin B has left before the threshold;
- B has not reached max concurrency.

It claims autonomously (assigns the task to itself, which means to B, and moves it to Doing) and posts in the Slack thread. If two agents claim the same task, the first one to become assignee wins; the other silently discards it and keeps looking.

### UC3: Implement
The agent executes the task using the target repo's harness (CLAUDE.md/AGENTS.md, skills, commands, hooks, tests, lint and conventions). If the repo has no harness, it applies the default harness and says so in the deliverable. While working, it updates the heartbeat periodically. Instructions are taken only from Asana.

### UC4: Review (reviewer agent)
An independent agent checks four things:
1. **Compliance:** every acceptance criterion is met.
2. **Scope:** nothing listed as out of scope was touched, and there are no changes unrelated to the task.
3. **Non-destructive:** no data deletion, irreversible migrations, force-pushes, permission or secret changes, or removal of files, tests or functionality that was not requested.
4. **Harness:** the repo's tests, lint and conventions pass.

While reviewing, it updates the heartbeat periodically. The result is recorded as a comment in Asana.

### UC5: Deliver
Once the reviewer passes all checks, the picker agent delivers, depending on the deliverable type:
- **Code:** a branch is pushed and a GitLab MR is opened, with a link to the task and a summary of changes.
- **Documentation:** published in Confluence as a draft or in a review space.
- **Asana tasks:** created in the target project.

While delivering, the picker agent updates the heartbeat. There is never a deploy. The task moves to QA Review and the Slack agent notifies the owner in the thread with a summary and links.

### UC6: Owner review
A reviews. If A approves, a human does the merge, final publication or deploy, and A marks Done. If A rejects, A comments in Asana on how to proceed and moves the task to Next. Any agent (including B's) can pick it up afterwards.

### UC7: Fill in information (owner agent)
A's agent polls A's tasks in Needs Info. It fills in what is missing **in Asana** (from its own context or by asking A on Slack, asynchronously) and moves the task back to Open. If nobody answers within the Needs Info timeout, a single reminder is sent to the owner. If the reminder also gets no answer, the task is flagged "stalled" on the dashboard, A's agent (or A) releases it to Next with no assignee, and Slack is notified. While flagged, it cannot be claimed; once the owner has completed the missing information in Asana, A's agent clears the flag and the task becomes eligible again.

### UC8: Release a task
B (or B's agent) can release a task at any time from Doing, To Review, QA Review or Needs Info; Done is final and cannot be released. Releasing a task in Open or Next is a no-op: nothing changes and no notification is sent. On a release, the state the task was in is documented in Asana (branch, last commit, what is left), the task moves to Next with no assignee, and Slack is notified.

### UC9: View the dashboard
The whole team sees the metrics, with names (section 11).

### UC10: Start side hustle
B (any user, including owners) runs the `start-side-hustle` skill once. It turns side hustle mode on and starts a recurring cycle in B's open session that repeats every poll interval, with no further human action. In each cycle, for B, it always does the owner agent duties, and does the picker and reviewer duties only while side hustle mode is on and B is below the threshold (work already underway is finished or released per B's policy):
- **Picker duties:** check side hustle mode, the threshold and max concurrency; claim eligible tasks (UC2); implement them with the heartbeat (UC3); release orphaned tasks (E6); deliver after the review passes (UC5).
- **Reviewer duties:** review tasks in To Review (UC4), including tasks B implemented, always with a context independent of the implementer's (AC22).
- **Owner agent duties for B's own tasks:** handle Needs Info, the reminder and stalled tasks (UC7); open and follow the Slack threads; copy replies into Asana; post the notices for transitions B made by hand.

When B turns side hustle mode off, or crosses the threshold (after finishing or releasing the current task, per B's policy), the cycle stops picking up and reviewing tasks by itself; the owner agent duties keep running. Running `start-side-hustle` again while B's cycle is active does nothing: each user has one active cycle. If B's session closes, B's tasks are recovered by the existing heartbeat timeout (E6).

## 10. Edge cases

| # | Case | Behavior |
|---|---|---|
| E1 | Two agents claim the same task | The first one to become assignee wins; the other discards it without commenting. |
| E2 | Required fields are missing | Not claimed. What is missing is commented in Asana, A is mentioned in the Slack thread, and the task moves to Needs Info. |
| E3 | The task contradicts the repo harness (e.g. "skip the tests") | Needs Info; the owner clarifies in Asana. |
| E4 | The repo has no harness | The default harness is applied and noted in the deliverable. |
| E5 | B crosses the threshold mid-task | Per B's policy: `finish` completes it and picks up nothing more; `release` documents the state and moves it to Next. |
| E6 | The agent disappears (crash, closed session) | When the heartbeat timeout expires, any polling agent releases it to Next with its state documented and notifies Slack. |
| E7 | The token budget runs out | The agent stops, documents the state in Asana and moves the task to Needs Info. |
| E8 | The budget does not fit B's margin | Not claimed; the task is skipped. |
| E9 | The reviewer finds a non-destructive failure | Back to Doing with the report, up to the max attempts; then Needs Info. |
| E10 | The reviewer finds a destructive failure | Straight to Needs Info, no retries. |
| E11 | The owner rejects | Next with an Asana comment; unassigned, and anyone can pick it up. |
| E12 | The owner does not answer in Needs Info | Single reminder, then flagged "stalled" on the dashboard and released to Next by the owner agent or the owner (unassigned, Slack notified). It cannot be claimed until the owner has completed the missing information in Asana and the owner agent clears the flag. |
| E13 | Information arrives only via Slack | It does not count as an instruction until it is in Asana; the owner agent copies it over. |
| E14 | The agent resuming a task has no prior context | It continues using only what is in Asana (description, comments and documented state). |
| E15 | The owner tries to pick up their own task | Allowed. No restriction. |
| E16 | B wants to abandon a task | B releases it to Next at any time from Doing, To Review, QA Review or Needs Info; not from Done. In Open or Next, release is a no-op with no notification. |
| E17 | Someone other than the owner moves a task to Done | Not valid: only the owner marks Done. |
| E18 | B's session closes while side hustle is running | The cycle stops; B's tasks in Doing or To Review are released by another agent when the heartbeat timeout expires (E6). |
| E19 | B turns side hustle mode off or crosses the threshold while the cycle runs | The cycle stops picking up and reviewing tasks by itself; with the threshold, the current task is first finished or released per B's policy (E5). Owner agent duties keep running. |
| E20 | B runs `start-side-hustle` while B's cycle is already active | Nothing happens; each user has one active cycle. |
| E21 | B's cycle reviews a task B implemented | Allowed, as long as the review runs with a context independent of the implementer's (AC22). |

## 11. Dashboard

**Visibility:** the whole team sees everything, with names.

| Block | Metrics |
|---|---|
| Usage | Tokens and cost per person, per model, per task and per phase (implementation, review, communication); % of window consumed per user. |
| Flow | Tasks per section; average time in each section; total Open → Done time; stalled or orphaned tasks. |
| Quality | % approved by the owner on the first pass; number of rejections (returns to Next); reviewer failures and their reasons. |
| People | Leaderboard of who publishes, picks up and completes the most. |
| Optimization | Actual consumption vs. budget and estimate; cost and success rate per model; bottlenecks (Needs Info, QA Review). |

**Insights:** computed from the metrics and **informational only**; they never change any configuration. For example: "S tasks exceed their budget 50% of the time".

## 12. Skills included in the side hustle harness

The side hustle harness (these skills, the default harness and the shared configuration) lives in a **GitHub repository** that every team member installs from. This is independent of the target repos where tasks are implemented, which remain on GitLab.

- **Create task:** guides the owner through publishing a task with the full template, including determining the budget.
- **Pick up tasks:** polling, eligibility filters, claiming, heartbeat and releasing.
- **Slack:** post in each task's thread, mention people and listen for replies.
- **Reviewer:** the four checks in UC4.
- **Fill in information:** poll own tasks in Needs Info and update them in Asana; clear the "stalled" flag once the information is complete.
- **Usage reporting:** publish metrics to the dashboard.
- **Start side hustle (`start-side-hustle`):** the single trigger a user runs once; it turns side hustle mode on and starts the recurring cycle that runs the user's picker, reviewer and owner agent duties every poll interval (UC10).

## 13. Pain points addressed

- Unused tokens while other people are overloaded.
- Small tasks nobody gets to for lack of time.
- Loss of context when delegating, solved by the template and by Asana as the source of truth.
- Fear of an agent breaking something: repo harness, independent reviewer, human review and no automatic deploys.
- Blocking on absent people: async flow, releases and timeouts.
- No visibility into usage: the dashboard.

## 14. Out of scope

1. Access and permission management (every agent is assumed to have access to GitLab, Asana, Confluence and Slack).
2. Production deploys done by agents.
3. Multiple teams, boards or channels.
4. Incentives, gamification or compensation beyond the leaderboard.
5. Auto-tuning of models, budgets or thresholds.
6. Handling of secrets or sensitive data inside tasks.
7. Tools other than GitLab, Asana, Confluence and Slack as task targets or communication channels (GitHub is used only to host the side hustle harness).
8. Tasks that require synchronous human interaction.
9. Smart prioritization or allocation between agents ("first one wins" applies).

## 15. Assumptions

- The budget is set by the owner; if missing, the create-task skill sets it when publishing (the default for the estimate, or a budget it determines when there is no estimate), so every published task has a budget.
- Documentation is published as a draft or in a review space until the owner approves it.
- All communication about a task goes to its thread in the side hustle channel.
- The reviewer has no access to the implementer's reasoning; it only sees the task and the deliverable.

---

## 16. Acceptance criteria

### Configuration and eligibility
- **AC1:** GIVEN B has side hustle mode off, WHEN a poll cycle runs, THEN B's agent neither looks for nor claims tasks.
- **AC2:** GIVEN B has the mode on and consumption above the threshold, WHEN a poll cycle runs, THEN B's agent claims no tasks.
- **AC3:** GIVEN B has the mode on and consumption below the threshold, WHEN there is an unassigned task in Open with all required fields filled in and a budget that fits B's margin, THEN B's agent claims it without asking for confirmation.
- **AC4:** GIVEN B has reached max concurrency, WHEN there are eligible tasks, THEN B's agent claims no more.
- **AC5:** GIVEN a task's budget exceeds the margin B has left before the threshold, WHEN the agent evaluates it, THEN it does not claim it and keeps looking.
- **AC6:** GIVEN the owner gives no budget, WHEN the create-task skill publishes the task, THEN it uses the default budget for the estimate, or determines a budget if there is no estimate either, and the task is created with it.
- **AC7:** GIVEN B changes their window, threshold, concurrency, threshold policy or poll interval, WHEN the next poll cycle runs, THEN the new configuration applies.

### Publish
- **AC8:** GIVEN A uses the create-task skill, WHEN it finishes, THEN the task exists in the Side Hustle project with all required fields, is in Open, and has a thread in the Slack channel.
- **AC9:** GIVEN a task in Open or Next is missing a required field, WHEN a picker agent evaluates it, THEN it does not claim it, comments in Asana on what is missing, mentions A in the Slack thread and moves it to Needs Info.

### Claim
- **AC10:** GIVEN an agent claims a task, WHEN it does so, THEN the task is assigned to B, moves to Doing, and the Slack thread is notified.
- **AC11:** GIVEN two agents try to claim the same task, WHEN the first one becomes assignee, THEN the second discards it without commenting and keeps looking.
- **AC12:** GIVEN an unassigned task in Next that is not flagged "stalled", WHEN any eligible agent polls, THEN it can claim it just like a task in Open.

### Implement
- **AC13:** GIVEN the target repo defines a harness, WHEN the agent implements, THEN it follows the repo's rules even if they contradict B's personal configuration.
- **AC14:** GIVEN the target repo defines no harness, WHEN the agent implements, THEN it applies the default harness and notes it in the deliverable.
- **AC15:** GIVEN the task asks for something that contradicts the repo harness, WHEN the agent detects it, THEN it does not do it, comments in Asana and moves the task to Needs Info.
- **AC16:** GIVEN an agent (picker or reviewer) is working on a task, WHEN it makes progress, THEN it updates the heartbeat periodically.
- **AC17:** GIVEN information exists only in Slack, WHEN the agent implements, THEN it does not treat it as an instruction until it is in Asana.
- **AC18:** GIVEN the budget runs out before reaching To Review, WHEN the agent detects it, THEN it stops, documents the state in Asana and moves the task to Needs Info.
- **AC19:** GIVEN B crosses the threshold mid-task with the `finish` policy, WHEN the agent keeps working, THEN it completes the task and claims no more.
- **AC20:** GIVEN B crosses the threshold mid-task with the `release` policy, WHEN the agent detects it, THEN it documents the state in Asana, unassigns the task, moves it to Next and notifies Slack.
- **AC21:** GIVEN the implementation is finished, WHEN the agent completes it, THEN it moves the task to To Review.

### Review
- **AC22:** GIVEN a task in To Review, WHEN the reviewer acts, THEN it is an agent with independent context that only sees the task and the deliverable.
- **AC23:** GIVEN the reviewer reviews, WHEN it finishes, THEN it leaves a report in Asana with the result of the four checks (compliance, scope, non-destructive, harness).
- **AC24:** GIVEN a non-destructive failure is found and attempts remain, WHEN the reviewer finishes, THEN the task goes back to Doing with the report and the implementer fixes it.
- **AC25:** GIVEN the max attempts have been reached, WHEN it fails again, THEN the task moves to Needs Info with the report.
- **AC26:** GIVEN the non-destructive check fails (a destructive failure), WHEN the reviewer finishes, THEN the task moves to Needs Info with no retries.
- **AC27:** GIVEN all checks pass, WHEN the reviewer finishes, THEN the picker agent makes the delivery (UC5) and moves the task to QA Review.

### Deliver
- **AC28:** GIVEN a code deliverable, WHEN it is delivered, THEN there is a pushed branch and a GitLab MR with a link to the task and a summary, and no merge or deploy.
- **AC29:** GIVEN a documentation deliverable, WHEN it is delivered, THEN it is published in Confluence as a draft or in a review space, with the link in the task.
- **AC30:** GIVEN an Asana-tasks deliverable, WHEN it is delivered, THEN the tasks exist in the target project and are linked from the original task.
- **AC31:** GIVEN a task moves to QA Review, WHEN it is delivered, THEN the Slack agent notifies the owner in the thread with a summary and links.
- **AC32:** GIVEN any delivery, WHEN an agent makes it, THEN it never includes a deploy.

### Owner review
- **AC33:** GIVEN a task in QA Review, WHEN the owner approves, THEN a human does the merge, final publication or deploy, and the owner moves it to Done.
- **AC34:** GIVEN someone who is not the owner, WHEN they try to move a task to Done, THEN it is not accepted as completed.
- **AC35:** GIVEN the owner rejects, WHEN they do so, THEN they leave an Asana comment on how to proceed, the task moves to Next with no assignee, and Slack is notified.

### Needs Info and owner agent
- **AC36:** GIVEN one of A's tasks is in Needs Info, WHEN A's agent polls, THEN it fills in what is missing in Asana (or asks A on Slack asynchronously, leaving the task in Needs Info) and moves it to Open once the information is in Asana.
- **AC37:** GIVEN a task in Needs Info with no answer when the timeout expires, WHEN the deadline passes, THEN a single reminder is sent to the owner.
- **AC38:** GIVEN the reminder also gets no answer, WHEN the deadline passes again, THEN the task is flagged "stalled" on the dashboard, the owner agent (or the owner) moves it to Next with no assignee, and Slack is notified; it cannot be claimed until the owner has completed the missing information in Asana and the owner agent clears the flag.
- **AC39:** GIVEN the owner answers only on Slack, WHEN the owner agent detects it, THEN it copies the answer into Asana before moving the task back to Open.
- **AC40:** GIVEN an agent is waiting on anything, WHEN it waits, THEN it keeps working on other tasks and does not block.

### Release and orphaned tasks
- **AC41:** GIVEN B or B's agent releases a task in Doing, To Review, QA Review or Needs Info, WHEN they do so, THEN the state is documented in Asana, the task moves to Next with no assignee, and Slack is notified; a task in Done cannot be released, and releasing a task in Open or Next changes nothing and sends no notification.
- **AC42:** GIVEN a task in Doing or To Review with no heartbeat for the timeout period, WHEN another agent polls, THEN it releases it to Next with the last known state and notifies Slack.
- **AC43:** GIVEN an agent resumes a released task, WHEN it starts, THEN it continues from the documented state (existing branch or draft), not from scratch.

### Communication
- **AC44:** GIVEN any task transition, WHEN it happens, THEN the notification goes to that task's thread in the side hustle channel.
- **AC45:** GIVEN any instruction or decision, WHEN it is recorded, THEN it lives in the Asana task.

### Dashboard
- **AC46:** GIVEN any team member, WHEN they open the dashboard, THEN they see usage, flow, quality and people metrics, with names.
- **AC47:** GIVEN an agent consumes tokens on a task, WHEN it reports its usage, THEN the dashboard attributes it to the person, model, task and phase.
- **AC48:** GIVEN completed tasks, WHEN the dashboard is viewed, THEN actual consumption vs. budget and estimate is shown.
- **AC49:** GIVEN enough data, WHEN the dashboard is viewed, THEN optimization insights are shown that never change any configuration.
- **AC50:** GIVEN stalled or orphaned tasks, WHEN the dashboard is viewed, THEN they are flagged.
- **AC51:** GIVEN tasks published, picked up and completed, WHEN the leaderboard is viewed, THEN it reflects who publishes, picks up and completes the most.

### Start side hustle
- **AC52:** GIVEN B runs `start-side-hustle` once, WHEN it starts, THEN side hustle mode is turned on and a recurring cycle runs in B's open session every poll interval with no further human action.
- **AC53:** GIVEN the cycle is running, WHEN a cycle runs, THEN it always performs the owner agent duties for B's own tasks, and performs B's picker duties and reviewer duties for tasks in To Review, including tasks B implemented (with a context independent of the implementer's), only while side hustle mode is on and B is below the threshold; work already underway is finished or released per B's policy.
- **AC54:** GIVEN the cycle is running, WHEN B turns side hustle mode off, THEN the cycle stops picking up and reviewing tasks by itself, and the owner agent duties keep running.
- **AC55:** GIVEN the cycle is running, WHEN B crosses the threshold, THEN the current task is finished or released per B's policy, the cycle stops picking up and reviewing tasks by itself, and the owner agent duties keep running.
- **AC56:** GIVEN B's session closes while the cycle is running, WHEN the heartbeat timeout expires, THEN B's tasks in Doing or To Review are released to Next by another agent (AC42), with no new mechanism.
- **AC57:** GIVEN B's cycle is already active, WHEN B runs `start-side-hustle` again, THEN nothing happens and no second cycle starts.
