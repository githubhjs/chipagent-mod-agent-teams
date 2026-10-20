---
description: "Summarize tasks.md and findings of the latest (or a named) agent-team run"
argument-hint: "[run-id] [--tasks] [--findings] [--json]"
---

# Team Status

Display the current state of an agent-team run from its run directory: task progress from `tasks.md`, teammate reports from `findings/`, and open questions. This command only reads files.

## Phase 1: Run Discovery

1. Parse `$ARGUMENTS` for a run id and flags:
   - If a run id is provided, use `.chipagents/team/{run-id}/`
   - If no run id, use the latest run: the directory under `.chipagents/team/` whose id ends in the latest `YYYYMMDD-HHMMSS` timestamp (ids start with the kind, so compare the timestamps, not the names; ignore `archive/`)
   - If there is no run, say so and list the runs under `.chipagents/team/archive/`
   - `--tasks`: show only the task table
   - `--findings`: show only findings and questions
   - `--json`: output JSON instead of formatted tables

2. Read from the run directory:
   - `brief.md`: kind, target, and the change log (contract changes, chip-swarm run ids)
   - `tasks.md`: all rows
   - `findings/*.md`: the `Status` and `Summary` header lines, plus severity counts (review) or verdict and confidence (debug)
   - `questions/*.md`: questions without an `Answer:` line are open
   - `report.md`: present or not

## Phase 2: Status Display

### Tasks Table

```
Run: review-20261020-143005 (review)
Target: main...HEAD (rtl/dma/, 7 files)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Tasks:
  ID   Wave  Status        Owner                 Scope
  ─────────────────────────────────────────────────────────────
  R1   1     done          functional-reviewer   read-only: rtl/dma/; dimension functional
  R2   1     done          cdc-reviewer          read-only: rtl/dma/; dimension cdc
  R3   1     in_progress   lint-reviewer         read-only: rtl/dma/; dimension lint

Progress: 67% (2/3 done) — wave 1 of 1
```

A row that is `in_progress` with no findings file usually means the dispatch did not finish; say so.

### Findings and Questions

```
Findings:
  functional-reviewer   done      Critical 0, High 1, Medium 2, Low 0
  cdc-reviewer          done      Critical 1, High 0, Medium 0, Low 1
  lint-reviewer         (no file yet)

Open questions:
  none

Report: not written yet (consolidation pending)
chip-swarm runs: none
```

For debug runs show each investigator's verdict and confidence; for feature runs show files changed and checks run.

### JSON Output

If `--json` is set, output `{"run": ..., "kind": ..., "tasks": [...], "findings": [...], "open_questions": [...], "report": true|false}`.
