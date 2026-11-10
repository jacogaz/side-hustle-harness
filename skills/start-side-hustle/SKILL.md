---
name: start-side-hustle
description: The single side hustle trigger. Asks how much of your weekly Claude usage to pledge, writes config/user.yaml, and starts one recurring side hustle cycle in this Claude Code session that runs your owner, picker and reviewer duties every poll interval. Run it again to change your pledge.
---

# Start side hustle (`/start-side-hustle`)

Running this skill is the whole opt-in (SPEC.md UC10, AC52–AC58). After it, nothing else needs to be done by hand. The user who runs it is **B**; that includes owners, who need it so that their owner agent runs.

## When the user runs `/start-side-hustle`

### 1. Ask the one question

Ask exactly:

> How much of your weekly Claude usage do you want to give to shared Side Hustle tasks this week? (a % of your weekly limit; 0 = only follow up on my own tasks)

Accept a whole number from 0 to 100. Ask nothing else.

### 2. Write `config/user.yaml` (nobody edits it by hand)

- **If the file does not exist:** create it from `config/user.example.yaml`, so every other setting gets its confirmed default.
- **If it exists:** keep all of B's other settings.

Then set:
- `pledge_percent`: the answer;
- `pledge_week`: the current ISO week, for example `2026-W46` (calendar week, Monday to Sunday);
- `side_hustle_mode`: `true` when the pledge is above 0, otherwise `false`.

### 3. Start the cycle once (one active cycle per user)

The lock `config/.cycle.lock` names the **owner session**, the one session whose cycle is active. It holds `session_id` and `alive_at`. The owner refreshes `alive_at` at least every `poll_interval_minutes`, also during long work (see "One cycle"). The lock is **live** while `alive_at` is younger than 2 × `poll_interval_minutes`, and **stale** otherwise.

**Steps:**
1. **The lock is live,** whichever session owns it: the cycle is active (AC57, E20). Only the pledge and the mode changed, and the next run reads them. Schedule nothing.

   If the owner is **another session**, tell B once, plainly: "A side hustle cycle in another session checked in <N> minutes ago. If you closed that session, run /start-side-hustle again after <HH:MM>." `<HH:MM>` is when the lock becomes stale: `alive_at` + 2 × `poll_interval_minutes`.
2. **No lock, or a stale one:** start a fresh cycle in this session.
   1. Write this session as owner, with `alive_at` = now.
   2. Read the lock again. If another session wrote it in between, treat the lock as live (step 1).
   3. Create this session's recurring Claude Code job with `/loop` at `poll_interval_minutes`, unless this session already has one, for example:
      ```
      /loop 15m run one side hustle cycle (skills/start-side-hustle/SKILL.md, "One cycle")
      ```
   4. Run the first cycle right away.

   The old cycle, if its session ever resumes, stops itself through the ownership check in "One cycle".

Tell B in one line:
- the pledge, and whether side hustle mode is on;
- that the cycle runs every `poll_interval_minutes` while this session stays open;
- that closing the session stops it, and that the heartbeat timeout then recovers B's tasks (E18, AC56); running `/start-side-hustle` again later starts a fresh cycle.

## One cycle

Run these steps in order, once, then return. Never wait on a person or another agent (AC40).

1. **Ownership (fencing):** read `config/.cycle.lock`.
   - **This session is the owner:** set `alive_at` = now and go on.
   - **Otherwise:** this cycle was replaced. Delete this session's side hustle job and stop. A replaced cycle never continues.

   Repeat this check before each later step, and at least every `poll_interval_minutes` during long work (for example each time pick-up updates a task heartbeat).
2. **Housekeeping:**
   - Re-read `config/global.yaml` and `config/user.yaml` (AC7).
   - **Job renewal:** Claude Code scheduled jobs expire 7 days after creation. If this session's job was created more than 6 days ago, create the replacement `/loop` job first, then delete the old one. The session always keeps exactly one job.
   - **New week (E22, AC58):** if `pledge_week` is not the current ISO week, keep `pledge_percent`, set `pledge_week` to the current week, and say once in the session: "New week: keeping your side hustle pledge of <n>%." Do not wait for an answer.
3. **Owner agent duties for B's own tasks (always, AC53).** These are the tasks B created. Use `skills/slack/SKILL.md`:
   - open the thread for any own task that has none, as the authorized thread creator;
   - post pending Slack notices;
   - read replies in the threads of own tasks in Needs Info, or in Next flagged "stalled", and copy the owner's answers into Asana;
   - post the notice for any own-task transition B made by hand that has no notice yet.

   If `skills/fill-in/SKILL.md` exists, run it next for the rest of the owner duties (Needs Info completion, reminder, stalled flag; UC7). If it does not exist, skip it.
4. **Picker duties:** run one pass of `skills/pick-up/SKILL.md`.
   - Its step 1 always does orphan and partial-claim recovery (E6, AC42).
   - It keeps B's work underway going, under its existing rules.
   - It makes new claims only with side hustle mode on and pledge left (AC1–AC5).
5. **Reviewer duties:** run one pass of `skills/reviewer/SKILL.md`. It reviews only with side hustle mode on, and needs no pledge. It may review B's own task in an independent context (E21, AC53).
6. **Usage reporting:** if `skills/usage-reporting/SKILL.md` exists, run it. Otherwise skip it.

The two gates are separate, and each is enforced by the skill it belongs to:
- **Mode off (pledge 0)** stops new claims **and** reviews.
- **No pledge left** stops **only** new claims. Reviews go on, including of B's own reserved task in an independent context.

Owner duties, recovery and work already underway keep their own rules. The cycle itself keeps running (AC54, AC55).

A cycle can take longer than the poll interval, for example while implementing a task. In that case the next run starts when this one finishes. During long work, pick-up keeps the task heartbeat updated, and the owner check in step 1 keeps `alive_at` fresh.
