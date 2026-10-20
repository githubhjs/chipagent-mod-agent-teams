---
name: parallel-feature-development
description: Coordinate parallel RTL/DV feature development with module and file ownership strategies, conflict avoidance rules, frozen interface contracts and integration patterns for multi-agent implementation. Use this skill when decomposing a large hardware feature into independent work streams, when two or more agents need to implement different modules or layers (RTL, testbench, formal) of the same design simultaneously, when establishing file ownership to prevent merge conflicts in a shared repository, when designing SV package, interface or port-list contracts so parallel implementers can build against each other's modules before they are ready, or when deciding whether to use vertical slices versus horizontal layers.
version: 1.0.2
---

# Parallel Feature Development

Strategies for decomposing hardware features into parallel work streams, establishing module ownership boundaries, avoiding conflicts, and integrating results from multiple implementer agents.

## When to Use This Skill

- Decomposing a feature for parallel implementation
- Establishing module/file ownership boundaries between agents
- Designing interface contracts (SV packages, interfaces, port lists) between parallel work streams
- Choosing integration strategies (vertical slice vs horizontal layer)
- Managing branch and merge workflows for parallel development (single tree or chip-swarm worktrees)

## File Ownership Strategies

### By Directory

Assign each implementer ownership of specific directories:

```
implementer-1: rtl/dma/
implementer-2: dv/dma/
implementer-3: fv/dma/
```

**Best for**: Repositories with clear RTL/DV/formal directory boundaries.

### By Module

Assign ownership of logical modules (which may span files):

```
implementer-1: Register block (dma_regs.sv, register-level SVA)
implementer-2: Transfer engine (dma_core.sv, dma_axi_mst.sv)
```

**Best for**: IPs with well-separated sub-blocks.

### By Layer

Assign ownership of verification and design layers:

```
implementer-1: RTL (modules, packages it owns)
implementer-2: UVM environment (agents, scoreboard, reference model)
implementer-3: Tests, sequences and coverage
```

**Best for**: A new IP with a matching testbench built at the same time.

## Conflict Avoidance Rules

### The Cardinal Rule

**One owner per file.** No file should be assigned to multiple implementers.

### When Files Must Be Shared

Filelists (`*.f`), top-level wrappers, packages, register descriptions, `define headers and bind files are typical shared files:

1. **Designate a single owner** — Usually the lead
2. **Other implementers request changes** — In their findings file (new files to add) or in `questions/<agent>.md`
3. **Owner applies changes between waves** — Prevents merge conflicts
4. **Alternative: Extract interfaces** — Move the shared definitions into a contract package that every module imports without modifying

### Interface Contracts

When implementers need to coordinate at boundaries, the lead freezes the contract in `contracts/` (and in the repo when it must compile):

```systemverilog
// contracts/dma_pkg.sv — frozen; repo copy rtl/dma/dma_pkg.sv (owned by lead)
package dma_pkg;
  parameter int unsigned NumCh = 4;
  parameter int unsigned AddrW = 32;

  typedef enum logic [1:0] {
    DIR_MEM2MEM  = 2'd0,
    DIR_MEM2PERI = 2'd1,
    DIR_PERI2MEM = 2'd2
  } dma_dir_e;

  typedef struct packed {
    logic [AddrW-1:0] src_addr;
    logic [AddrW-1:0] dst_addr;
    logic [15:0]      len_bytes;
    dma_dir_e         dir;
    logic             start;     // one-cycle pulse
  } dma_ch_cfg_t;

  typedef struct packed {
    logic busy;
    logic done;                  // one-cycle pulse
    logic err;
  } dma_ch_status_t;
endpackage
```

```markdown
<!-- contracts/ports.md — dma_regs <-> dma_core, clk domain, rst_n -->
| Signal    | Driver   | Type                        | Reset | Notes                          |
|-----------|----------|-----------------------------|-------|--------------------------------|
| ch_cfg    | dma_regs | dma_ch_cfg_t [NumCh-1:0]    | '0    | stable while ch_status.busy    |
| ch_status | dma_core | dma_ch_status_t [NumCh-1:0] | '0    | done and err are 1-cycle pulses |
```

Both implementers import `dma_pkg` and follow the port list, but neither modifies them.

## Integration Patterns

### Vertical Slice

Each implementer builds a complete feature slice (RTL + assertions + tests):

```
implementer-1: Scatter-gather mode (dma_sg.sv + its SVA + sg tests)
implementer-2: Interrupt coalescing (dma_irq.sv + its SVA + irq tests)
```

**Pros**: Each slice is independently verifiable, minimal integration needed.
**Cons**: May duplicate testbench utilities, harder with tightly coupled features.

### Horizontal Layer

Each implementer builds one layer across all features:

```
implementer-1: All RTL (register block, core, AXI master)
implementer-2: UVM environment (APB/AXI agents, scoreboard, reference model)
implementer-3: Tests, sequences and coverage
```

**Pros**: Consistent patterns within each layer, natural specialization.
**Cons**: More integration points; tests depend on both RTL and environment (later wave).

### Hybrid

Mix vertical and horizontal based on coupling:

```
implementer-1: Scatter-gather mode (vertical — RTL + SVA + tests)
implementer-2: Shared infrastructure (horizontal — AXI master, testbench agents)
```

**Best for**: Most real-world features with some shared infrastructure.

## Branch Management

### Single Branch Strategy

All implementers work in the same working tree on the feature branch (ChipAgents subagents):

- Simple setup, no merge overhead
- Requires strict file ownership to avoid conflicts
- Best for: small teams (2-3), well-defined boundaries

### Multi-Branch Strategy

Each implementer works in its own worktree and branch, as `chip-swarm fanout` sets up:

```
feature/dma
  ├── swarm/fanout-20261020-150210/task-1   (implementer-1)
  ├── swarm/fanout-20261020-150210/task-2   (implementer-2)
  └── swarm/fanout-20261020-150210/task-3   (implementer-3)
```

- More isolation (separate build and simulation output), explicit merge points
- Higher overhead; worktrees start from HEAD, so commit lead-owned shared files first
- Best for: larger teams (4+), long simulations, complex features

## Troubleshooting

**Implementers are blocking each other waiting for shared code.**
Extract the shared piece into a contract package or interface owned by the lead and have implementers import it. Neither implementer modifies the contract — they only implement against it.

**Merge conflicts appear even with clear ownership rules.**
A file was assigned to two agents, or a filelist, top-level wrapper, `define header or bind file was modified by both. Designate one owner (the lead) for those files, or have the lead apply all such edits at the end of each wave.

**An implementer finishes early but the integration step is blocked.**
Use a staging interface: the finished implementer writes a stub, tie-off or behavioral model of the missing neighbor in its own test code so the other side can be checked. Replace with the real module at integration time.

**The feature decomposition turned out wrong mid-stream.**
Stop dispatching, redistribute files in `tasks.md`, log the change in `brief.md`, and re-dispatch the affected tasks. Sunk cost on partially written code is acceptable — continuing with the wrong split is worse.

**Tests written by one implementer fail against RTL written by another.**
Interface contracts drifted: a port, width or protocol changed without going through the lead. Enforce the rule that contract changes go through the lead, the change log in `brief.md`, and a re-dispatch of every dependent task.

## Related Skills

- [team-composition-patterns](../team-composition-patterns/SKILL.md) — Choose the right team size and agent profiles before decomposing work
- [team-communication-protocols](../team-communication-protocols/SKILL.md) — The file protocol for integration handoffs, blockers and plan approval
