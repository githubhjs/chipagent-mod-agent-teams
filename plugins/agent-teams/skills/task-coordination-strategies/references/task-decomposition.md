# Task Decomposition Examples

Practical examples of decomposing hardware features into parallelizable tasks with clear ownership.

## Example 1: 4-Channel DMA Engine

### Feature Description

Add a 4-channel DMA with an APB register block, an AXI4 master and a UVM testbench.

### Decomposition (Vertical by Block, Plus Testbench)

**Wave 0: Contract** (lead)

- Owned files: `rtl/dma/dma_pkg.sv`, `rtl/dma/dma_top.sv` (stub with ports), `sim/dma.f`
- Contract: `contracts/dma_pkg.sv`, `contracts/ports.md`

**Stream 1: Register Block** (implementer-1, wave 1)

- Owned files: `rtl/dma/dma_regs.sv`
- Requirements: APB slave, per-channel config and status registers, W1C status, start pulse
- Interface: Imports `dma_ch_cfg_t`, `dma_ch_status_t` from `dma_pkg`

**Stream 2: Transfer Engine** (implementer-2, wave 1)

- Owned files: `rtl/dma/dma_core.sv`, `rtl/dma/dma_axi_mst.sv`
- Requirements: Channel arbitration, AXI read/write bursts, error handling, backpressure
- Interface: Imports `dma_ch_cfg_t`, `dma_ch_status_t` from `dma_pkg`

**Stream 3: Testbench** (implementer-3, wave 2)

- Owned files: `dv/dma/`
- Requirements: APB and AXI agents, scoreboard, smoke and backpressure tests
- Dependencies: Needs working RTL from streams 1 and 2

### Dependency Graph

```
Wave 0: lead (dma_pkg, top stub, filelist) ──→ Wave 1: Stream 1 (regs)
                                             └→ Wave 1: Stream 2 (core, AXI master)
                                                        ──→ Wave 2: Stream 3 (testbench)
```

## Example 2: Add an AXI4-Lite Register Interface to an Existing IP

### Feature Description

Replace the IP's legacy register bus with AXI4-Lite and add a register test.

### Decomposition (By Layer)

**Stream 1: Bus Adapter** (implementer-1, wave 1)

- Owned files: `rtl/{ip}/{ip}_axil_slave.sv`
- Requirements: AXI4-Lite slave converting to the IP's internal register strobe interface
- Dependencies: None (internal strobe interface frozen in `contracts/ports.md`)

**Stream 2: Register Block Update** (implementer-2, wave 1)

- Owned files: `rtl/{ip}/{ip}_regs.sv`
- Requirements: Keep register behavior, accept the internal strobe interface from the contract
- Dependencies: None (same contract)

**Stream 3: Register Tests** (implementer-3, wave 2)

- Owned files: `dv/{ip}/tests/{ip}_reg_test.sv`, `dv/{ip}/seq_lib/{ip}_axil_seq.sv`
- Requirements: Reset-value, read/write and access-type tests through AXI4-Lite
- Dependencies: Needs streams 1 and 2

## Task Template

```markdown
## Task: {Stream Name} (ID {id}, wave {n})

### Objective

{1-2 sentence description of what to build}

### Owned Files

- {file1} — {purpose}
- {file2} — {purpose}

### Requirements

1. {Specific deliverable 1}
2. {Specific deliverable 2}
3. {Specific deliverable 3}

### Interface Contract

- Provides: {ports/modports/types this stream implements}
- Uses: {package types, ports or interfaces from other streams, in contracts/}

### Acceptance Criteria

- [ ] {Lint/compile criterion}
- [ ] {Simulation or assertion criterion}
- [ ] {Spec conformance criterion}

### Out of Scope

- {Explicitly excluded work}
```
