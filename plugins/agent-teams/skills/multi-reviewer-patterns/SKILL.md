---
name: multi-reviewer-patterns
description: Coordinate parallel RTL/DV reviews across multiple hardware quality dimensions with finding deduplication, severity calibration, and consolidated reporting. Use this skill when organizing multi-reviewer design reviews, calibrating finding severity, or consolidating review results.
version: 1.0.2
---

# Multi-Reviewer Patterns

Patterns for coordinating parallel reviews of RTL, testbench and flow code across multiple quality dimensions, deduplicating findings, calibrating severity, and producing consolidated reports.

## When to Use This Skill

- Organizing a multi-dimensional design or verification review
- Deciding which review dimensions to assign
- Deduplicating findings from multiple reviewers
- Calibrating severity ratings consistently
- Producing a consolidated review report

## Review Dimension Allocation

### Available Dimensions

| Dimension (key)                      | Focus                                                        | When to Include                                       |
| ------------------------------------ | ------------------------------------------------------------ | ----------------------------------------------------- |
| **Functional / spec** (`functional`) | FSMs, reset values, register map, spec corner cases          | Always for new or changed RTL                         |
| **CDC/RDC & resets** (`cdc`)         | Synchronizers, multi-bit crossings, reset sync and ordering  | Anything touching more than one clock or reset domain |
| **Lint & synthesizability** (`lint`) | Latches, case coverage, blocking vs non-blocking, width/sign, X | Always for RTL                                     |
| **Protocol** (`protocol`)            | AXI/APB/valid-ready rules, backpressure, deadlock            | Bus interfaces, bridges, streaming paths              |
| **Timing / PPA** (`timing`)          | Logic depth, fanout, clock gating, area, power               | Datapath changes, high-frequency blocks, timing ECOs  |
| **Verification** (`verification`)   | SVA, coverage, checkers, test quality                        | New features, testbench changes, sign-off             |
| **Hardware security** (`security`)   | Debug/JTAG lock, secure registers, lifecycle, secrets        | Security IP, debug, OTP/keys, access control          |
| **DFT** (`dft`)                      | Scan readiness, test-mode clock/reset control, MBIST         | Clock/reset structure changes, new memories, sign-off |
| **Scripts / tools** (`scripts`, optional) | Flow script correctness and robustness                  | Changes to Python/Tcl/Make/shell flows                |

### Recommended Combinations

| Scenario                         | Dimensions                                                       |
| -------------------------------- | ---------------------------------------------------------------- |
| New RTL block                    | Functional, Lint, CDC, Verification                              |
| Bus interface or bridge          | Protocol, Functional, Verification                               |
| Clock/reset changes              | CDC, DFT, Lint                                                   |
| Security IP (OTP, keys, debug)   | Security, Functional, DFT                                        |
| Timing ECO / PPA optimization    | Timing, Lint, Functional                                         |
| Testbench / UVM changes          | Verification, Scripts                                            |
| Full IP sign-off review          | Functional, CDC, Lint, Protocol, Verification, Security, DFT     |

## Finding Deduplication

When multiple reviewers report issues at the same location:

### Merge Rules

1. **Same file:line, same issue** — Merge into one finding, credit all reviewers
2. **Same file:line, different issues** — Keep as separate findings
3. **Same issue, different locations** — Keep separate but cross-reference (e.g. the same missing synchronizer on several signals)
4. **Conflicting severity** — Use the higher severity rating
5. **Conflicting recommendations** — Include both with reviewer attribution

### Deduplication Process

```
For each finding in all reviewer findings files:
  1. Check if another finding references the same file:line
  2. If yes, check if they describe the same issue
  3. If same issue: merge, keeping the more detailed description
  4. If different issue: keep both, tag as "co-located"
  5. Use highest severity among merged findings
```

## Severity Calibration

### Severity Criteria

| Severity     | Impact                                                        | Likelihood             | Examples                                                                        |
| ------------ | ------------------------------------------------------------- | ---------------------- | ------------------------------------------------------------------------------- |
| **Critical** | Silicon failure without workaround, security bypass, data corruption | Certain or very likely | Unsynchronized multi-bit CDC on a control path, debug unlock without authentication, bus deadlock |
| **High**     | Significant functional impact; costly workaround (firmware, ECO) | Likely               | Protocol violation under backpressure, latch in a control path, wrong register reset value |
| **Medium**   | Partial impact, workaround exists, sign-off risk              | Possible               | Deep logic path near the timing limit, coverage hole on a feature, missing SVA on a handshake |
| **Low**      | Minimal impact, cosmetic                                      | Unlikely               | Naming, comments, harmless lint warning, minor area optimization                |

### Calibration Rules

- CDC on control signals or multi-bit buses without a proper scheme: at least High
- Security issues reachable from a debug port or unprivileged software: always Critical or High
- Protocol violations that can hang or corrupt the bus: at least High
- Missing verification (assertions, coverage, tests) for a critical feature: at least Medium
- Timing risks on paths already near the target frequency: at least Medium
- Lint warnings proven harmless, style issues with no functional impact: Low

## Consolidated Report Template

```markdown
## RTL/DV Review Report

**Target**: {files/PR/directory}
**Reviewers**: {dimension-1}, {dimension-2}, {dimension-3}
**Date**: {date}
**Files Reviewed**: {count}
**Run**: .chipagents/team/{run-id}/

### Critical Findings ({count})

#### [CR-001] {Title}

**Location**: `{file}:{line}`
**Dimension**: {Functional/CDC/etc.}
**Description**: {what was found}
**Impact**: {what could happen}
**Fix**: {recommended remediation}

### High Findings ({count})

...

### Medium Findings ({count})

...

### Low Findings ({count})

...

### Summary

| Dimension    | Critical | High  | Medium | Low   | Total  |
| ------------ | -------- | ----- | ------ | ----- | ------ |
| Functional   | 0        | 2     | 3      | 0     | 5      |
| CDC/RDC      | 1        | 1     | 0      | 1     | 3      |
| Lint         | 0        | 1     | 4      | 3     | 8      |
| **Total**    | **1**    | **4** | **7**  | **4** | **16** |

### Recommendation

{Overall assessment and prioritized action items}
```
