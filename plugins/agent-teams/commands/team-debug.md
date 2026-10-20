---
description: "Debug simulation, regression or bring-up failures using competing hypotheses with parallel investigation by multiple agents"
argument-hint: "<failing-test-or-error-or-file> [--hypotheses N] [--scope files|module|project] [--seed S]"
---

# Team Debug

Debug complex issues using the Analysis of Competing Hypotheses (ACH) methodology. Several `team-debugger` subagents investigate different hypotheses in parallel, gathering evidence to confirm or falsify each one. You (the main agent) are the lead; investigators report through files in the run directory.

## Pre-flight Checks

1. Parse `$ARGUMENTS`:
   - `<failing-test-or-error-or-file>`: failing test name, error or assertion message, log file, or path to a file exhibiting the issue
   - `--hypotheses N`: number of hypotheses to generate (default: 3)
   - `--scope`: investigation scope — `files` (specific files), `module` (one IP or module hierarchy), `project` (entire design and testbench)
   - `--seed S`: seed of the failing run, if known

## Phase 1: Initial Triage

1. Analyze the failure:
   - If log file: find the first error (assertion, scoreboard mismatch, UVM_ERROR/FATAL, timeout), its simulation time, and the seed
   - If file path: read the file, look for obvious issues, collect error context
   - If error description: search the RTL, testbench and logs for the message and the code that emits it
2. Identify the symptom clearly: what fails, at which simulation time, with which seed and configuration, RTL or gate-level
3. Gather initial context: recent git changes, the reproduction command, filelist and defines, waveform availability, pass/fail history across seeds

## Phase 2: Hypothesis Generation

Generate N hypotheses about the root cause, covering different failure mode categories:

1. **RTL Logic** — Wrong condition, FSM transition, off-by-one counter, priority, width truncation
2. **Testbench/Stimulus** — Illegal or unintended stimulus, wrong sequence, reference model or scoreboard bug, drive/sample race
3. **Reset/Init/X** — Missing reset, uninitialized flop or memory, X-optimism masking a bug, reset release timing
4. **CDC/Ordering** — Missing or wrong synchronizer, reconvergence, event ordering between domains, simulation race
5. **Handshake/Protocol** — valid/ready violation, ignored backpressure, AXI ID ordering, burst length, deadlock
6. **Register Programming/Configuration** — Wrong register sequence or value, RAL mirror mismatch, address map, mode setting
7. **Tool & Environment** — Filelist, `define/ifdef, simulator version or options, compile order, seed, plusargs
8. **Gate-Level/SDF/Timing** — SDF annotation, timing-check X, zero-delay races, synthesis/simulation mismatch

Present hypotheses to user: "Generated {N} hypotheses. Dispatching investigators..."

## Phase 3: Run Setup and Dispatch

1. Create the run directory `.chipagents/team/debug-{YYYYMMDD-HHMMSS}/` with `findings/` and `questions/`
2. Write `brief.md`: symptom, failing test, seed, log and waveform paths, first-failure time, reproduction command, recent changes, scope, and a "Hypotheses" section with H1..Hn — each with its statement, category, confirming and falsifying evidence criteria
3. Write `tasks.md` with one row per hypothesis: ID `H{n}`, owner `investigator-{n}`, owned `read-only: {scope}; hypothesis H{n}`, wave `1`, status `pending`
4. For each row, dispatch a `team-debugger` subagent with a self-contained prompt: its name, task ID, run directory, the full hypothesis statement and evidence criteria, and "Write findings/{name}.md and return the same report." Set the row to `in_progress`

## Phase 4: Evidence Collection

1. As each investigator returns, check `findings/{name}.md`; if it is missing, save the returned report there
2. Set the row to `done` (or `blocked`/`failed`); answer any `questions/{name}.md`, log the answer in `brief.md`, and re-dispatch
3. Track: "{completed}/{total} investigations complete"

## Phase 5: Arbitration

1. Compare findings across all investigators:
   - Which hypotheses were confirmed (high confidence)?
   - Which were falsified (contradicting evidence)?
   - Which are inconclusive (insufficient evidence)?

2. Rank confirmed hypotheses by:
   - Confidence level (High > Medium > Low)
   - Strength of causal chain
   - Amount of supporting evidence
   - Absence of contradicting evidence

3. Present the root cause analysis and write it to `report.md` in the run directory:

   ```
   ## Debug Report: {failure description}

   Test: {test}  Seed: {seed}  First failure: {time}
   Run: .chipagents/team/{run-id}/

   ### Root Cause (Most Likely)
   **Hypothesis**: {description}
   **Confidence**: {High/Medium/Low}
   **Evidence**: {summary with file:line and signal@time citations}
   **Causal Chain**: {step-by-step from cause to symptom}

   ### Recommended Fix
   {specific fix with RTL/testbench changes}

   ### Other Hypotheses
   - {hypothesis 2}: {status} — {brief evidence summary}
   - {hypothesis 3}: {status} — {brief evidence summary}
   ```

4. If no hypothesis is confirmed, generate new hypotheses from the evidence gathered and run a second wave

## Phase 6: Close

Leave the run directory in place for `/team-status`; `/team-shutdown` archives it.

## Running the team as parallel processes

To run the investigators as separate ChipAgents processes, do Phases 1-2 and steps 1-2 of Phase 3, then:

```bash
RUN=.chipagents/team/debug-20261020-143005
cat > "$RUN/hypotheses.txt" <<'EOF'
H1 RTL logic: dma_ctrl leaves WAIT_RESP on BVALID without checking BID, so channel 1's response completes channel 0
H2 Handshake/protocol: the AXI slave model drops BVALID while BREADY is low, so the response is lost under backpressure
H3 Reset/init/X: ch_busy_q has no reset, and the X after reset masks the start pulse with seed 1234
EOF
chip-swarm investigate -j 3 --worktrees "$RUN/hypotheses.txt" "$RUN/brief.md"
```

Use `--worktrees` when investigators re-run simulations (each gets its own worktree for build output); leave it out for pure code and log reading. chip-swarm prints its run id and writes under `$SWARM_DIR/<swarm-run-id>/` (default `../.chip-swarm/<repo>/`): log the id in `brief.md`, copy each `team-k/report.md` to `findings/investigator-k.md`, and use `VERDICT.md` as input to Phase 5. Map its verdicts SUPPORTED/REFUTED/INCONCLUSIVE to Confirmed/Falsified/Inconclusive and its 0-100 confidence to High (>80), Medium (50-80), Low (<50).
