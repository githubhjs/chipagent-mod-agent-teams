---
name: parallel-debugging
description: Debug complex hardware simulation, regression and bring-up failures using competing hypotheses with parallel investigation, evidence collection, and root cause arbitration. Use this skill when debugging bugs with multiple potential causes, performing root cause analysis, or organizing parallel investigation workflows.
version: 1.0.2
---

# Parallel Debugging

Framework for debugging complex issues using the Analysis of Competing Hypotheses (ACH) methodology with parallel agent investigation.

## When to Use This Skill

- Failure has multiple plausible root causes (RTL, testbench, environment)
- Initial debugging attempts haven't identified the issue
- Issue spans multiple modules, clock domains or the RTL/testbench boundary
- Need systematic root cause analysis with evidence
- Want to avoid confirmation bias in debugging

## Hypothesis Generation Framework

Generate hypotheses across 8 failure mode categories:

### 1. RTL Logic

- Incorrect condition or operator, missing `case` item
- Wrong FSM transition or priority between requests
- Off-by-one in counters, pointers or length calculations
- Width truncation, sign extension error

### 2. Testbench/Stimulus

- Illegal or unintended stimulus (constraint too loose or too tight)
- Wrong sequence order or timing
- Reference model or scoreboard bug (the DUT is right)
- Drive/sample race (missing clocking block, sampling on the wrong edge)

### 3. Reset/Init/X

- Flop or memory without reset or initialization
- X-optimism in `if`/`case` masking or creating a failure
- Reset de-assertion timing relative to clocks or stimulus
- Power-on or default configuration differs from expectation

### 4. CDC/Ordering

- Missing or wrong synchronizer; multi-bit crossing without a safe scheme
- Reconvergence of separately synchronized signals
- Event ordering between clock domains changes with clock ratio or seed
- Simulation race (blocking assignment ordering, `#0`)

### 5. Handshake/Protocol

- VALID depends on READY, or payload changes while waiting
- Backpressure ignored (data dropped or duplicated)
- AXI ID ordering, burst length/xLAST, or response handling wrong
- Deadlock between channels or blocks

### 6. Register Programming/Configuration

- Wrong register sequence, value or write order
- Register model (RAL) mirror differs from hardware
- Address map or field offset mismatch
- Mode or parameter combination not supported

### 7. Tool & Environment

- Filelist picks up the wrong file version or misses a file
- Different `define/ifdef settings between runs
- Simulator version, compile options, or library mapping
- Seed, plusargs or test configuration not what the report says

### 8. Gate-Level/SDF/Timing

- SDF annotation missing or mismatched
- Timing-check violations producing X (setup/hold notifiers)
- Zero-delay races in the netlist simulation
- Synthesis/simulation mismatch (`full_case`/`parallel_case`, `initial`, incomplete sensitivity)

## Evidence Collection Standards

### What Constitutes Evidence

| Evidence Type     | Strength | Example                                                                                       |
| ----------------- | -------- | --------------------------------------------------------------------------------------------- |
| **Direct**        | Strong   | `dma_ctrl.sv:142` leaves WAIT_RESP on `bvalid` alone; waveform shows `bid=1` while channel 0 completes @ 12345 ns (seed 1234) |
| **Correlational** | Medium   | Fails only on seeds run after commit `abc123`, or only with `+define+DMA_NUM_CH=4`             |
| **Testimonial**   | Weak     | "It passes with my seed"                                                                      |
| **Absence**       | Variable | No synchronizer found between `req_clka` and its use in the `clkb` domain                     |

### Citation Format

Always cite evidence with file:line plus the signal value and time from the waveform or log, and the seed:

```
**Evidence**: The response FSM at `rtl/dma/dma_ctrl.sv:142` leaves WAIT_RESP
when `bvalid` is high without comparing `bid`. In seed 1234,
`tb.u_dut.u_ctrl.state_q` goes WAIT_RESP -> IDLE @ 12345 ns while
`bid = 1` (sim.log:880), completing channel 0 with channel 1's response.
```

### Confidence Levels

| Level               | Criteria                                                                            |
| ------------------- | ----------------------------------------------------------------------------------- |
| **High (>80%)**     | Multiple direct evidence pieces, clear causal chain, no contradicting evidence      |
| **Medium (50-80%)** | Some direct evidence, plausible causal chain, minor ambiguities                     |
| **Low (<50%)**      | Mostly correlational evidence, incomplete causal chain, some contradicting evidence |

## Result Arbitration Protocol

After all investigators report:

### Step 1: Categorize Results

- **Confirmed**: High confidence, strong evidence, clear causal chain
- **Plausible**: Medium confidence, some evidence, reasonable causal chain
- **Falsified**: Evidence contradicts the hypothesis
- **Inconclusive**: Insufficient evidence to confirm or falsify

### Step 2: Compare Confirmed Hypotheses

If multiple hypotheses are confirmed, rank by:

1. Confidence level
2. Number of supporting evidence pieces
3. Strength of causal chain
4. Absence of contradicting evidence

### Step 3: Determine Root Cause

- If one hypothesis clearly dominates: declare as root cause
- If multiple hypotheses are equally likely: may be compound issue (multiple contributing causes)
- If no hypotheses confirmed: generate new hypotheses based on evidence gathered

### Step 4: Validate Fix

Before declaring the bug fixed:

- [ ] Fix addresses the identified root cause
- [ ] Fix doesn't introduce new issues (lint and CDC clean)
- [ ] Original failing test and seed now pass
- [ ] The test passes across a batch of new seeds and related configurations
- [ ] An assertion, check or coverage point is added that would catch a recurrence
