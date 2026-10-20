---
name: task-coordination-strategies
description: Decompose complex RTL/DV tasks, order them into waves, and coordinate multi-agent work with proper task descriptions and workload balancing. Use this skill when breaking down work for agent teams, managing task dependencies, or monitoring team progress.
version: 1.0.2
---

# Task Coordination Strategies

Strategies for decomposing complex tasks into parallelizable units, ordering them into waves, writing effective task descriptions, and monitoring workload across agent teams. Tasks live in `tasks.md` of the run directory (see the team-communication-protocols skill).

## When to Use This Skill

- Breaking down a complex task for parallel execution
- Designing task dependencies as waves
- Writing task descriptions with clear acceptance criteria
- Monitoring and rebalancing workload across teammates
- Identifying the critical path in a multi-task workflow

## Task Decomposition Strategies

### By Layer

Split work by design/verification layer:

- RTL modules
- UVM environment (agents, scoreboard, reference model)
- Tests, sequences and coverage
- Formal properties and bind files

**Best for**: A new IP with its testbench, vertical slices

### By Component

Split work by functional block:

- Register block
- Datapath / transfer engine
- Bus interface (AXI master, APB slave)

**Best for**: IPs with well-separated sub-blocks

### By Concern

Split work by cross-cutting concern:

- Functional review
- CDC/RDC review
- Lint and synthesizability review

**Best for**: Design reviews, sign-off audits

### By File Ownership

Split work by file/directory boundaries:

- `rtl/dma/` — Implementer 1
- `dv/dma/` — Implementer 2
- `fv/dma/` — Implementer 3

**Best for**: Parallel implementation, conflict avoidance

## Dependency Graph Design

### Principles

1. **Minimize the number of waves** — Prefer wide, shallow graphs over deep chains
2. **Identify the critical path** — The longest chain of waves determines minimum completion time
3. **Add a wave boundary only when needed** — A task goes into a later wave only if it truly needs another task's output
4. **Avoid circular dependencies** — Task A needs B needs A is a deadlock

### Patterns

**Independent (Best parallelism)**:

```
Task A ─┐
Task B ─┼─→ Integration
Task C ─┘
```

**Sequential (Necessary dependencies)**:

```
Task A → Task B → Task C
```

**Diamond (Mixed)**:

```
        ┌→ Task B ─┐
Task A ─┤          ├→ Task D
        └→ Task C ─┘
```

### Recording Dependencies as Waves

There are no task dependency links; a task's wave number in `tasks.md` encodes them:

```
| ID | Owner         | Owned files / modules  | Wave | Status  |
|----|---------------|------------------------|------|---------|
| T1 | implementer-1 | rtl/dma/dma_regs.sv    | 1    | pending |
| T2 | implementer-2 | rtl/dma/dma_core.sv    | 1    | pending |
| T3 | implementer-3 | dv/dma/                | 2    | pending |   ← waits for T1 and T2
```

A task's wave is one more than the highest wave of the tasks it needs. Wave N starts only when every wave N-1 row is `done`.

## Task Description Best Practices

Every task should include:

1. **Objective** — What needs to be accomplished (1-2 sentences)
2. **Owned Files** — Explicit list of files/modules this teammate may modify
3. **Requirements** — Specific deliverables or behaviors expected
4. **Interface Contracts** — How this work connects to other teammates' work (`contracts/`)
5. **Acceptance Criteria** — How to verify the task is done correctly (lint, test, assertion)
6. **Scope Boundaries** — What is explicitly out of scope

### Template

```
## Objective
Build the APB register block for the 4-channel DMA.

## Owned Files
- rtl/dma/dma_regs.sv (create)
- rtl/dma/dma_pkg.sv (shared — read only, owned by lead)

## Requirements
- Per-channel SRC, DST, LEN, CTRL registers as in rdl/dma.rdl
- CTRL.START is write-1, self-clearing, and produces a one-cycle start pulse
- STATUS.DONE and STATUS.ERR are W1C and raise an interrupt when enabled

## Interface Contract
- Import dma_ch_cfg_t and dma_ch_status_t from dma_pkg (contracts/dma_pkg.sv)
- Drive ch_cfg and consume ch_status as listed in contracts/ports.md

## Acceptance Criteria
- Lint clean with the brief's lint command
- Reset values and access types match rdl/dma.rdl
- APB PSLVERR on accesses to unmapped addresses

## Out of Scope
- Transfer engine and AXI master
- Interrupt aggregation at SoC level
```

## Workload Monitoring

### Indicators of Imbalance

| Signal                                         | Meaning             | Action                             |
| ---------------------------------------------- | ------------------- | ---------------------------------- |
| Wave waits on one task while the rest are done | Uneven distribution | Split that task or rebalance next wave |
| Teammate returned blocked                      | Blocker             | Answer `questions/`, re-dispatch   |
| All remaining tasks blocked                    | Dependency issue    | Resolve critical path first        |
| One task has 3x the files of the others        | Overloaded          | Split tasks or reassign            |

### Rebalancing Steps

1. Read `tasks.md`, `findings/` and `questions/` to assess current state
2. Identify blocked, failed or oversized tasks
3. Change the owner, split the row, or move it to another wave in `tasks.md`
4. Log the change in `brief.md` and re-dispatch the affected tasks with a note
5. Check throughput at the next wave boundary
