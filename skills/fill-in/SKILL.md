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
- **What is needed:** given by the latest agent comment in the stint that says why the task went to Needs Info. That can be:
  - a list of missing or invalid fields (pick-up);
  - a harness contradiction, or the budget running out (pick-up);
  - a review report with `Outcome: attempts exhausted` or `destructive` (reviewer).
- **An answer:** a comment by A, or a `From Slack (A, ...)` copy, written after the point the step refers to. Comments by agents never count as answers.
- **Facts versus decisions:** A's agent may fill in a fact from its own context, such as a link, a field value, or a detail from the target or the task's links. It writes it with its source. A **decision** always needs A's answer: a contradiction, accepting a destructive change, a bigger budget, or what to do after attempts are exhausted. The agent never decides it for A.

## 1. Tasks in Needs Info

For each of A's tasks in Needs Info, read it again, then take the first of these that applies.

1. **The information is complete in Asana.** Either the required fields now parse as valid, or A's answers settle every point of what is needed, with nothing left to ask.
   1. Write the settled values into the description or the custom fields (template editing rule for the owner agent).
   2. Move the task to Open. Moving to Open leaves it unassigned (SPEC.md §8).
   3. Post the "Needs Info → Open" notice (Slack skill).
2. **A's agent can fill in what is missing from its own context** (facts only):
   1. Add a comment `Side hustle fill-in: <what> (source: <where it comes from>)`.
   2. Write the values into the description or the custom fields.
   3. Then handle the task as in item 1.
3. **A has not been asked yet in this stint:**
   1. Ask A in the task's thread, mentioning A, with one short question (Slack skill, "Owner agent asking the owner on Slack").
   2. Add the comment `Side hustle asked: <question>`.
   3. Leave the task in Needs Info (AC36).
4. **No answer since the stint started, `needs_info_timeout_hours` have passed, and no reminder was sent in this stint (AC37):**
   1. Send the single reminder in the thread, mentioning A and saying what is still missing.
   2. Add the comment `Side hustle reminder`.
5. **No answer since the reminder, and `needs_info_timeout_hours` have passed again (AC38, E12):** release the task as stalled.
   1. Add `Side hustle stalled`. Add it first, so the task is never claimable in Next without the flag.
   2. Add `Side hustle released: stalled`.
   3. Remove the assignee.
   4. Move the task to Next.
   5. Post the stalled notice mentioning A (Slack skill).
6. **Otherwise:** leave the task as it is, and look at it again next cycle.

A partial answer counts as an answer. Ask the follow-up question in the thread (and add `Side hustle asked:`), but do not restart the reminder timing.

## 2. Stalled tasks in Next

For each of A's tasks in Next that is flagged "stalled", meaning its latest stalled marker is `Side hustle stalled`, read it again. Once the information is complete in Asana, by the same test as item 1 above, using answers after the `Side hustle stalled` comment:
1. Write the settled values into the description or the custom fields.
2. Add `Side hustle stalled cleared`.

The task **stays in Next** and becomes claimable again. Do not move it to Open.

## Rules

- **Asana is the record (AC45):** every value and decision is in Asana before it is used. Slack content counts only once it is copied into Asana.
- **Owner's task only:** never act on another owner's task, never claim, implement or review, and never move a task to Done.
