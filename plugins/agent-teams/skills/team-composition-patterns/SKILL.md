---
name: team-composition-patterns
description: Design optimal agent team compositions for RTL/DV work with sizing heuristics, preset configurations, agent profile selection, and the choice between in-session subagents and chip-swarm processes. Use this skill when deciding how many agents to dispatch for a task, when choosing between a review team versus a feature team versus a debug team, when selecting the agent profile for each role so teammates have the tools they need, when choosing whether to run teammates as ChipAgents subagents or as parallel chip-swarm processes, or when building a custom team composition for a non-standard workflow such as a migration or a security audit.
version: 1.0.2
---

# Team Composition Patterns

Best practices for composing multi-agent teams in ChipAgents: selecting team sizes, choosing agent profiles, and choosing how teammates run (in-session subagents or chip-swarm processes).

## When to Use This Skill

- Deciding how many teammates to dispatch for a task
- Choosing between preset team configurations
- Selecting the right agent profile for each role
- Choosing between subagents and chip-swarm processes
- Building custom team compositions for non-standard workflows

## Team Sizing Heuristics

| Complexity   | Team Size | When to Use                                                     |
| ------------ | --------- | --------------------------------------------------------------- |
| Simple       | 1-2       | Single-dimension review, isolated bug, small feature            |
| Moderate     | 2-3       | Multi-file changes, 2-3 concerns, medium features               |
| Complex      | 3-4       | Cross-cutting concerns, large features, deep debugging          |
| Very Complex | 4-5       | Full IP with testbench, sign-off reviews, systemic failures      |

**Rule of thumb**: Start with the smallest team that covers all required dimensions. Adding teammates increases coordination overhead.

## Preset Team Compositions

`/team-review`, `/team-debug` and `/team-feature` set up the first three presets. There is no separate spawn command: for the other presets, set up the run directory as described in the team-communication-protocols skill and dispatch the profiles listed here. The main agent is the lead in every preset.

### Review Team

- **Size**: 3 reviewers
- **Agents**: 3x `team-reviewer`
- **Default dimensions**: functional, cdc, lint
- **Use when**: RTL or testbench changes need multi-dimensional quality assessment

### Debug Team

- **Size**: 3 investigators
- **Agents**: 3x `team-debugger`
- **Default hypotheses**: 3 competing hypotheses from different failure categories
- **Use when**: A failure has multiple plausible root causes

### Feature Team

- **Size**: lead + 2 implementers
- **Agents**: main agent as lead + 2x `team-implementer`
- **Use when**: Feature can be decomposed into parallel work streams

### RTL-DV Team

- **Size**: lead + 3 implementers
- **Agents**: main agent as lead + `team-implementer` (RTL) + `team-implementer` (UVM environment) + `team-implementer` (tests, SVA, coverage)
- **Use when**: A new IP or feature needs design and testbench built together

### Research Team

- **Size**: 3 researchers
- **Agents**: 3x ChipAgents default subagent (no profile), told not to edit files
- **Default areas**: Each assigned a different question: design hierarchy, spec or standard documents, tool flow
- **Use when**: Need to understand a design, compare architectures, or extract requirements from specs in parallel

### Security Team

- **Size**: 4 reviewers
- **Agents**: 4x `team-reviewer`
- **Default dimensions**: debug and lifecycle, access control and secure registers, secrets and key handling, fault resistance
- **Use when**: Security audit of an IP or SoC subsystem covering multiple attack surfaces

### Migration Team

- **Size**: lead + 2 implementers + 1 reviewer
- **Agents**: main agent as lead + 2x `team-implementer` + `team-reviewer`
- **Use when**: Large migration (Verilog to SystemVerilog, memory macro or library swap, bus protocol upgrade, simulator or UVM version change) requiring parallel work with correctness verification

## Agent Profile Selection

Dispatch teammates by agent profile, based on what tools they need:

| Profile                         | Tools                                               | Use For                                                    |
| ------------------------------- | --------------------------------------------------- | ---------------------------------------------------------- |
| `team-lead`                     | Read, Grep, Search, List, Bash, Git, Edit, Create   | Planning, run directory, synthesis (usually the main agent itself) |
| `team-reviewer`                 | Read, Grep, Search, List, Bash, Git                 | Review with structured findings                            |
| `team-debugger`                 | Read, Grep, Search, List, Bash, Git                 | Hypothesis-driven investigation                            |
| `team-implementer`              | Read, Grep, Search, List, Bash, Git, Edit, Create   | Building modules within file ownership boundaries          |
| Default subagent (no profile)   | The main agent's tools                              | One-off tasks, research                                    |

**Key distinction**: `team-reviewer` and `team-debugger` are read-only — they write only their own findings and questions files. Never assign implementation tasks to them.

## Execution Modes

| Mode                       | How teammates run                                                        | Best For                                                       |
| -------------------------- | ------------------------------------------------------------------------ | -------------------------------------------------------------- |
| In-session subagents       | The lead dispatches profiles inside one ChipAgents session               | Interactive work, small teams, quick reviews                   |
| `chip-swarm investigate`   | One read-only process per hypothesis or dimension, plus a merge run      | Reviews and debug with many dimensions or slow simulation reruns (`--worktrees`) |
| `chip-swarm fanout`        | One process per task, each in its own git worktree and branch            | Implementation waves, long builds or simulations               |
| `chip-swarm race`          | N processes solve the same task with different strategies; the verify script picks the winner | One hard task with one owner (timing fix, tricky bug fix) |

chip-swarm runs each teammate as the headless agent command set in `.chip-swarm.conf` (`AGENT_ARGV`, typically `chipagents batch …`) and needs a git repository with at least one commit.

## Custom Team Guidelines

When building custom teams:

1. **Every team needs a coordinator** — The main agent acts as lead, or the user coordinates directly
2. **Match roles to profiles** — Use specialized profiles (reviewer, debugger, implementer) when available
3. **Avoid duplicate roles** — Two agents doing the same thing wastes resources
4. **Define boundaries upfront** — Each teammate needs clear ownership of files or responsibilities in `tasks.md`
5. **Keep it small** — 2-4 teammates is the sweet spot; 5+ requires significant coordination overhead

## Troubleshooting

**A teammate was dispatched as `team-reviewer` or `team-debugger` but needs to write files.**
Those profiles are read-only. Re-dispatch the task with `team-implementer`. Never assign implementation tasks to read-only profiles.

**The team is growing too large and coordination is slowing everything down.**
Each additional teammate adds a findings file to read and reconcile. Consolidate roles: can one reviewer cover two dimensions? A 4-person team doing 6 independent tasks is usually better served by 3 agents covering 2 tasks each.

**chip-swarm does not start any teammate.**
Set `AGENT_ARGV` in `.chip-swarm.conf`, run inside the git repository, and make sure it has at least one commit. For `fanout`, commit the lead-owned files first: worktrees start from HEAD.

**Two reviewers are flagging the same issues.**
The review dimensions overlap (e.g. lint and functional both report an incomplete `case`). Redefine each reviewer's focus: lint covers structure and synthesizability, functional covers behavior against the spec. Overlapping coverage wastes tokens and produces duplicate findings.

**Teammates act without the needed context.**
Teammates start fresh with no conversation history. The dispatch prompt must name the run directory and the task, and `brief.md` must hold everything else (target, filelist, clocks, commands).

## Related Skills

- [parallel-feature-development](../parallel-feature-development/SKILL.md) — Decompose work streams and assign file ownership once the team is composed
- [team-communication-protocols](../team-communication-protocols/SKILL.md) — The run directory, report files and approval gates for the assembled team
