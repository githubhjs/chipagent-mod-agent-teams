---
description: "Develop RTL/DV features in parallel with multiple agents using module ownership boundaries, frozen interface contracts and waves"
argument-hint: "<feature-description> [--team-size N] [--branch feature/name] [--plan-first]"
---

# Team Feature

Orchestrate parallel feature development with several `team-implementer` subagents. You (the main agent) are the lead: you decompose the feature into work streams with strict module ownership, freeze the interface contracts, run the streams wave by wave, and verify integration.

## Pre-flight Checks

1. Parse `$ARGUMENTS`:
   - `<feature-description>`: description of the feature to build
   - `--team-size N`: number of implementers (default: 2)
   - `--branch`: git branch name (default: auto-generated from feature description)
   - `--plan-first`: decompose and get user approval before dispatching

## Phase 1: Analysis

1. Analyze the feature description to understand scope (spec section, register changes, interfaces, performance targets)
2. Explore the codebase to identify:
   - Modules and packages that will need modification or creation
   - Existing conventions (naming, reset style, parameterization, bus interfaces)
   - Integration points: parent modules, filelists (`*.f`), register description, `define headers
   - Testbench, sequences, assertions and tests that need updates
   - Lint, build and simulation commands (and `.chipagents/verify.sh` if present)

## Phase 2: Decomposition

1. Decompose the feature into work streams:
   - Each stream gets exclusive ownership of modules/packages/files (no overlapping files)
   - Define interface contracts between streams: SV package (parameters, typedefs, structs, enums), SV interface, or port list (name, direction, width, clock domain, reset value, protocol)
   - Turn dependencies into waves: a stream that needs another stream's output goes into a later wave
   - Keep shared files (filelists, top-level wrapper, packages, register description) lead-owned
   - Balance workload across streams

2. If `--plan-first` is set:
   - Present the decomposition to the user in plan mode (or ask for explicit confirmation):

     ```
     ## Feature Decomposition: {feature}

     ### Stream 1: {name} (wave 1)
     Owner: implementer-1
     Files: rtl/dma/dma_regs.sv
     Dependencies: none

     ### Stream 2: {name} (wave 1)
     Owner: implementer-2
     Files: rtl/dma/dma_core.sv, rtl/dma/dma_axi_mst.sv
     Dependencies: none (builds against contracts/dma_pkg.sv)

     ### Stream 3: {name} (wave 2)
     Owner: implementer-3
     Files: dv/dma/
     Dependencies: waits for wave 1 (tests need the real register block and core)

     ### Integration Contract
     {SV package / interface / port lists}

     ### Lead-owned files
     rtl/dma/dma_pkg.sv, rtl/dma/dma_top.sv, sim/dma.f
     ```

   - Wait for user approval before proceeding
   - If user requests changes, adjust decomposition

## Phase 3: Run Setup

1. If `--branch` specified, use Bash to create and checkout the branch:
   ```
   git checkout -b {branch-name}
   ```
2. Create the run directory `.chipagents/team/feature-{YYYYMMDD-HHMMSS}/` with `contracts/`, `findings/` and `questions/`
3. Write `brief.md`: feature, spec references, conventions, lint/build/sim commands, acceptance criteria
4. Write the frozen contracts to `contracts/` and create the lead-owned repo files the streams compile against (package, top-level stub, filelist entries)
5. Write `tasks.md`: one row per stream with ID, owner `implementer-{n}`, owned files/modules, wave, status `pending`

## Phase 4: Dispatch by Wave

For each wave, starting with wave 1:

1. For each task of the wave, dispatch a `team-implementer` subagent with a self-contained prompt: its name, task ID, run directory, owned files, contract files, requirements and acceptance criteria, and "Write findings/{name}.md and return the same report." Set the row to `in_progress`
2. Dispatch all tasks of the wave together when ChipAgents runs subagents in parallel, otherwise one after another — ownership makes the order irrelevant

## Phase 5: Collect and Coordinate

1. As implementers return:
   - Check `findings/{name}.md` (save the returned report if missing) and set the row `done`, `blocked` or `failed`
   - Check `git status` for changes outside the owner's files; revert and re-dispatch if found
   - Answer `questions/{name}.md`; if a contract must change, edit it, log the change in `brief.md`, and set every dependent task back to `pending`
2. Apply requested changes to lead-owned files (filelists, top-level instantiation) between waves
3. When every row of the wave is `done`, run a quick lint/compile, then start the next wave
4. Rebalance if a stream is much larger than the others

## Phase 6: Integration Verification

After the last wave:

1. Lint and compile/elaborate the whole design with the project's commands
2. Run the relevant simulations (smoke test, then the feature's tests) and `.chipagents/verify.sh` if present
3. Check that every implementation matches its contract (ports, widths, types)
4. If issues are found, add fix tasks to `tasks.md` as a new wave, assigned to the owners of the affected files, and dispatch them
5. Report integration status to user

## Phase 7: Summary

1. Present the feature summary and write it to `report.md` in the run directory:

   ```
   ## Feature Complete: {feature}

   Files modified: {count}
   Streams completed: {count}/{total}
   Lint: {clean/issues}  Tests: {pass/fail}

   Changes are on branch: {branch-name}
   ```

2. Leave the run directory in place for `/team-status`; `/team-shutdown` archives it and cleans chip-swarm worktrees

## Running the team as parallel processes

To run a wave's streams as separate ChipAgents processes in their own git worktrees, do Phases 1-3, commit the lead-owned files (worktrees start from HEAD), then write one block per wave-1 stream, separated by lines containing only `---`:

```bash
RUN=.chipagents/team/feature-20261020-143005
git add rtl/dma/dma_pkg.sv rtl/dma/dma_top.sv sim/dma.f && git commit -m "dma: freeze interface contract"
cat > "$RUN/wave1.md" <<'EOF'
You are implementer-1, task T1 of run .chipagents/team/feature-20261020-143005/.
Build the APB register block. Owned files: rtl/dma/dma_regs.sv (create). Modify no other file.
Contract (frozen): rtl/dma/dma_pkg.sv and the port list in the run directory's contracts/ports.md.
Acceptance: lint clean with the brief's lint command; register reset values match the register description.
End with your completion report (files changed, interfaces provided, checks run).
---
You are implementer-2, task T2 of run .chipagents/team/feature-20261020-143005/.
Build the DMA core and AXI master. Owned files: rtl/dma/dma_core.sv, rtl/dma/dma_axi_mst.sv (create). Modify no other file.
Contract (frozen): rtl/dma/dma_pkg.sv and contracts/ports.md.
Acceptance: lint clean; AXI handshakes follow the contract under backpressure.
End with your completion report (files changed, interfaces provided, checks run).
EOF
chip-swarm fanout -j 2 "$RUN/wave1.md"
```

Each block becomes one task, branch `swarm/<swarm-run-id>/task-k`, in block order; nothing is merged. Do not put a line containing only `---` inside a block. chip-swarm copies `.chipagents/` (and so the run directory) into each worktree only when no file under `.chipagents/` is tracked by git; otherwise paste the contract text into each block. Afterwards:

1. Log the swarm run id in `brief.md` and save each `$SWARM_DIR/<swarm-run-id>/team-k/response.txt` as `findings/implementer-k.md`; check `SUMMARY.md` for exit and verify status
2. Merge the branches in dependency order (foundation streams first, then dependents): `git merge --no-ff swarm/<swarm-run-id>/task-1`, then `task-2`, ...
3. Run the next wave from the merged branch the same way (or as subagents), then Phase 6
