---
name: create-task
description: Publish a side hustle task in Asana with the full template, a budget and a Slack thread. Use when an owner wants to hand off a task to the side hustle team.
---

# Create task skill (side hustle)

The owner, or the owner's agent on their behalf, uses this skill to publish a task in the Side Hustle Asana project (SPEC.md UC1). The task:
- has every template field (`${CLAUDE_PLUGIN_ROOT}/templates/task-description.md`);
- always has a budget, as % of the weekly Claude usage limit;
- is unassigned and in Open;
- has its own Slack thread.

The person running this skill is the task's **owner** (User A).

## 1. Read the configuration

Read `${CLAUDE_PLUGIN_ROOT}/config/global.yaml`. You need:
- `asana_project`: the project gid, the gid of its Open section, and the custom field and option gids;
- `default_budget_per_size` (% of the weekly limit: S 2, M 5, L 10);
- `slack_channel`.

Also read the project agent rules (`${CLAUDE_PLUGIN_ROOT}/templates/task-description.md`, "Project agent rules"), and apply them, for example the language of the task.

## 2. Collect every template field

Go through the fields of `${CLAUDE_PLUGIN_ROOT}/templates/task-description.md` in order. Ask the owner only for what they have not already given, and keep it short.
- **Task name:** a short title for the Asana task. Propose one from the goal and let the owner change it.
- **Goal** (required).
- **Deliverable type** (required): `Code`, `Documentation` or `Asana tasks`.
- **Target resource** (required): a link that matches the deliverable type (a GitHub repo URL `https://github.com/<owner>/<repo>` for Code, a Confluence space or an Asana project).
- **Acceptance criteria** (required): at least one verifiable item. If an item cannot be checked (for example "make it better"), ask the owner to make it verifiable.
- **Out of scope / do not touch** (required): at least one item. If the owner names nothing, write `nothing beyond the acceptance criteria` and confirm it with them.
- **Context / links**, **Priority**, **Estimate** (optional): empty means `none`.
- **Budget:** see step 3.

**Refuse to publish while a required field is missing or invalid.** Say which field is missing and ask for it. If the owner stops, create nothing.

## 3. Set the budget (AC6)

Use the first rule that applies:
1. **The owner gives a budget:** use it, as a positive whole % of the weekly limit. Source: `owner`.
2. **The owner gives an estimate but no budget:** use `default_budget_per_size` for that estimate. Source: `default for <S|M|L>`.
3. **Neither is given:**
   - judge which size (S, M or L) the task matches, from its goal, acceptance criteria and target;
   - use that size's default budget, with source `determined`;
   - leave **Estimate** as `none`, because it is the owner's field;
   - tell the owner the size you chose and the budget.

The task is never published without a budget.

## 4. Confirm and publish

1. Write the description exactly in the format of `${CLAUDE_PLUGIN_ROOT}/templates/task-description.md`.
2. Show the owner the task name, the description, and the Priority, Estimate and Budget values once. Publish when they confirm.
3. Create the task in the `asana_project`, in the `open` section, with **no assignee**:
   - the name is the task name and the description is the template text;
   - set the custom fields: Priority and Estimate to their option gids (left empty if `none`), and Budget as a fraction (5% → 0.05);
   - leave Last heartbeat empty.
4. Add an Asana comment recording the budget and its source, for example `Budget: 5% of the weekly limit (source: default for M)`.
5. Read the task back from Asana and check that:
   - it is in Open;
   - it is unassigned;
   - the description parses with every required field present;
   - Budget is set to the fraction you wrote.

   Fix anything that does not match before going on.

## 5. Open the Slack thread (UC1, AC8)

Follow **"Open the thread" in `${CLAUDE_PLUGIN_ROOT}/skills/slack/SKILL.md`**. You are the owner's agent, so you are the authorized thread creator. Do not create the thread any other way.
- **Publication is complete only when** the thread exists and its `Side hustle Slack thread:` comment is recorded in Asana. Then tell the owner the task is published, with the Asana link and the thread link.
- **If the thread cannot be posted right now:** keep the task, and do not create a second task or let any other agent open the thread. Tell the owner the publication is **pending: thread not yet open**, with the Asana link. You, the owner's agent, retry asynchronously. On each later owner-agent cycle you find the task with no thread and open it through the Slack skill. The publication becomes complete once the permalink is recorded.

## Asana is the record (AC45)

- Every instruction and decision about the task is written in Asana: in the description, custom fields and budget comment at publication, or as comments later. This includes field values agreed in conversation and the chosen budget and its source.
- The Slack thread only announces and links the task. Never put information there that is not also in Asana.
