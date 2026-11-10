---
name: side-hustle-create-task
description: Publish a side hustle task in Asana with the full template, a token budget and a Slack thread. Use when an owner wants to hand off a task to the side hustle team.
---

# Create task skill (side hustle)

The owner, or the owner's agent on their behalf, uses this skill to publish a task in the Side Hustle Asana project (SPEC.md UC1). The task:
- has every template field (`templates/task-description.md`);
- always has a token budget;
- is unassigned and in Open;
- has its own Slack thread.

The person running this skill is the task's **owner** (User A).

## 1. Read the configuration

Read `config/global.yaml`. You need:
- `asana_project`: the project gid and the gid of its Open section;
- `default_budget_per_size`;
- `slack_channel`.

## 2. Collect every template field

Go through the fields of `templates/task-description.md` in order. Ask the owner only for what they have not already given, and keep it short.
- **Task name:** a short title for the Asana task. Propose one from the goal and let the owner change it.
- **Goal** (required).
- **Deliverable type** (required): `Code`, `Documentation` or `Asana tasks`.
- **Target resource** (required): a link that matches the deliverable type (GitLab repo, Confluence space or Asana project).
- **Acceptance criteria** (required): at least one verifiable item. If an item cannot be checked (for example "make it better"), ask the owner to make it verifiable.
- **Out of scope / do not touch** (required): at least one item. If the owner names nothing, write `nothing beyond the acceptance criteria` and confirm it with them.
- **Context / links**, **Priority**, **Estimate** (optional): empty means `none`.
- **Token budget:** see step 3.

**Refuse to publish while a required field is missing or invalid.** Say which field is missing and ask for it. If the owner stops, create nothing.

## 3. Set the token budget (AC6)

Use the first rule that applies:
1. **The owner gives a budget:** use it, as a positive integer number of tokens. Source: `owner`.
2. **The owner gives an estimate but no budget:** use `default_budget_per_size` for that estimate. Source: `default for <S|M|L>`.
3. **Neither is given:**
   - judge which size (S, M or L) the task matches, from its goal, acceptance criteria and target;
   - use that size's default budget, with source `determined`;
   - leave **Estimate** as `none`, because it is the owner's field;
   - tell the owner the size you chose and the budget.

The task is never published without a budget.

## 4. Confirm and publish

1. Write the description exactly in the format of `templates/task-description.md`, with `Last heartbeat` set to `none`.
2. Show the owner the task name and the description once, and publish when they confirm.
3. Create the task in the `asana_project`, in the `open` section, with **no assignee**. The name is the task name and the description is the template text.
4. Read the task back from Asana and check that:
   - it is in Open;
   - it is unassigned;
   - the description parses with every required field present.

   Fix anything that does not match before going on.

## 5. Open the Slack thread (UC1, AC8)

Follow **"Open the thread" in `skills/slack/SKILL.md`**. You are the owner's agent, so you are the authorized thread creator. Do not create the thread any other way.
- **If the thread cannot be posted right now:** the task still stands. The owner agent opens the thread the first time it sees the task with no thread (Slack skill).
- **Afterwards:** tell the owner the task is published, with the Asana link and the thread link (or "thread pending").

## Asana is the record (AC45)

- Every instruction and decision about the task is written in Asana: in the description at publication, or as comments later. This includes field values agreed in conversation and the chosen budget and its source.
- The Slack thread only announces and links the task. Never put information there that is not also in Asana.
