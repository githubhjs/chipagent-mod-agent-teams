# Preset Team Definitions

Detailed preset team configurations with task templates for common RTL/DV workflows. In every preset the main agent is the lead: it creates `.chipagents/team/<run-id>/`, writes `brief.md` and `tasks.md`, and dispatches the members below. Each task template becomes a `tasks.md` row plus a dispatch prompt.

## Review Team Preset

**Command**: `/team-review <target>`

### Configuration

- **Team Size**: 3
- **Agent Profile**: `team-reviewer`
- **Parallel processes**: `chip-swarm investigate` (one line per dimension)

### Members

| Name                | Dimension        | Focus Areas                                                |
| ------------------- | ---------------- | ---------------------------------------------------------- |
| functional-reviewer | Functional       | FSMs, reset values, register map, spec corner cases        |
| cdc-reviewer        | CDC/RDC & resets | Synchronizers, multi-bit crossings, reset sync and ordering |
| lint-reviewer       | Lint & synthesis | Latches, case coverage, blocking vs non-blocking, widths, X |

### Task Template

```
Task: Review {target} for {dimension} issues
  Dimension: {dimension}
  Target: {file list or diff command}
  Checklist: {dimension-specific checklist}
  Output: findings/{dimension}-reviewer.md — structured findings with file:line, severity, evidence, fix
```

### Variations

- **Interface-focused**: `--reviewers protocol,functional,verification` (3 members)
- **Sign-off review**: `--reviewers functional,cdc,lint,protocol,verification,security,dft` (7 members; consider chip-swarm)
- **Clock/reset change**: `--reviewers cdc,dft,lint` (3 members)

## Debug Team Preset

**Command**: `/team-debug <failing-test-or-error>`

### Configuration

- **Team Size**: 3 (default) or N with `--hypotheses N`
- **Agent Profile**: `team-debugger`
- **Parallel processes**: `chip-swarm investigate` (`--worktrees` when investigators re-run simulations)

### Members

| Name           | Role                      |
| -------------- | ------------------------- |
| investigator-1 | Investigates hypothesis 1 |
| investigator-2 | Investigates hypothesis 2 |
| investigator-3 | Investigates hypothesis 3 |

### Task Template

```
Task: Investigate hypothesis: {hypothesis summary}
  Hypothesis: {full hypothesis statement}
  Category: {failure category}
  Reproduction: {test, seed, command, log, waveform}
  Scope: {files/module/project}
  Evidence criteria:
    Confirming: {what would confirm}
    Falsifying: {what would falsify}
  Output: findings/investigator-{n}.md — confidence, evidence with file:line and signal@time, causal chain
```

## Feature Team Preset

**Command**: `/team-feature <feature-description>`

### Configuration

- **Team Size**: lead + 2 implementers
- **Agent Profiles**: main agent as lead + `team-implementer`
- **Parallel processes**: `chip-swarm fanout` per wave

### Members

| Name          | Role             | Responsibility                                          |
| ------------- | ---------------- | ------------------------------------------------------- |
| lead          | main agent       | Decomposition, contracts, shared files, integration     |
| implementer-1 | team-implementer | Work stream 1 (assigned modules)                        |
| implementer-2 | team-implementer | Work stream 2 (assigned modules)                        |

### Task Template

```
Task: Implement {work stream name} (wave {n})
  Owned files: {explicit file list}
  Requirements: {specific deliverables}
  Interface contract: {contracts/ files: package, interface, port list}
  Acceptance criteria: {lint, compile, test, assertion}
  Output: findings/implementer-{n}.md — files changed, interfaces provided, checks run
```

## RTL-DV Team Preset

**Command**: none — set up by hand with the team-feature workflow

### Configuration

- **Team Size**: lead + 3 implementers
- **Agent Profiles**: main agent as lead + 3x `team-implementer`

### Members

| Name     | Role             | Layer                                              |
| -------- | ---------------- | -------------------------------------------------- |
| lead     | main agent       | Contracts (package, interface, ports), integration |
| rtl-dev  | team-implementer | RTL modules                                        |
| env-dev  | team-implementer | UVM environment: agents, scoreboard, reference model |
| test-dev | team-implementer | Tests, sequences, SVA bind files, coverage         |

### Dependency Pattern

```
Wave 1: rtl-dev ──┐
                  ├──→ Wave 2: test-dev
Wave 1: env-dev ──┘
```

`env-dev` builds against the frozen port list and interface, so it runs in parallel with `rtl-dev`.

## Research Team Preset

**Command**: none — set up by hand

### Configuration

- **Team Size**: 3
- **Agent Profile**: ChipAgents default subagent (no profile), told not to edit files

### Members

| Name         | Role             | Focus                                                    |
| ------------ | ---------------- | -------------------------------------------------------- |
| researcher-1 | default subagent | Research area 1 (e.g. design hierarchy and data flow)    |
| researcher-2 | default subagent | Research area 2 (e.g. spec, standard or datasheet)       |
| researcher-3 | default subagent | Research area 3 (e.g. tool flow, scripts, constraints)   |

### Task Template

```
Task: Research {topic or question}
  Question: {specific research question}
  Scope: {RTL/testbench files, spec documents, tool logs, or all}
  Deliverable: Summary with citations (file:line for code, document and section for specs)
  Output: findings/researcher-{n}.md — structured report with sections, evidence, recommendations
```

### Variations

- **Design-only**: 3 researchers exploring different sub-blocks or clock domains
- **Spec extraction**: 3 researchers turning different spec chapters into requirement lists
- **Mixed**: 1 design researcher + 1 spec researcher + 1 flow researcher (recommended for evaluating a third-party IP)

### Example Research Assignments

```
Researcher 1 (design): "Trace a DMA transfer from the CTRL.START write to the AXI write response. List every module and FSM state on the way."
Researcher 2 (spec): "From doc/dma_spec.md and the AXI4 specification, list the ordering and error-response requirements that apply to the DMA's AXI master."
Researcher 3 (flow): "How are filelists, defines and seeds chosen for the dma regression? Cite the scripts and Makefile targets."
```

## Security Team Preset

**Command**: none — set up by hand (`/team-review <target> --reviewers security` gives a single security reviewer)

### Configuration

- **Team Size**: 4
- **Agent Profile**: `team-reviewer`

### Members

| Name              | Dimension               | Focus Areas                                                           |
| ----------------- | ----------------------- | --------------------------------------------------------------------- |
| debug-reviewer    | Debug & lifecycle       | JTAG/debug lock and unlock authentication, lifecycle gating and transitions |
| access-reviewer   | Access control          | Secure registers, lock bits, privilege/secure attributes, bus firewalls |
| secrets-reviewer  | Secrets & keys          | Key/OTP storage, readback via bus/debug/scan, zeroization             |
| fault-reviewer    | Fault resistance        | Sparse FSM encodings, redundant checks, default-deny, glitch handling |

### Task Template

```
Task: Security audit {target} for {sub-dimension}
  Dimension: {security sub-dimension}
  Target: {file list, directory, or IP}
  Checklist: {hardware security checklist items for this sub-dimension}
  Threat model: {attacker capabilities from brief.md: debug access, software privilege, physical}
  Output: findings/{name}.md — structured findings with file:line, severity, evidence, remediation
```

### Variations

- **Quick scan**: debug and secrets reviewers only (2 members)
- **Full audit**: All 4 sub-dimensions (default)
- **With DFT**: Add a 5th reviewer on the `dft` dimension for scan-based leakage

## Migration Team Preset

**Command**: none — set up by hand with the team-feature workflow

### Configuration

- **Team Size**: lead + 2 implementers + 1 reviewer
- **Agent Profiles**: main agent as lead + 2x `team-implementer` + `team-reviewer`

### Members

| Name             | Role             | Responsibility                                  |
| ---------------- | ---------------- | ----------------------------------------------- |
| lead             | main agent       | Migration plan, contracts, conflict handling    |
| migrator-1       | team-implementer | Migration stream 1 (assigned files/modules)     |
| migrator-2       | team-implementer | Migration stream 2 (assigned files/modules)     |
| migration-verify | team-reviewer    | Verify migrated code correctness and patterns   |

### Task Template

```
Task: Migrate {module/files} from {old} to {new}
  Owned files: {explicit file list}
  Migration rules: {specific transformation patterns}
  Old pattern: {what to change from}
  New pattern: {what to change to}
  Acceptance criteria: {lint clean, regression passes, equivalence or no new warnings}
  Wave: {n}
```

### Dependency Pattern

```
Wave 0: lead (plan, rules) → Wave 1: migrator-1 ──┐
                           → Wave 1: migrator-2 ──┼→ Wave 2: migration-verify
                                                  ┘
```

### Use Cases

- Language migrations (Verilog-2001 → SystemVerilog, VHDL → SystemVerilog)
- Memory macro, standard-cell library or third-party IP version swaps
- Bus protocol upgrades (AXI3 → AXI4, APB3 → APB4)
- Register flow changes (hand-written registers → RDL/IP-XACT generation)
- Simulator, UVM or build system changes (compile options, filelist format, Make → other runners)
