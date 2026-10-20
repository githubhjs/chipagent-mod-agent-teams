# Porting Notes

ChipAgents port of the `agent-teams` plugin from [wshobson/agents](https://github.com/wshobson/agents) (MIT, upstream commit 0874a27, upstream plugin version 1.0.3). Port version: `1.0.0-chipagents`.

## Global Changes

- **Agent Teams removed.** ChipAgents has no experimental Agent Teams. All uses of TeamCreate/TeamDelete, SendMessage (message, broadcast, shutdown_request/shutdown_response, plan_approval_request/response), TaskCreate/TaskUpdate/TaskList/TaskGet, `subagent_type`, `~/.claude/teams/` and `~/.claude/tasks/`, display modes (tmux, iTerm2, in-process) and the `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS` check are gone.
- **File protocol instead.** Each run lives in `.chipagents/team/<run-id>/`: `brief.md` (target, scope, context), `tasks.md` (id, owner, owned files/modules, wave, status), `contracts/` (frozen SV packages, interfaces, port lists), `findings/<agent>.md` (replaces messages to the lead), `questions/<agent>.md` (blockers), plus `report.md` for the lead's consolidated report (added by the port). Teammates write their findings file and return the same report as their final answer.
- **Waves instead of blockedBy/blocks.** Wave N starts after every wave N-1 task is done.
- **Lead = main agent.** The command's main agent creates the run directory and dispatches ChipAgents subagent profiles. User approval gates use plan mode or an explicit confirmation.
- **chip-swarm.** `team-review` and `team-debug` show the equivalent `chip-swarm investigate` run; `team-feature` shows `chip-swarm fanout` per wave followed by merges in dependency order; `team-shutdown` runs `chip-swarm clean`.
- **Hardware adaptation.** Web/software content (XSS, SQL injection, CSRF, N+1 queries, WCAG/ARIA, JWT, React/Express/Django examples) replaced by RTL/DV content: review dimensions functional, cdc, lint, protocol, timing, verification, security, dft (+ optional scripts); debug categories RTL logic, testbench/stimulus, reset/init/X, CDC/ordering, handshake/protocol, register programming/configuration, tool & environment, gate-level/SDF/timing; SystemVerilog contracts and DMA-based examples.
- **Tools and models.** Agent `tools:` use ChipAgents names (Bash, Read, Edit, Create, Grep, Search, List, Git); `model:` lines removed so agents inherit the main model.
- **Not carried over:** `.codex-plugin/plugin.json`, plugin `README.md` (written separately).

## Manifests and License

- `.claude-plugin/marketplace.json` — new marketplace manifest for this repository.
- `plugins/agent-teams/.claude-plugin/plugin.json` — version `1.0.0-chipagents`; new description; author `wshobson (ChipAgents port: Josh_HUANG)` (upstream: Seth Hobson); `homepage` added; license MIT kept.
- `plugins/agent-teams/LICENSE` — upstream repository's MIT license, unchanged.

## Agents

- `agents/team-lead.md` — Tools: Read, Grep, Search, List, Bash, Git, Edit, Create (upstream: Agent and team/task tools). Removed display-mode configuration, the messaging "Communication Protocols" and the spawn/shutdown/TeamDelete lifecycle; added "File Protocol" and a run lifecycle (set up, plan, dispatch by wave, collect, synthesize, close). Ownership examples now modules, packages, filelists, `define headers. Notes how to work when dispatched as a subagent (plan and synthesize, or run chip-swarm).
- `agents/team-reviewer.md` — Tools: Read, Grep, Search, List, Bash, Git (read-only). The five web dimensions (security, performance, architecture, testing, accessibility) replaced by nine hardware dimensions. Added "Run Protocol" (read brief, write only its findings/questions file via Bash, return the same report). Output format location/dimension values updated.
- `agents/team-debugger.md` — Tools: Read, Grep, Search, List, Bash, Git (read-only). Steps adapted to logs, waveforms, seeds, filelists and defines; evidence must cite file:line plus signal@time and the seed; re-runs only into a scratch/build directory. "Communicate via message" replaced by findings/questions files.
- `agents/team-implementer.md` — Tools: Read, Grep, Search, List, Bash, Git, Edit, Create. Owned files are modules/packages/testbench components; shared files (filelists, top-level, packages, register descriptions) are lead-owned; contracts in `contracts/` are immutable. Messaging and TaskUpdate replaced by findings/questions files; verification uses lint, elaboration and unit simulation.

## Commands

- `commands/team-review.md` — Rewritten: run directory setup, one `team-reviewer` per dimension, collection via findings files; consolidation phase kept as upstream. Default reviewers `functional,cdc,lint`. Added "Running the team as parallel processes" (`chip-swarm investigate`).
- `commands/team-debug.md` — Rewritten: hardware triage (first error, seed, time), eight hardware hypothesis categories (upstream: six software ones), hypotheses recorded in `brief.md`, one `team-debugger` per hypothesis; arbitration kept as upstream. Added `--seed`. Added "Running the team as parallel processes" (`chip-swarm investigate --worktrees`).
- `commands/team-feature.md` — Rewritten: contracts as SV package/interface/port lists, dependencies as waves, dispatch and collection per wave, integration verification with lint/elaboration/simulation. `--plan-first` approval via plan mode or explicit confirmation. No separate team-lead teammate (the main agent leads). Added "Running the team as parallel processes" (`chip-swarm fanout`, then merges in dependency order).
- `commands/team-status.md` — Replaced: summarizes `tasks.md`, `findings/`, open questions and report state of the latest (or named) run. Flags `--tasks`, `--findings`, `--json` (upstream `--members` dropped).
- `commands/team-shutdown.md` — Replaced: confirms unfinished tasks, removes the run's chip-swarm worktrees/branches (`chip-swarm clean <id>`), archives the run to `.chipagents/team/archive/`. Flags `--force`, `--keep-worktrees` (upstream `--keep-tasks` dropped).
- `commands/team-delegate.md` — Deleted (no task list or messaging to manage; rebalancing guidance lives in task-coordination-strategies).
- `commands/team-spawn.md` — Deleted; its presets are folded into team-composition-patterns.

## Skills

Frontmatter `name` and `version` kept; descriptions edited where they named removed features or web topics.

- `skills/multi-reviewer-patterns/SKILL.md` — Hardware dimension table, recommended combinations and severity calibration; dedup rules and report template kept.
- `skills/multi-reviewer-patterns/references/review-dimensions.md` — Checklists rewritten for the nine hardware dimensions.
- `skills/parallel-debugging/SKILL.md` — Eight hardware failure categories; evidence examples and citation format with signal@time and seed; fix validation across seeds plus a recurrence check. Arbitration protocol kept.
- `skills/parallel-debugging/references/hypothesis-testing.md` — Templates gain a reproduction section and signal@time evidence; common patterns by failure type rewritten for simulation/regression failures; decision tree kept.
- `skills/parallel-feature-development/SKILL.md` — Ownership by directory/module/layer for RTL/DV/formal; SystemVerilog package and port-list contract replaces the TypeScript interface; multi-branch strategy maps to chip-swarm worktrees; troubleshooting adapted (filelists, bind files, contract change log instead of broadcast).
- `skills/parallel-feature-development/references/file-ownership.md` — Project types replaced by RTL IP block, UVM environment, RTL + verification, generated register block.
- `skills/parallel-feature-development/references/merge-strategies.md` — Sub-branch integration via chip-swarm fanout; feature flags become parameter/define gating; integration checklist uses lint, elaboration, simulation and `.chipagents/verify.sh`.
- `skills/task-coordination-strategies/SKILL.md` — blockedBy examples replaced by wave numbers in `tasks.md`; RTL/DV decomposition and task template; rebalancing through `tasks.md` and re-dispatch.
- `skills/task-coordination-strategies/references/dependency-graphs.md` — Each pattern expressed as waves.
- `skills/task-coordination-strategies/references/task-decomposition.md` — Examples replaced (4-channel DMA; AXI4-Lite register interface).
- `skills/team-communication-protocols/SKILL.md` — Rewritten as the file-based team protocol (run directory, who writes what, reporting rule, dispatch prompt, contract changes, approval gate, anti-patterns, cleanup, troubleshooting). Skill name kept.
- `skills/team-communication-protocols/references/messaging-patterns.md` — Replaced by `references/file-templates.md` (same message types as file templates; shutdown acknowledgment dropped, change-log entry added).
- `skills/team-composition-patterns/SKILL.md` — Presets mapped to hardware (fullstack → RTL-DV); `subagent_type` table replaced by agent profiles; display modes replaced by execution modes (subagents, chip-swarm investigate/fanout/race).
- `skills/team-composition-patterns/references/agent-type-selection.md` — Profile decision matrix and comparison (Explore/Plan/general-purpose replaced by the default subagent).
- `skills/team-composition-patterns/references/preset-teams.md` — Presets rewritten for hardware (review, debug, feature, RTL-DV, research, security, migration) including the presets from the deleted team-spawn command.
