---
name: team-communication-protocols
description: File-based team protocol for ChipAgents agent teams, covering the run directory layout (brief, tasks, contracts, findings, questions), who writes which file, dispatch prompts, blocker handling, contract changes and run cleanup. Use this skill when setting up the run directory for a review, debug or feature team, when writing a teammate's dispatch prompt, when a teammate is blocked or an interface contract must change, when collecting teammate reports, or when debugging why teammates are not coordinating correctly at integration points.
version: 1.0.2
---

# Team Communication Protocols (File-Based Team Protocol)

ChipAgents has no team messaging and no shared task list. Teammates are subagents (or chip-swarm processes) that start with no conversation history, run to completion and return one final answer. They coordinate through files in a run directory that the lead owns.

## When to Use This Skill

- Setting up the run directory for a new team
- Writing a teammate's dispatch prompt
- Collecting teammate reports and updating task status
- Handling blockers and contract changes
- Closing and archiving a finished run

## Run Directory

```
.chipagents/team/<run-id>/
├── brief.md              target, scope, context (lead writes; teammates read)
├── tasks.md              one row per task (lead is the only writer)
├── contracts/            frozen interfaces: SV packages, interfaces, port lists
├── findings/<agent>.md   each teammate's report (written by that teammate)
├── questions/<agent>.md  blockers (teammate writes; lead appends answers)
└── report.md             the lead's consolidated report
```

- `<run-id>` is `<kind>-<YYYYMMDD-HHMMSS>`, e.g. `review-20261020-143005`, `debug-…`, `feature-…`
- `<agent>` is the teammate's name: `cdc-reviewer`, `investigator-2`, `implementer-1`. Use the same name in `tasks.md`, the dispatch prompt and the file names
- Keep `.chipagents/team/` out of commits (add it to `.gitignore`). chip-swarm copies `.chipagents/` into its worktrees only when no file under it is tracked; otherwise worktree teammates do not see the run directory, so put what they need into their task text

### brief.md

```markdown
# Brief: feature-20261020-143005
Kind: feature
Target: 4-channel DMA with APB registers and AXI4 master
Scope: rtl/dma/, dv/dma/ — Out of scope: SoC integration, firmware
Context:
- Filelist: sim/dma.f   Top: dma_top   Defines: +define+DMA_NUM_CH=4
- Clocks/resets: clk (400 MHz, APB and AXI both on clk); rst_n (async assert, sync de-assert)
- Spec / registers: doc/dma_spec.md, rdl/dma.rdl
- Lint: verilator --lint-only -f sim/dma.f   Sim: make -C sim TEST=dma_smoke SEED=1
Acceptance: lint clean, dma_smoke and dma_backpressure pass on 20 seeds
## Change log
- (lead appends answers, contract changes, re-dispatches, chip-swarm run ids)
```

### tasks.md

```markdown
| ID | Owner         | Owned files / modules                        | Wave | Status      |
|----|---------------|----------------------------------------------|------|-------------|
| T1 | implementer-1 | rtl/dma/dma_regs.sv                          | 1    | done        |
| T2 | implementer-2 | rtl/dma/dma_core.sv, rtl/dma/dma_axi_mst.sv  | 1    | in_progress |
| T3 | implementer-3 | dv/dma/                                      | 2    | pending     |
| -  | lead          | rtl/dma/dma_pkg.sv, rtl/dma/dma_top.sv, sim/dma.f | 0 | done   |
```

- Status: `pending`, `in_progress`, `done`, `blocked`, `failed`
- Review and debug runs put the read-only scope plus the dimension or hypothesis in the "Owned" column (`read-only: rtl/dma/; dimension cdc`)
- Waves replace task dependencies: wave N starts only after every wave N-1 row is `done`. Wave 0 is the lead's own setup work

### contracts/

Frozen interface definitions the lead writes before wave 1: an SV package (parameters, typedefs, enums, structs), an SV interface with modports, or a port list (name, direction, width, clock domain, reset value, protocol). Teammates read them and never edit them. When the design must compile against a contract, the lead also creates the real file in the repo (e.g. `rtl/dma/dma_pkg.sv`) and lists it in `tasks.md` with owner `lead`.

### findings/<agent>.md

The teammate's report in the format of its profile (review findings, investigation report, or completion report), starting with:

```markdown
# implementer-2 — T2
Status: done | blocked | failed
Summary: DMA core and AXI master implemented; lint clean; unit test passes (seed 1-5)
```

### questions/<agent>.md

A blocker the teammate cannot resolve inside its scope: missing spec detail, contract mismatch, a change needed in a file it does not own. The teammate writes the question, sets `Status: blocked` in its findings file and returns; it does not guess. The lead appends `Answer:` under the question, logs it in the change log of `brief.md`, and re-dispatches the task.

## Who Writes What

| File                                        | Lead                                    | Reviewer / debugger | Implementer      |
| ------------------------------------------- | --------------------------------------- | ------------------- | ---------------- |
| brief.md, tasks.md, contracts/, report.md   | writes                                  | reads               | reads            |
| findings/<own>.md, questions/<own>.md       | reads; saves a returned report if missing | writes            | writes           |
| Source, testbench, filelists                | shared files only                       | never               | owned files only |

Reviewers and debuggers have no edit tools; they write their own findings and questions files with Bash (`cat > file <<'EOF'`) and nothing else.

## Reporting Rule

A teammate reports twice: it writes `findings/<agent>.md` and returns the same report as its final answer. If the file is missing when the teammate returns (for example a read-only chip-swarm run that cannot write files), the lead saves the returned report there. Then the lead updates the row in `tasks.md`.

## Dispatch Prompt

Teammates start fresh, so every dispatch prompt is self-contained:

```
You are {agent}, task {id} of run .chipagents/team/{run-id}/.
Read brief.md, contracts/ and your row in tasks.md.
Your assignment: {dimension | hypothesis with evidence criteria | owned files, requirements, acceptance}.
Write your report to findings/{agent}.md and return the same report as your final answer.
If you are blocked, write questions/{agent}.md, set "Status: blocked" in your findings file, and return.
```

## Contract Changes

Contracts are frozen for the run. If one is wrong:

1. The teammate that found it writes `questions/<agent>.md` and returns blocked
2. The lead holds the next wave, edits the contract and its repo copy, and logs the change in `brief.md`
3. The lead sets every task that uses the contract back to `pending` and re-dispatches it with a note naming the change

The change log in `brief.md` is the single record of what changed; no other notification exists.

## Approval Gate

Where the user must approve (a decomposition before implementation, archiving a run with unfinished tasks, deleting unmerged chip-swarm branches), present the plan in ChipAgents plan mode or ask for an explicit yes and wait. Teammates never ask the user; they write `questions/`.

## Anti-Patterns

| Anti-Pattern                                   | Problem                                         | Better Approach                          |
| ---------------------------------------------- | ----------------------------------------------- | ---------------------------------------- |
| Teammates editing `tasks.md`                   | Two writers, lost updates                       | The lead is the only writer              |
| Pasting a whole diff into every prompt         | Wastes context                                  | Put the diff command in `brief.md`       |
| Implementer guessing past a blocker            | Builds against the wrong interface              | Write `questions/`, return blocked       |
| Editing a contract without a change-log entry  | Teammates build against stale interfaces        | Log it and re-dispatch dependents        |
| Starting wave N before wave N-1 is done        | Missing inputs, rework                          | Check `tasks.md` first                   |
| Report only in the final answer                | Lost after the session; `/team-status` can't see it | Also write `findings/`               |
| Checking on teammates after every step         | Overhead, no gain                               | Check at wave boundaries                 |

## Cleanup

`/team-shutdown` removes the run's chip-swarm worktrees and branches (`chip-swarm clean <swarm-run-id>`) and moves the run directory to `.chipagents/team/archive/<run-id>/`.

## Troubleshooting

**A findings file is missing after the teammate returned.**
The teammate could not write it (read-only run, wrong path in the prompt). Save the returned report to `findings/<agent>.md` yourself and check the run path in the dispatch prompt.

**A teammate modified a file it does not own.**
Revert that file with git, restate the ownership in the prompt and re-dispatch. Implementers touch only owned files; reviewers and debuggers touch none.

**Two implementers both need a change in the same file.**
The file has two owners in practice. Make it lead-owned (filelists, top-level wrappers and packages usually are) and apply the changes yourself between waves, or split the file.

**A teammate reports that the contract does not compile.**
Fix the contract and its repo copy, log the change in `brief.md`, and re-dispatch every task that uses it.

**Two tasks wait on each other.**
They are in the same wave but depend on each other. Move one to the next wave, or have the lead write a stub of the missing side (a tie-off module or a package constant) before the wave starts.

## Related Skills

- [team-composition-patterns](../team-composition-patterns/SKILL.md) — Select agent profiles and team size before setting up the run directory
- [parallel-feature-development](../parallel-feature-development/SKILL.md) — Use the protocol to coordinate integration between parallel implementers
