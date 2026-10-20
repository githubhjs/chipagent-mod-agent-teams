---
description: "Close an agent-team run: archive its run directory and remove its chip-swarm worktrees"
argument-hint: "[run-id] [--force] [--keep-worktrees]"
---

# Team Shutdown

Close a finished agent-team run: check for unfinished tasks, make sure the report is saved, remove the run's chip-swarm worktrees and branches, and move the run directory to the archive.

## Phase 1: Pre-Shutdown

1. Parse `$ARGUMENTS` for a run id and flags:
   - If no run id, use the latest run (same discovery as `/team-status`)
   - `--force`: skip the confirmations below
   - `--keep-worktrees`: archive the run directory but leave chip-swarm worktrees and branches

2. Read `tasks.md` and `brief.md` from `.chipagents/team/{run-id}/`

3. If any row is not `done` (`pending`, `in_progress`, `blocked`, `failed`) and `--force` is not set:
   - Display warning: "Warning: {N} tasks are not done"
   - List those tasks
   - Ask the user for explicit confirmation: "Archive anyway? Unfinished tasks will not be resumed." Wait for a clear yes

4. If `report.md` is missing, offer to run the consolidation phase of the matching command (`/team-review`, `/team-debug`, `/team-feature`) first

## Phase 2: chip-swarm Cleanup

Unless `--keep-worktrees` is set:

1. Collect the chip-swarm run ids logged in the change log of `brief.md` (e.g. `fanout-20261020-150210`)
2. For each id, list branches not merged into the current branch: `git branch --no-merged HEAD --list "swarm/{id}/*"`
   - If any exist and `--force` is not set, list them and ask the user for confirmation: `chip-swarm clean` deletes them
3. Run `chip-swarm clean {id}` for each id. Never run `chip-swarm clean` without an id here: that removes every chip-swarm run in the repository

## Phase 3: Archive

1. Move the run directory:

   ```bash
   mkdir -p .chipagents/team/archive
   mv .chipagents/team/{run-id} .chipagents/team/archive/{run-id}
   ```

2. Display shutdown summary:

   ```
   Run "{run-id}" archived to .chipagents/team/archive/{run-id}/

   Tasks done: {done}/{total}
   Tasks not done: {remaining}
   Report: {archive path to report.md, or "none"}
   chip-swarm runs cleaned: {ids, or "none" / "kept"}
   ```
