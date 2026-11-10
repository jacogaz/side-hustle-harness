# Side hustle harness

Shared configuration and default harness for **Side Hustle**: team members publish tasks in Asana, and the agents of teammates with spare token capacity pick them up, implement, review and deliver them. The functional spec is [SPEC.md](SPEC.md).

This repo is installed by every team member. Target repos, where tasks are implemented, stay on GitLab.

## Layout

| Path | What it is | Spec |
|---|---|---|
| `config/global.yaml` | Global configuration, shared by the team. | §6 |
| `config/user.example.yaml` | Template for per-user configuration. | §5 |
| `config/user.yaml` | Your own per-user configuration (git-ignored; created by you). | §5 |
| `harness/DEFAULT_HARNESS.md` | Default harness, applied when a target repo has none. | E4, AC14 |
| `skills/slack/SKILL.md` | Slack skill: one thread per task, transition notices, mentions, reading replies. | §12, UC1, AC44 |

## Setup (each team member)

1. Clone or install this repo.
2. Copy the per-user template:
   ```sh
   cp config/user.example.yaml config/user.yaml
   ```
3. Edit `config/user.yaml`. To opt in, set `side_hustle_mode: true` and adjust the window, threshold, concurrency and threshold policy.

You can edit `config/user.yaml` at any time. The change applies from your agent's next poll cycle.

## Configuration reference

### Per-user (`config/user.yaml`)

| Key | Values | Meaning |
|---|---|---|
| `side_hustle_mode` | `true` / `false` | Opt-in. When `false`, your agent does not pick up or review tasks (AC1); owner agent duties still run. `start-side-hustle` turns it on (UC10). |
| `usage_window` | e.g. `5h`, `weekly` | Window over which your consumption is measured. |
| `usage_threshold_percent` | 0–100 | Your agent picks up tasks only while consumption is strictly below this % of the window (AC2, AC3). |
| `max_concurrency` | integer ≥ 1 | Maximum number of tasks claimed at the same time (AC4). |
| `on_threshold_crossed` | `finish` / `release` | What happens to the current task when the threshold is crossed mid-task (E5, AC19, AC20). |
| `poll_interval_minutes` | integer ≥ 1 | Minutes between two cycles of your side hustle loop, started once with `start-side-hustle` (UC10). |

### Global (`config/global.yaml`)

| Key | Meaning |
|---|---|
| `max_reviewer_attempts` | Doing ↔ To Review loops before the task moves to Needs Info. |
| `heartbeat_timeout_hours` | Hours without a heartbeat before a task in Doing or To Review is orphaned. |
| `needs_info_timeout_hours` | Hours before the single reminder to the owner, and again before the task is flagged "stalled". |
| `default_budget_per_size` | Default tokens for `S` / `M` / `L` estimates when the owner gives no budget. |
| `default_harness` | Path to the default harness file. |
| `slack_channel` | Name and id of the shared side hustle channel. |

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
