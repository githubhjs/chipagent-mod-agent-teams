---
description: "Launch a multi-reviewer parallel RTL/DV review with specialized hardware review dimensions"
argument-hint: "<target> [--reviewers functional,cdc,lint,protocol,timing,verification,security,dft,scripts] [--base-branch main]"
---

# Team Review

Orchestrate a multi-reviewer parallel review where each `team-reviewer` subagent focuses on one hardware quality dimension. You (the main agent) are the lead; reviewers report through files in the run directory. Produces a consolidated, deduplicated report organized by severity.

## Pre-flight Checks

1. Parse `$ARGUMENTS`:
   - `<target>`: file path, directory (e.g. `rtl/dma/`), git diff range (e.g. `main...HEAD`), or PR number (e.g. `#123`)
   - `--reviewers`: comma-separated dimensions (default: `functional,cdc,lint`)
   - `--base-branch`: base branch for diff comparison (default: `main`)
2. Valid dimensions: `functional`, `cdc`, `lint`, `protocol`, `timing`, `verification`, `security`, `dft`, `scripts`. The multi-reviewer-patterns skill has the checklists and recommended combinations.

## Phase 1: Target Resolution

1. Determine target type:
   - **File/Directory**: Use as-is for review scope
   - **Git diff range**: Use Bash to run `git diff {range} --name-only` to get changed files
   - **PR number**: Use Bash to run `gh pr diff {number} --name-only` to get changed files
2. Collect the context reviewers need: filelist (`*.f`), top module, clocks and resets, spec and register description (RDL, IP-XACT, hjson), constraints, testbench location, lint command
3. Display review scope to user: "{N} files to review across {M} dimensions"

## Phase 2: Run Setup

1. Create the run directory `.chipagents/team/review-{YYYYMMDD-HHMMSS}/` with `findings/` and `questions/`
2. Write `brief.md`: target, file list, the diff command reviewers should run (do not paste the whole diff), context from Phase 1, out-of-scope items
3. Write `tasks.md` with one row per dimension:
   - ID `R1`..`Rn`, owner `{dimension}-reviewer` (e.g. `cdc-reviewer`)
   - Owned: `read-only: {target}; dimension {dimension}`
   - Wave `1`, status `pending`

## Phase 3: Dispatch

1. For each row, dispatch a `team-reviewer` subagent with a self-contained prompt:
   - Its name, task ID and the run directory
   - Its dimension and the matching checklist from the multi-reviewer-patterns skill
   - "Read brief.md. Review only your dimension. Write findings/{name}.md and return the same report."
2. Set each dispatched row to `in_progress`. All rows are wave 1: dispatch them together when ChipAgents runs subagents in parallel, otherwise one after another

## Phase 4: Collect

1. As each reviewer returns, check `findings/{name}.md`; if it is missing, save the returned report there
2. Set the row to `done` (or `failed` with the reason). If `questions/{name}.md` exists, answer it, log the answer in `brief.md`, and re-dispatch that reviewer
3. Track progress: "{completed}/{total} reviews complete"

## Phase 5: Consolidation

1. **Deduplicate**: Merge findings that reference the same file:line location
2. **Resolve conflicts**: If reviewers disagree on severity, use the higher rating
3. **Organize by severity**: Group findings as Critical, High, Medium, Low
4. **Cross-reference**: Note findings that appear in multiple dimensions

## Phase 6: Report

1. Present the consolidated report and write it to `report.md` in the run directory:

   ```
   ## RTL/DV Review Report: {target}

   Reviewed by: {dimensions}
   Files reviewed: {count}
   Run: .chipagents/team/{run-id}/

   ### Critical ({count})
   [findings...]

   ### High ({count})
   [findings...]

   ### Medium ({count})
   [findings...]

   ### Low ({count})
   [findings...]

   ### Summary
   Total findings: {count} (Critical: N, High: N, Medium: N, Low: N)
   ```

2. Leave the run directory in place for `/team-status`; `/team-shutdown` archives it

## Running the team as parallel processes

To run the reviewers as separate ChipAgents processes, do Phases 1-2, then hand the dimensions to `chip-swarm investigate`, phrased as claims that each reviewer tries to refute:

```bash
RUN=.chipagents/team/review-20261020-143005
cat > "$RUN/dimensions.txt" <<'EOF'
rtl/dma/ has no functional/spec defects (FSM, reset values, register map); list every defect as evidence against
rtl/dma/ has no CDC/RDC or reset defects; list every defect as evidence against
rtl/dma/ has no lint or synthesizability defects (latches, incomplete case, blocking vs non-blocking, width/sign, X-propagation); list every defect as evidence against
EOF
{ cat "$RUN/brief.md"; echo; echo "Report each defect in the team-reviewer finding format (file:line, dimension, severity, evidence, impact, fix). When merging: deduplicate by file:line, keep the higher severity, group by severity."; } > "$RUN/swarm-task.md"
chip-swarm investigate -j 3 "$RUN/dimensions.txt" "$RUN/swarm-task.md"
```

chip-swarm prints its run id (e.g. `investigate-20261020-143210`) and writes under `$SWARM_DIR/<swarm-run-id>/` (default `../.chip-swarm/<repo>/`). Log the id in `brief.md`, copy each `team-k/report.md` to `findings/{dimension}-reviewer.md`, then run Phases 5-6 (the merged `VERDICT.md` is a good starting point for `report.md`).
