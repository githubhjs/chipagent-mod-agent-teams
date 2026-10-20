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

- **Not yet run inside real ChipAgents.** The plugin validates with
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

## Related

- [chipagent-mod-swarm](https://github.com/githubhjs/chipagent-mod-swarm): run ChipAgents teams as parallel processes (race, investigate, fanout)
- [chipagent-mod-feature-dev](https://github.com/githubhjs/chipagent-mod-feature-dev): Anthropic's feature-dev workflow for ChipAgents
- [chipagents-loop-kit](https://github.com/githubhjs/chipagents-loop-kit): keep the agent working until the goal is met

## License

MIT, as upstream ([LICENSE](LICENSE)). Original work by Seth Hobson
([wshobson/agents](https://github.com/wshobson/agents)).

## Author

Chih-Hsueh "Josh" HUANG (ChipAgents port).
