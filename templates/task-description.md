# Side hustle task template (Asana), v1

Every side hustle task keeps its template fields (SPEC.md §7) in two places:
- **Asana custom fields** of the Side Hustle project: Priority, Estimate, Budget and Last heartbeat. Their gids are in `config/global.yaml` under `asana_project.custom_fields`.
- **The task description:** all the other fields.

The create-task skill writes both, and every other skill reads them with the rules below.

## Description format

The description is plain text. It starts with the marker line, followed by one section per field: a heading line `## <Field>`, then the field's value on the following lines. Sections appear exactly in this order:

```
Side hustle task (template v1)

## Goal
<what needs to be achieved>

## Deliverable type
<Code | Documentation | Asana tasks>

## Target resource
<GitLab repo URL | Confluence space URL | Asana project URL>

## Acceptance criteria
- <verifiable criterion>
- <verifiable criterion>

## Out of scope / do not touch
- <what must not be modified>

## Context / links
<specs, MRs, threads; or "none">
```

## Custom fields

| Field | Type | Value |
|---|---|---|
| Priority | enum | Option `High`, `Medium` or `Low`; empty means not set. |
| Estimate | enum | Option `S`, `M` or `L`; empty means not set. |
| Budget | number, shown as a percentage | A % of the weekly Claude usage limit, stored as a **fraction**: `number_value` 0.05 means 5%. Whole percentages only (precision 0). |
| Last heartbeat | date | Date and time (`date_time`, UTC) of the last heartbeat; empty at publication. |

The option gids for Priority and Estimate are in `config/global.yaml`.

## Fields

| Field | Required | Valid value |
|---|---|---|
| Goal | Yes | Non-empty text. |
| Deliverable type | Yes | Exactly one of `Code`, `Documentation`, `Asana tasks`. |
| Target resource | Yes | One link, matching the deliverable type: a GitLab repo for Code, a Confluence space for Documentation, an Asana project for Asana tasks. |
| Acceptance criteria | Yes | At least one `- ` list item; each item is verifiable. |
| Out of scope / do not touch | Yes | At least one `- ` list item. Write `- nothing beyond the acceptance criteria` if the owner names nothing specific. |
| Context / links | No | Free text, or `none`. |
| Priority (custom field) | No | One of its options, or empty. |
| Estimate (custom field) | No | One of its options, or empty. |
| Budget (custom field) | Yes | A positive fraction (e.g. 0.05 = 5%). It is a % of the weekly Claude usage limit and is reserved from the claimer's pledge at claim (SPEC.md §5). |
| Last heartbeat (custom field) | System | Empty at publication. Set by the picker (the assignee) while the task is in Doing or To Review, including while it waits for review or delivery, and by the reviewer while reviewing (SPEC.md §7). |

## Parsing rules (for every skill)

- **Marker line.** The first non-empty line must be `Side hustle task (template v1)`.
- **Section bodies.** A section's value is the text between its heading and the next `## ` heading, trimmed.
- **Optional fields.** An empty value or `none` means "not set".
- **Required fields.** A required field is **missing** if its section or custom field is absent, empty, `none`, or has a value outside the valid values above. A task with a missing required field is not eligible (SPEC.md E2, AC9).
- **Budget.** Read the Budget custom field's `number_value` and multiply by 100 to get the % of the weekly limit (0.05 → 5%). Write it the same way (5% → 0.05).
- **Instructions.** Everything an agent needs to know comes from this description, these custom fields and the task's comments (SPEC.md P1). The description and the custom fields hold the canonical value of each field.
- **Editing.** Only these may edit the description or the custom fields:
  - **The owner**, in this format.
  - **The owner agent**, when it completes missing or corrected information (SPEC.md UC7, AC36). Before moving the task back to Open or clearing the "stalled" flag, it writes the resolved values, already recorded in Asana, into the affected sections or custom fields. The source answer stays in Asana (a comment, or a reply copied from Slack).
  - **The agent currently working on the task**, but only the `Last heartbeat` custom field. Its exact update rules belong to the skill that works on the task.

  No other agent edits the description or the custom fields.

## Comment markers (for every skill)

Skills recognize their own Asana comments by a fixed first line. Each marker is written only by the skill named here; any skill may read it.

| First line | Written by | Meaning |
|---|---|---|
| `Side hustle Slack thread: <permalink>` | Owner agent (Slack skill) | The task's Slack thread. |
| `Side hustle claim: <user> · <n>% · <ISO week>` | Picker agent (pick-up skill) | Claim record, written only after `<user>` is confirmed as the assignee. `<n>` is the task's Budget, in % of the weekly Claude usage limit, reserved from `<user>`'s pledge for that week. The assignee is the lock; this record is only for pledge accounting. |
| `Side hustle released: <reason>` | The releasing agent (pick-up skill) | The task was released to Next: voluntary, pledge used up, or orphaned. |
| `Side hustle state:` | Picker agent or any releasing agent | Documented state: branch, last commit, draft or proposed tasks, what is left. |
| `Side hustle review started: <user>` | Reviewer agent (reviewer skill) | `<user>` is reviewing the task. Other reviewers skip the task while this comment is younger than `heartbeat_timeout_hours`. |
| `Side hustle review: PASS` or `Side hustle review: FAIL` | Reviewer agent (reviewer skill) | The review result and report (SPEC.md UC4, AC23). The second line is `Outcome: pass`, `retry (attempt <n> of <max>)`, `attempts exhausted` or `destructive`. Only the first result after the task last entered To Review counts. |
| `Side hustle stalled` / `Side hustle stalled cleared` | Owner agent (fill-in skill) | The "stalled" flag. A task is flagged while its latest of these two comments is `Side hustle stalled`. |
