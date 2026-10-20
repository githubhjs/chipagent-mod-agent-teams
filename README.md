# chipagent-mod-agent-teams: agent teams for ChipAgents

English | [繁體中文](README.zh-TW.md)

The **agent-teams** plugin from [wshobson/agents](https://github.com/wshobson/agents)
(★41k), ported to ChipAgents and rewritten for RTL and verification work. The
main agent leads a team of specialised subagents that work independently and
report back, then merges their results:

| Command | Team | Result |
|---|---|---|
| `/team-review` | One `team-reviewer` per review dimension (default: functional, CDC, lint) | Deduplicated findings ranked by severity |
| `/team-debug` | One `team-debugger` per competing hypothesis | An arbitrated root cause with evidence (Analysis of Competing Hypotheses) |
| `/team-feature` | `team-implementer`s, one per module or stream, in waves | Integrated code that passes lint, elaboration and simulation |
| `/team-status` | — | Tasks, findings, open questions of the latest or a named run |
| `/team-shutdown` | — | Archives a run and cleans its chip-swarm worktrees |

Six skills carry the methods: multi-reviewer patterns, parallel debugging,
parallel feature development, task coordination, the file-based team protocol
and team composition presets.

## Which one to use

Pick the mode first: picking the wrong one wastes the most time.

| You want | Use | How |
|---|---|---|
| several strategies competing, the winner kept | `chip-swarm race` | separate processes, a script decides |
| several hypotheses checked independently, then merged | `chip-swarm investigate` | read-only investigators plus one merge run |
| several independent tasks in parallel | `chip-swarm fanout` | one worktree each |
| a team inside the current session | [agent-teams](https://github.com/githubhjs/chipagent-mod-agent-teams) | subagents, no new processes |
| explore, design and review one feature | [feature-dev](https://github.com/githubhjs/chipagent-mod-feature-dev) | parallel exploration, then design |

The key difference: [chip-swarm](https://github.com/githubhjs/chipagent-mod-swarm)
starts **real separate processes**, every team has its own context, and a
script decides the result. agent-teams and feature-dev run inside the current
session and share its context.

chip-swarm is a CLI, not a plugin: it has no marketplace.json or plugin.json.
Put `bin/chip-swarm` on `PATH`. agent-teams and feature-dev are Claude-format
plugins, installed with `--type claude`.

## How the team works in ChipAgents

Upstream relies on Claude Code's experimental Agent Teams (`TeamCreate`,
`SendMessage`, a shared task list), which ChipAgents does not have. This port
replaces them with a **run directory** that every agent reads and writes:

```
.chipagents/team/<run-id>/
├── brief.md          target, scope, context, hypotheses, change log
├── tasks.md          one row per task: id, owner, owned files, wave, status
├── contracts/        frozen interfaces: SV packages, interfaces, port lists
├── findings/<agent>.md   each agent's report (replaces messages to the lead)
├── questions/<agent>.md  blockers for the lead
└── report.md         the lead's merged result
```

- The **main agent is the lead**: it writes the brief, dispatches subagents
  with self-contained prompts, and merges `findings/`.
- Task dependencies become **waves**: wave N starts when wave N-1 is done.
- Interfaces are **frozen in `contracts/`** before implementers start, because
  teammates cannot negotiate mid-run.
- For **true process-level parallelism**, each command shows how to run the
  same team with [chip-swarm](https://github.com/githubhjs/chipagent-mod-swarm):
  `chip-swarm investigate` for review and debug, `chip-swarm fanout` for
  feature streams.

## Hardware adaptation

- **Review dimensions**: functional/spec conformance (FSMs, reset values,
  register map), CDC/RDC and resets, lint and synthesizability, protocol
  compliance (AXI/APB/valid-ready, backpressure), timing/PPA, verification
  quality (SVA, coverage, checkers), hardware security (debug lock, secure
  registers, lifecycle), DFT, plus an optional scripts/tools dimension.
- **Debug hypothesis categories**: RTL logic, testbench/stimulus,
  reset/init/X, CDC/ordering, handshake/protocol, register
  programming/configuration, tool and environment (filelist, defines,
  simulator version, seeds), gate-level/SDF/timing. Evidence must cite
  `file:line` plus `signal@time` and the seed.
- **Feature work**: owned files are modules, packages and testbench components;
  contracts are SV packages, interfaces and port lists; examples use a DMA and
  an AXI4-Lite block instead of TypeScript/React.

## Install

```bash
chipagents plugin marketplace add githubhjs/chipagent-mod-agent-teams --type claude
chipagents plugin install agent-teams@chipagent-mod-agent-teams
```

Run `/reload` in a running session. The repository also installs in Claude Code
(`claude plugin marketplace add githubhjs/chipagent-mod-agent-teams`), where the
same file protocol works with Claude's subagents.

### Check that ChipAgents loaded it

Installation is confirmed on ChipAgents 0.42.3 (Josh_HUANG: `plugin list` shows
the plugin and its agents, commands and skills are in the install directory).
Whether ChipAgents **loads Claude-format agents and commands into a session**
has not been confirmed yet. Two quick checks, with the same `chipagents` binary
that runs your sessions:

```bash
# 1. Does the plugin command resolve? (batch mode runs custom commands with --command)
chipagents batch --command "team-status" --output /tmp/agent-teams-cmd.txt; echo "rc=$?"

# 2. Can the agent see the profiles and commands?
chipagents batch --prompt "List the slash commands and subagent profiles available to you. Names only." \
  --output /tmp/agent-teams-caps.txt
grep -E 'team-review|team-debug|team-feature|team-reviewer|team-debugger|team-implementer' /tmp/agent-teams-caps.txt
```

If the command is not found or the names are missing, the files are installed but
not loaded; please report it with the output.

## Usage

```
/team-review rtl/dma/ --reviewers functional,cdc,protocol
/team-debug "dma_tb: scoreboard mismatch at 18.4us" --hypotheses 3 --seed 1234
/team-feature "Add a second DMA channel" --plan-first
/team-status
/team-shutdown
```

`/team-feature --plan-first` stops after the decomposition and contracts and
waits for your approval (plan mode or an explicit confirmation) before any
implementer starts.

## Notes

- **Installed on ChipAgents 0.42.3 but not yet run there** (see the field
  report below). The plugin validates with
  `claude plugin validate --strict`; how ChipAgents dispatches subagents, and
  whether it runs several at once, depends on ChipAgents. The commands dispatch
  a wave together when that is supported and one after another otherwise.
- Reviewers and debuggers are read-only (no Edit/Create). They write their
  findings file with a shell heredoc; if your command allowlist blocks that,
  the lead saves their returned report instead.
- Upstream's `team-spawn` and `team-delegate` are removed; their presets live in
  the team-composition-patterns skill.

All changes per file are listed in [PORTING.md](PORTING.md) (upstream commit
`0874a27`).

## Field report: ChipAgents 0.42.3 (2026-10-20)

Installed and inspected by Josh_HUANG on a CentOS 8 machine:

| plugin | agents | commands | skills |
|---|---|---|---|
| agent-teams | 4 | 5 | 6 |
| feature-dev | 3 | 1 | 0 |

- Both are Claude-format marketplaces, added with `--type claude` and then
  installed; `plugin list` shows both.
- **Neither uses a hook**: no hooks.toml, no hooks.json, no agent hook, no
  JSON block, no post_turn. Every silent failure mode in the
  [hook mechanics notes](https://github.com/githubhjs/chipagents-loop-kit/blob/main/docs/chipagents-hook-mechanics.md)
  is hook-related, so these two avoid all of them by using only agents and
  commands.
- A search for the Claude-only `TeamCreate` and `SendMessage` found 5 hits.
  Each was checked: all are in documentation explaining what upstream used
  and that the shared run directory replaces it; the implementation has
  none. The port is clean.

Not verified:

- **Whether the agents and commands are actually loaded into a session.**
  Both plugins are installed and listed, and their files are in the install
  directory, but `installed.json` records only enabled, format and path, not
  agents, commands or skills. Files on disk are not proof of loading; use the
  check in [Check that ChipAgents loaded it](#check-that-chipagents-loaded-it).
- **Real runs** of the commands. Only installation and structure were checked.
- Whether ChipAgents runs several subagents at the same time. A subagent's
  turn does trigger `post_turn` hooks but does not appear in the run log, so
  it can only be observed through side effects; concurrency itself was not
  measured. If you need guaranteed parallelism, use
  [chip-swarm](https://github.com/githubhjs/chipagent-mod-swarm), whose teams
  are separate processes (its README has the measured run times).

## Related

- [chipagent-mod-swarm](https://github.com/githubhjs/chipagent-mod-swarm): run ChipAgents teams as parallel processes (race, investigate, fanout)
- [chipagent-mod-feature-dev](https://github.com/githubhjs/chipagent-mod-feature-dev): Anthropic's feature-dev workflow for ChipAgents
- [chipagents-loop-kit](https://github.com/githubhjs/chipagents-loop-kit): keep the agent working until the goal is met

## License

MIT, as upstream ([LICENSE](LICENSE)). Original work by Seth Hobson
([wshobson/agents](https://github.com/wshobson/agents)).

## Author

Chih-Hsueh "Josh" HUANG (ChipAgents port).
