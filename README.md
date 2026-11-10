# Side hustle harness

Shared configuration and default harness for **Side Hustle**: team members publish tasks in Asana, and the agents of teammates with spare token capacity pick them up, implement, review and deliver them. The functional spec is [SPEC.md](SPEC.md).

This repo is installed by every team member. Target repos, where tasks are implemented, stay on GitLab.

## Layout

| Path | What it is | Spec |
|---|---|---|
| `config/global.yaml` | Global configuration, shared by the team. | §6 |
| `config/user.example.yaml` | Template for per-user configuration. | §5 |
| `config/user.yaml` | Your own per-user configuration (git-ignored). Written by `/start-side-hustle`. | §5 |
| `harness/DEFAULT_HARNESS.md` | Default harness, applied when a target repo has none. | E4, AC14 |
| `skills/start-side-hustle/SKILL.md` | The single trigger: one pledge question, writes `config/user.yaml`, starts the recurring cycle that runs the other skills. | §12, UC10, AC52–AC58 |
| `skills/slack/SKILL.md` | Slack skill: one thread per task, transition notices, mentions, reading replies. | §12, UC1, AC44 |
| `skills/create-task/SKILL.md` | Create-task skill: publishes a task with the full template, a budget and a Slack thread. | §12, UC1, AC6, AC8 |
| `skills/reviewer/SKILL.md` | Reviewer skill: one cycle step that reviews tasks in To Review with the four UC4 checks in an independent context. | §12, UC4, AC22–AC27 |
| `skills/pick-up/SKILL.md` | Pick-up skill: one cycle step that recovers orphaned tasks, claims, implements, delivers after review and releases. | §12, UC2, UC3, UC5, UC8 |
| `templates/task-description.md` | Asana task template: the description format and custom fields every skill writes and reads. | §7 |

## Setup (each team member)

1. Clone or install this repo.
2. In Claude Code, run `/start-side-hustle` and answer its one question: how much of your weekly Claude usage to pledge (a %; 0 = only follow up on your own tasks).

That's all.
- **What it does:** it writes `config/user.yaml` for you, with the confirmed defaults for everything else, and starts the side hustle cycle in your open session every 15 minutes. Nobody copies or edits the file by hand.
- **To change your pledge:** run `/start-side-hustle` again. It never starts a second cycle.
- **To stop:** close the session (and any other session where you ran it). Your tasks are then recovered by the heartbeat timeout.

## Configuration reference

### Per-user (`config/user.yaml`)

| Key | Values | Meaning |
|---|---|---|
| `pledge_percent` | 0–100 | % of your weekly Claude usage limit given to shared side hustle tasks this week. `0` means owner duties only. Your agent claims a task only if its budget fits in the pledge left this week (AC2, AC3, AC5). |
| `pledge_week` | ISO week, e.g. `2026-W46` | Calendar week (Monday to Sunday, same for the whole team) the pledge applies to. A new week keeps last week's pledge (E22). |
| `side_hustle_mode` | `true` / `false` | `true` when `pledge_percent` is above 0. When `false`, your agent does not pick up or review tasks (AC1); owner agent duties still run. |
| `max_concurrency` | integer ≥ 1 | Maximum number of tasks claimed at the same time (AC4). |
| `on_pledge_used_up` | `finish` / `release` | What happens to the current task when your pledge is used up mid-task (E5, AC19, AC20). Applies only once actual usage is measured; until then a claimed task's budget is reserved for the whole task. |
| `poll_interval_minutes` | integer ≥ 1 | Minutes between two cycles of your side hustle loop, started once with `start-side-hustle` (UC10). |

### Global (`config/global.yaml`)

| Key | Meaning |
|---|---|
| `max_reviewer_attempts` | Doing ↔ To Review loops before the task moves to Needs Info. |
| `heartbeat_timeout_hours` | Hours without a heartbeat before a task in Doing or To Review is orphaned. |
| `needs_info_timeout_hours` | Hours before the single reminder to the owner, and again before the task is flagged "stalled". |
| `default_budget_per_size` | Default budget for `S` / `M` / `L` estimates, as % of the weekly usage limit, when the owner gives no budget. |
| `default_harness` | Path to the default harness file. |
| `slack_channel` | Name and id of the shared side hustle channel. |
| `asana_project` | Gids of the Side Hustle Asana project, its sections (all seven) and its custom fields (Priority, Estimate, Budget, Last heartbeat) with their options. |

Global values change only through a commit to this repo. Values marked `confirmed default` are the defaults confirmed by the owner.

## Rules for skills that read this configuration

- **Read per-user config at the start of every poll cycle.**
  - Read `config/user.yaml` again at the start of each cycle; never cache it between cycles. This is how a change applies from the next poll cycle (AC7).
  - If the file is missing, treat side hustle mode as off.
- **Read global config at the start of every poll cycle**, the same way.
- **Use the default harness only as a fallback.**
  - When the target repo defines a harness, follow it (P4, AC13).
  - Only when it has none, apply `default_harness` and note it in the deliverable (AC14).
- **Don't let personal config override a repo harness.** The target repo's rules win over a user's personal configuration (AC13).
