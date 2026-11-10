---
name: side-hustle-fill-in
description: One owner-duty cycle step. It unblocks the owner's own tasks in Needs Info (fill in from context or ask the owner, reopen once the information is in Asana), sends the single reminder, flags and releases stalled tasks, and clears the stalled flag once the information is complete. start-side-hustle calls it; it does not loop by itself.
---

# Fill-in skill (side hustle, owner agent)

This skill does the rest of **A's owner agent duties** for one cycle (SPEC.md UC7, E12, E13, AC36–AC39). A is the user whose session runs it. It acts only on **A's own tasks**, the tasks A created; never on other owners' tasks. It runs one pass, never waits on anyone (AC40), and runs whatever A's side hustle mode or pledge is.

It uses:
- **`config/global.yaml`** for the section and custom field gids, and `needs_info_timeout_hours`;
- **`templates/task-description.md`** for the fields, the editing rules and the comment markers;
- **`skills/slack/SKILL.md`** for questions, the reminder and notices.

`start-side-hustle` runs the Slack skill's owner duties first, so owner replies given only on Slack are already copied into Asana as `From Slack (<owner>, <permalink>):` comments (AC39, E13).

## What counts

- **The Needs Info stint:** from when the task last entered Needs Info, according to its section history.
- **The hand-off.** These are the agent comments that caused this entry. They are usually written just **before** the move. Take the most recent ones written between the task entering its previous section and this Needs Info entry:
  - the pick-up comment on missing or invalid fields;
  - the pick-up comment on a harness contradiction;
  - the `Side hustle state:` of a budget stop;
  - or the **authoritative** review result of the To Review window the task came from, with `Outcome: attempts exhausted` or `destructive`. Void results never count.

  Older blockers from earlier stints do not count.
- **The open points:** split the hand-off into points. Each point is either a **field** (a missing or invalid template field) or a **decision**: a contradiction, a destructive change, a bigger budget, or what to do after attempts are exhausted.
- **An answer:** a comment by A, or a `From Slack (A, ...)` copy, written after the stint started. Comments by agents never count as answers.
- **When a point is resolved:**
  - **A field point** is resolved when that field now parses as valid. A's agent may also fill it itself, with a fact from its own context such as a link, a value, or a detail from the target or the task's links.
  - **A decision point** is resolved only by A's answer that settles it. The agent never decides it for A.
  - **A destructive point** is resolved only when A changes what the task asks, so that the work needs no prohibited action. The always-applicable destructive restrictions (UC4, default harness rule 3) cannot be waived.
- **Complete:** every open point is resolved and the template parses with every required field valid. Valid fields alone never settle a decision point.

## 1. Tasks in Needs Info

For each of A's tasks in Needs Info, read it again, work out its open points, and take the first of these that applies.

1. **Complete.**
   1. Write the settled values into the description or the custom fields (template editing rule for the owner agent).
   2. Read the task again, and check every required field parses with the settled value.
   3. **Remove the assignee**, then move the task to Open.
   4. Check the task is in Open and unassigned. The previous picker's claim record stays as it is.
   5. Post the "Needs Info → Open" notice (Slack skill).
2. **A field point the agent can fill from its own context:**
   1. Add a comment `Side hustle fill-in: <what> (source: <where it comes from>)`.
   2. Write the value.
   3. Re-check whether the task is now complete.
3. **Some open point has not been asked about since the last answer:**
   1. Ask A in the task's thread, mentioning A, in one short message listing only those points (Slack skill, "Owner agent asking the owner on Slack").
   2. Add `Side hustle asked: <points>`.
   3. Leave the task in Needs Info (AC36).

   Never ask again about a point that is already asked and still pending.
4. **Open points remain, `needs_info_timeout_hours` have passed since the stint started, and no reminder was sent in this stint (AC37):**
   1. Send the single reminder in the thread, mentioning A and listing the points still open.
   2. Add `Side hustle reminder`.
5. **Open points remain, and `needs_info_timeout_hours` have passed since the reminder (AC38, E12):** release the task as stalled.
   1. Add `Side hustle stalled`. Add it first, so the task is never claimable in Next without the flag.
   2. Add `Side hustle released: stalled`.
   3. Remove the assignee.
   4. Move the task to Next.
   5. Post the stalled notice mentioning A (Slack skill).
6. **Otherwise:** leave the task as it is, and look at it again next cycle.

Partial answers resolve the points they settle, and never restart either timer. Both timers depend only on points still being open, so a partial answer can never stop the single reminder or the stalled release.

## 2. Stalled tasks in Next

For each of A's tasks in Next that is flagged "stalled", meaning its latest stalled marker is `Side hustle stalled`, read it again.
- Its open points are those of the Needs Info stint that ended in the stall.
- Answers count from the start of that stint.

Once it is **complete** by the same test:
1. Write the settled values into the description or the custom fields.
2. Read the task again, and check the values parse.
3. Add `Side hustle stalled cleared`.

The task **stays in Next** and becomes claimable again. Do not move it to Open.

## Rules

- **Asana is the record (AC45):** every value and decision is in Asana before it is used. Slack content counts only once it is copied into Asana.
- **Owner's task only:** never act on another owner's task, never claim, implement or review, and never move a task to Done.
