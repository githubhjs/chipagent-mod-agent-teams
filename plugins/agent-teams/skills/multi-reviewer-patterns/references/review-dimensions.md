# Review Dimension Checklists

Detailed checklists for each review dimension that reviewers follow during parallel RTL/DV review.

## Functional / Spec Conformance Checklist

### State Machines

- [ ] Every state is reachable and has an exit; no unintended dead-end states
- [ ] Illegal or unused encodings recover to a safe state
- [ ] Transition conditions and priorities match the spec
- [ ] Outputs per state match the spec (Moore/Mealy as intended)

### Registers and Reset

- [ ] Register addresses, widths and access types (RW, RO, W1C, RC, WO) match the register description
- [ ] Reset values of registers and control flops match the spec
- [ ] Side effects (clear-on-read, write-1-to-clear, interrupts) are implemented exactly once
- [ ] Reserved bits read as specified and ignore writes

### Datapath and Corner Cases

- [ ] Counters, pointers and FIFOs handle wrap-around, full and empty correctly (no off-by-one)
- [ ] Back-to-back, zero-length and maximum-length operations work
- [ ] Abort, error and timeout paths return to a clean state
- [ ] Parameterized widths and generate branches are correct for every supported configuration

## CDC/RDC and Resets Checklist

### Clock Domain Crossings

- [ ] Every single-bit crossing goes through a 2-flop (or deeper) synchronizer
- [ ] No combinational logic directly before a synchronizer input
- [ ] Multi-bit crossings use a handshake, gray-coded pointers, or an async FIFO
- [ ] Separately synchronized bits do not reconverge
- [ ] Pulses into a slower domain are stretched or toggle-encoded
- [ ] Clock muxes are glitch-free

### Resets

- [ ] Reset asserts asynchronously and de-asserts synchronously in each domain
- [ ] Each clock domain has its own reset synchronizer
- [ ] Flops reset by one reset do not corrupt flops in another reset domain (RDC)
- [ ] Reset ordering between domains and blocks is defined and respected

## Lint & Synthesizability Checklist

### Combinational Logic

- [ ] No latches: every `always_comb` assigns all outputs on all paths
- [ ] `case` statements have a `default` or are provably complete; `unique`/`priority` used deliberately
- [ ] No combinational loops

### Sequential Logic

- [ ] Non-blocking (`<=`) in sequential blocks, blocking (`=`) in combinational blocks
- [ ] One driver per signal; no mixed reset styles in one block

### Types and Widths

- [ ] No width mismatches or implicit truncation/extension on assignments, comparisons and ports
- [ ] No unintended signed/unsigned mixing
- [ ] Constants sized explicitly where width matters

### X-Propagation and Synthesis

- [ ] No `casex`; `casez` only where wildcard matching is intended
- [ ] No X-optimism hiding bugs in `if`/`case` on uninitialized signals
- [ ] No `initial`, `#` delays, `force` or other non-synthesizable constructs in RTL
- [ ] Lint run is clean or every waiver has a written reason

## Protocol Compliance Checklist

### valid/ready Handshakes

- [ ] VALID never waits for READY
- [ ] Payload stays stable while VALID && !READY
- [ ] No data is lost or duplicated under backpressure
- [ ] No combinational path from READY to VALID where the protocol forbids it

### AXI / APB

- [ ] AXI burst length matches xLAST; bursts do not cross a 4 KB boundary
- [ ] AXI transaction ordering per ID respected; outstanding limits enforced
- [ ] Response codes (OKAY/SLVERR/DECERR) generated and handled
- [ ] APB PSEL/PENABLE phases correct; PREADY wait states and PSLVERR handled

### Liveness

- [ ] No circular waits between channels or blocks (deadlock)
- [ ] Credits and FIFO occupancy cannot leak or overflow/underflow

## Timing / PPA Checklist

### Timing

- [ ] No long combinational chains (wide adders/comparators, deep mux trees) on critical paths
- [ ] High-fanout enables, selects and resets are replicated or registered
- [ ] Module outputs (and inputs where required) are registered at block boundaries
- [ ] Multicycle or false paths are documented for constraints

### Power and Area

- [ ] Clock gating uses ICG cells only, never plain logic on the clock
- [ ] Memories and large registers have enables; no needless toggling
- [ ] No redundant flops or duplicated logic; memory sizes match requirements

## Verification Quality Checklist

### Assertions

- [ ] SVA for handshake rules, FSM legality, one-hot encodings and FIFO bounds
- [ ] Assertions are disabled during reset and cannot pass vacuously by accident

### Coverage

- [ ] Functional coverage for each feature and spec corner case
- [ ] Meaningful crosses (mode × length × backpressure, etc.)
- [ ] Code coverage holes reviewed and explained

### Checking and Stimulus

- [ ] Scoreboards and reference models check every output transaction
- [ ] Tests can fail (a deliberately broken DUT would be caught)
- [ ] Constraints do not exclude legal scenarios; seeds are recorded and reproducible
- [ ] Reset, error and abort paths have tests

## Hardware Security Checklist

### Debug and Lifecycle

- [ ] Debug/JTAG access is locked by default and unlocked only after authentication
- [ ] Debug features are gated by lifecycle state
- [ ] Only legal lifecycle transitions are possible; no rollback

### Registers and Secrets

- [ ] Lock bits, write-once and sticky registers behave as specified and cannot be bypassed
- [ ] Bus accesses check privilege and secure attributes (e.g. AxPROT) where required
- [ ] Keys and OTP contents cannot be read through bus, debug, scan or error responses
- [ ] Secrets are cleared on reset, zeroization or lifecycle change

### Fault Resistance

- [ ] Security-critical FSMs use sparse encodings with a safe default
- [ ] Default-deny on decode errors; redundant checks on critical decisions

## DFT Checklist

### Scan

- [ ] No latches or internally generated clocks/resets without test-mode control
- [ ] Clock gates have a test enable for scan
- [ ] Asynchronous resets are controllable during scan shift and capture
- [ ] No combinational loops

### Memories and X Sources

- [ ] Memories have BIST or a bypass/wrapper for test
- [ ] X sources (black boxes, uninitialized memories, analog models) are blocked in test mode
- [ ] Key internal state is observable

## Scripts / Tools Checklist (optional)

- [ ] Scripts fail with a non-zero exit code on errors; no swallowed failures
- [ ] No hard-coded user paths, tool versions or secrets
- [ ] Shell variables quoted; file and tool lookups robust
- [ ] Generated files are reproducible from the script and its inputs
