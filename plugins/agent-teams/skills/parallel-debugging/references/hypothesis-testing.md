# Hypothesis Testing Reference

Task templates, evidence formats, and arbitration decision trees for parallel debugging. The lead puts each hypothesis task in the "Hypotheses" section of `brief.md`; each investigator writes the evidence report to `findings/<name>.md`.

## Hypothesis Task Template

```markdown
## Hypothesis Investigation: {Hypothesis Title}

### Hypothesis Statement

{Clear, falsifiable statement about the root cause}

### Failure Mode Category

{RTL Logic | Testbench/Stimulus | Reset/Init/X | CDC/Ordering | Handshake/Protocol | Register Programming/Configuration | Tool & Environment | Gate-Level/SDF/Timing}

### Reproduction

- Test: {test name}   Seed: {seed}   Configuration: {defines, plusargs}
- Command: {reproduction command}
- Log: {path}   Waveform: {path}   First failure: {time}

### Investigation Scope

- Files to examine: {RTL/testbench files or directories}
- Signals to inspect: {hierarchical names}
- Git history: {relevant date range or commits}

### Evidence Criteria

**Confirming evidence** (if I find these, hypothesis is supported):

1. {Observable condition 1, e.g. signal value at a time}
2. {Observable condition 2}

**Falsifying evidence** (if I find these, hypothesis is wrong):

1. {Observable condition 1}
2. {Observable condition 2}

### Report Format

- Confidence: High/Medium/Low
- Evidence: list with file:line and signal@time citations, plus the seed
- Causal chain: step-by-step from cause to symptom
- Recommended fix: if confirmed
```

## Evidence Report Template

```markdown
## Investigation Report: {Hypothesis Title}

### Verdict: {Confirmed | Falsified | Inconclusive}

### Confidence: {High (>80%) | Medium (50-80%) | Low (<50%)}

### Reproduction

{test, seed, command; whether the failure reproduced}

### Confirming Evidence

1. `rtl/dma/dma_ctrl.sv:142` — {what the code does}
2. `tb.u_dut.u_ctrl.state_q = IDLE @ 12345 ns` (seed 1234, sim.log:880) — {what the waveform/log shows}

### Contradicting Evidence

1. `dv/dma/dma_scoreboard.sv:211` — {what contradicts}

### Causal Chain (if confirmed)

1. {First cause} →
2. {Intermediate effect, with signal@time} →
3. {Observable symptom}

### Recommended Fix

{Specific RTL or testbench change with location}

### Additional Notes

{Anything discovered that may be relevant to other hypotheses}
```

## Arbitration Decision Tree

```
All investigators reported (tasks.md rows done or failed)?
├── NO → Wait for the remaining findings files
└── YES → Count confirmed hypotheses
          ├── 0 confirmed
          │   ├── Any medium confidence? → Investigate further
          │   └── All low/falsified? → Generate new hypotheses
          ├── 1 confirmed
          │   └── High confidence?
          │       ├── YES → Declare root cause, propose fix
          │       └── NO → Flag as likely cause, recommend verification
          └── 2+ confirmed
              └── Are they related?
                  ├── YES → Compound issue (multiple contributing causes)
                  └── NO → Rank by confidence, declare highest as primary
```

## Common Hypothesis Patterns by Failure Type

### "Scoreboard mismatch / data corruption"

1. Datapath bug: wrong byte enable, alignment or truncation (RTL Logic)
2. Reference model or expected-data bug (Testbench/Stimulus)
3. Data accepted or changed while not ready (Handshake/Protocol)

### "Simulation hangs / watchdog timeout"

1. Deadlock between channels or a missing response (Handshake/Protocol)
2. FSM stuck after reset or in an unhandled state (Reset/Init/X or RTL Logic)
3. Sequence waits for an event the configuration never produces (Register Programming/Configuration)

### "Fails only with some seeds"

1. Clock-ratio or arrival-order dependent crossing (CDC/Ordering)
2. Rare stimulus corner not handled (RTL Logic) or illegal stimulus (Testbench/Stimulus)
3. Uninitialized state picked up under certain timing (Reset/Init/X)

### "Passes RTL simulation, fails gate-level"

1. Timing violation or SDF annotation problem (Gate-Level/SDF/Timing)
2. Flop without reset that RTL X-optimism hid (Reset/Init/X)
3. Synthesis/simulation mismatch (Gate-Level/SDF/Timing)

### "Regression after a merge"

1. New RTL change introduced the bug (RTL Logic)
2. Filelist or `define change picked up a different file or mode (Tool & Environment)
3. Changed register default or programming sequence (Register Programming/Configuration)

### "X on outputs"

1. Missing reset or uninitialized memory (Reset/Init/X)
2. Unsynchronized input from another domain (CDC/Ordering)
3. Undriven or multiply driven net from a filelist or `ifdef problem (Tool & Environment)
