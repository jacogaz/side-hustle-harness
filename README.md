# Side hustle harness

Shared configuration and default harness for **Side Hustle**: team members publish tasks in Asana, and the agents of teammates with spare token capacity pick them up, implement, review and deliver them. The functional spec is [SPEC.md](SPEC.md).

This repo is a **Claude Code plugin** (`side-hustle`) with its own marketplace (`side-hustle-harness`). Every team member installs it once from GitHub and updates it from there. Target repos, where tasks are implemented, stay on GitLab.

## Layout

| Path | What it is | Spec |
|---|---|---|
| `config/global.yaml` | Global configuration, shared by the team. | §6 |
| `config/user.example.yaml` | Template for per-user configuration. | §5 |
| `${CLAUDE_PLUGIN_DATA}/user.yaml` | Your own per-user configuration, outside the plugin (`~/.claude/plugins/data/side-hustle-side-hustle-harness/user.yaml`), so plugin updates never replace it. Written by `/side-hustle:start`. | §5 |
| `${CLAUDE_PLUGIN_DATA}/cycle.lock` | Marks your active side hustle cycle (same folder). | UC10 |
| `.claude-plugin/plugin.json`, `.claude-plugin/marketplace.json` | Plugin and marketplace manifests. | §12 |
| `harness/DEFAULT_HARNESS.md` | Default harness, applied when a target repo has none. | E4, AC14 |
| `skills/start/SKILL.md` | The single trigger: one pledge question, writes your `user.yaml`, starts the recurring cycle that runs the other skills. | §12, UC10, AC52–AC58 |
| `skills/slack/SKILL.md` | Slack skill: one thread per task, transition notices, mentions, reading replies. | §12, UC1, AC44 |
| `skills/create-task/SKILL.md` | Create-task skill: publishes a task with the full template, a budget and a Slack thread. | §12, UC1, AC6, AC8 |
| `skills/dashboard/SKILL.md` | Dashboard (`/side-hustle:dashboard`): reads Asana and shows the usage, flow, quality, people and optimization blocks with names, plus informational insights. Read-only. | §11, AC46–AC51 |
| `skills/usage-reporting/SKILL.md` | Usage reporting: one cycle step that reports usage per person, model, task and phase, and the weekly pledge, as Asana comments. | §11, AC47, AC48 |
| `skills/fill-in/SKILL.md` | Fill-in skill: owner-duty cycle step for Needs Info (fill in, ask, reopen), the single reminder, stalled release and clearing. | §12, UC7, AC36–AC39 |
| `skills/reviewer/SKILL.md` | Reviewer skill: one cycle step that reviews tasks in To Review with the four UC4 checks in an independent context. | §12, UC4, AC22–AC27 |
| `skills/pick-up/SKILL.md` | Pick-up skill: one cycle step that recovers orphaned tasks, claims, implements, delivers after review and releases. | §12, UC2, UC3, UC5, UC8 |
| `templates/task-description.md` | Asana task template: the description format and custom fields every skill writes and reads. | §7 |

## Setup (each team member)

In Claude Code:

1. Add the marketplace: `/plugin marketplace add jacogaz/side-hustle-harness`
2. Install the plugin: `/plugin install side-hustle@side-hustle-harness`. On Claude Code 2.1.275 or later, `/plugin install side-hustle --marketplace jacogaz/side-hustle-harness` does steps 1 and 2 in one go.
3. Run `/reload-plugins`, or restart Claude Code.
4. Run `/side-hustle:start` and answer its one question: how much of your weekly Claude usage to pledge (a %; 0 = only follow up on your own tasks).

That's all.
- **Commands:** they are `/side-hustle:<skill>`, for example `/side-hustle:start`, `/side-hustle:create-task` and `/side-hustle:dashboard`. The short form, for example `/start`, also works when no other command has that name.
- **Updates:** in a terminal, run `claude plugin update side-hustle@side-hustle-harness`, then run `/reload-plugins` in Claude Code. Inside Claude Code you can instead use `/plugin` → Installed → side-hustle → Update now. Your `user.yaml` is kept.
- **Renamed trigger:** the trigger was renamed from `/side-hustle:start-side-hustle` to `/side-hustle:start`. If you had a cycle running under the old name, run `/side-hustle:start` once after updating, because the old recurring job calls a command that no longer exists. If it says a cycle is still live, run it again at the time it gives.
- **What it does:** it writes your `user.yaml` (`~/.claude/plugins/data/side-hustle-side-hustle-harness/`) for you, with the confirmed defaults for everything else, and starts the side hustle cycle in your open session every 15 minutes. Nobody copies or edits the file by hand.
- **To change your pledge:** run `/side-hustle:start` again. It never starts a second cycle.
- **To stop:** close the session. Your tasks are then recovered by the heartbeat timeout. Run `/side-hustle:start` again to start a fresh cycle.
- **Known PoC limitation:** if you close a session and run `/side-hustle:start` in a new one within two poll intervals (about 30 minutes), the old cycle still looks alive. Only your pledge is updated, and no cycle is started. The skill tells you when to run it again, which is once the old check-in is stale.

## Configuration reference

### Per-user (`${CLAUDE_PLUGIN_DATA}/user.yaml`, template `config/user.example.yaml`)

| Key | Values | Meaning |
|---|---|---|
| `pledge_percent` | 0–100 | % of your weekly Claude usage limit given to shared side hustle tasks this week. `0` means owner duties only. Your agent claims a task only if its budget fits in the pledge left this week (AC2, AC3, AC5). |
| `pledge_week` | ISO week, e.g. `2026-W46` | Calendar week (Monday to Sunday, same for the whole team) the pledge applies to. A new week keeps last week's pledge (E22). |
| `side_hustle_mode` | `true` / `false` | `true` when `pledge_percent` is above 0. When `false`, your agent does not pick up or review tasks (AC1); owner agent duties still run. |
| `max_concurrency` | integer ≥ 1 | Maximum number of tasks claimed at the same time (AC4). |
| `on_pledge_used_up` | `finish` / `release` | What happens to the current task when your pledge is used up mid-task (E5, AC19, AC20). Applies only once actual usage is measured; until then a claimed task's budget is reserved for the whole task. |
| `poll_interval_minutes` | integer ≥ 1 | Minutes between two cycles of your side hustle loop, started once with `/side-hustle:start` (UC10). |

### Global (`config/global.yaml`)

| Key | Meaning |
|---|---|
| `max_reviewer_attempts` | Doing ↔ To Review loops before the task moves to Needs Info. |
| `heartbeat_timeout_hours` | Hours without a heartbeat before a task in Doing or To Review is orphaned. |
| `needs_info_timeout_hours` | Hours before the single reminder to the owner, and again before the task is flagged "stalled". |
| `default_budget_per_size` | Default budget for `S` / `M` / `L` estimates, as % of the weekly usage limit, when the owner gives no budget. |
| `default_harness` | Path to the default harness file. |
| `slack_channel` | Name and id of the shared side hustle channel. |
| `usage` | Usage reporting: estimated weekly limit in tokens, price per model, and the ledger task for pledges. | §11 |
| `asana_project` | Gids of the Side Hustle Asana project, its sections (all seven) and its custom fields (Priority, Estimate, Budget, Last heartbeat) with their options. |

Global values change only through a commit to this repo. Values marked `confirmed default` are the defaults confirmed by the owner.

## Rules for skills that read this configuration

- **Paths.** Shared files (`config/`, `templates/`, `harness/`, `skills/`) are read from the plugin root, `${CLAUDE_PLUGIN_ROOT}`. Per-user state (`user.yaml`, `cycle.lock`) lives in `${CLAUDE_PLUGIN_DATA}`, never inside the plugin directory, which updates replace.

- **Read per-user config at the start of every poll cycle.**
  - Read `${CLAUDE_PLUGIN_DATA}/user.yaml` again at the start of each cycle; never cache it between cycles. This is how a change applies from the next poll cycle (AC7).
  - If the file is missing, treat side hustle mode as off.
- **Read global config at the start of every poll cycle**, the same way.
- **Use the default harness only as a fallback.**
  - When the target repo defines a harness, follow it (P4, AC13).
  - Only when it has none, apply `default_harness` and note it in the deliverable (AC14).
- **Don't let personal config override a repo harness.** The target repo's rules win over a user's personal configuration (AC13).
