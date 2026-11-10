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

The cycle is **active** when `config/.cycle.lock` exists and its `last_cycle` time is younger than twice `poll_interval_minutes`.

- **Active (AC57, E20):**
  - tell B the pledge is updated;
  - start nothing: the running cycle picks up the new pledge and mode at its next run, since every skill re-reads the config.
- **Not active:**
  1. Write `config/.cycle.lock` with this session's id and `last_cycle` = now.
  2. Start the recurring cycle in this session with the Claude Code `/loop` command at `poll_interval_minutes`, for example:
     ```
     /loop 15m run one side hustle cycle (skills/start-side-hustle/SKILL.md, "One cycle")
     ```
  3. Run the first cycle right away.

Tell B in one line:
- the pledge, and whether side hustle mode is on;
- that the cycle runs every `poll_interval_minutes` while this session stays open;
- that closing the session stops it, and that the heartbeat timeout then recovers B's tasks (E18, AC56).

## One cycle

Run these steps in order, once, then return. Never wait on a person or another agent (AC40).

1. **Housekeeping:**
   - Re-read `config/global.yaml` and `config/user.yaml` (AC7).
   - Set `last_cycle` in `config/.cycle.lock` to now.
   - **New week (E22, AC58):** if `pledge_week` is not the current ISO week, keep `pledge_percent`, set `pledge_week` to the current week, and say once in the session: "New week: keeping your side hustle pledge of <n>%." Do not wait for an answer.
2. **Owner agent duties for B's own tasks (always, AC53).** These are the tasks B created. Use `skills/slack/SKILL.md`:
   - open the thread for any own task that has none, as the authorized thread creator;
   - post pending Slack notices;
   - read replies in the threads of own tasks in Needs Info, or in Next flagged "stalled", and copy the owner's answers into Asana;
   - post the notice for any own-task transition B made by hand that has no notice yet.

   If `skills/fill-in/SKILL.md` exists, run it next for the rest of the owner duties (Needs Info completion, reminder, stalled flag; UC7). If it does not exist, skip it.
3. **Picker duties:** run one pass of `skills/pick-up/SKILL.md`.
   - Its step 1 always does orphan and partial-claim recovery (E6, AC42).
   - It keeps B's work underway going, under its existing rules.
   - It makes new claims only with side hustle mode on and pledge left (AC1–AC5).
4. **Reviewer duties:** run one pass of `skills/reviewer/SKILL.md`. It reviews only with side hustle mode on, and needs no pledge. It may review B's own task in an independent context (E21, AC53).
5. **Usage reporting:** if `skills/usage-reporting/SKILL.md` exists, run it. Otherwise skip it.

Mode off (pledge 0) or no pledge left only stops new claims and reviews, through those skills' own gates. The cycle itself keeps running for the owner duties and recovery (AC54, AC55).

A cycle can take longer than the poll interval, for example while implementing a task. In that case the next run starts when this one finishes. Pick-up keeps the heartbeat updated during long work.
