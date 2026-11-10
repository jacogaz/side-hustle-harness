# Side hustle task template (Asana description), v1

Every side hustle task keeps its template fields in the **Asana task description** (the Side Hustle project has no custom fields). The create-task skill writes it, and every other skill reads it with the parsing rules below. Fields follow SPEC.md §7.

## Format

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

## Priority
<High | Medium | Low | none>

## Estimate
<S | M | L | none>

## Token budget
<integer> tokens (source: <owner | default for S | default for M | default for L | determined>)

## Last heartbeat
<none | YYYY-MM-DDTHH:MMZ by <agent's user>>
```

## Fields

| Section | Required | Valid value |
|---|---|---|
| Goal | Yes | Non-empty text. |
| Deliverable type | Yes | Exactly one of `Code`, `Documentation`, `Asana tasks`. |
| Target resource | Yes | One link, matching the deliverable type: a GitLab repo for Code, a Confluence space for Documentation, an Asana project for Asana tasks. |
| Acceptance criteria | Yes | At least one `- ` list item; each item is verifiable. |
| Out of scope / do not touch | Yes | At least one `- ` list item. Write `- nothing beyond the acceptance criteria` if the owner names nothing specific. |
| Context / links | No | Free text, or `none`. |
| Priority | No | `High`, `Medium`, `Low` or `none`. |
| Estimate | No | `S`, `M`, `L` or `none`. |
| Token budget | Yes | A positive integer followed by `tokens` and the source in parentheses. |
| Last heartbeat | System | `none` at publication. Only the agent currently working on the task updates this line (SPEC.md §7). |

## Parsing rules (for every skill)

- **Marker line.** The first non-empty line must be `Side hustle task (template v1)`.
- **Section bodies.** A section's value is the text between its heading and the next `## ` heading, trimmed.
- **Optional fields.** An empty value or `none` means "not set".
- **Required fields.** A required field is **missing** if its section is absent, empty, `none`, or has a value outside the valid values above. A task with a missing required field is not eligible (SPEC.md E2, AC9).
- **Token budget.** The budget is the first integer in the section.
- **Instructions.** Everything an agent needs to know comes from this description and the task's comments (SPEC.md P1). Later clarifications are added as Asana comments, or by the owner editing the description in this format.
- **Editing.** Agents other than the owner change only the `Last heartbeat` section. Its exact update rules belong to the skill that works on the task.
