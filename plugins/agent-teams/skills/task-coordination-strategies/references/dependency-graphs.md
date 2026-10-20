# Dependency Graph Patterns

Visual patterns for task dependency design with trade-offs. Dependencies are recorded as waves in `tasks.md`: a task runs in the wave after the last task it needs.

## Pattern 1: Fully Independent (Maximum Parallelism)

```
Task A ─┐
Task B ─┼─→ Final Integration
Task C ─┘
```

- **Parallelism**: Maximum — all tasks run simultaneously
- **Risk**: Integration may reveal incompatibilities late
- **Use when**: Tasks operate on completely separate modules/files
- **Waves**: A, B, C in wave 1; integration in wave 2

## Pattern 2: Sequential Chain (No Parallelism)

```
Task A → Task B → Task C → Task D
```

- **Parallelism**: None — each task waits for the previous
- **Risk**: Bottleneck at each step; one delay cascades
- **Use when**: Each task depends on the output of the previous (avoid if possible)
- **Waves**: A in 1, B in 2, C in 3, D in 4

## Pattern 3: Diamond (Shared Foundation)

```
           ┌→ Task B ─┐
Task A ──→ ┤          ├→ Task D
           └→ Task C ─┘
```

- **Parallelism**: B and C run in parallel after A completes
- **Risk**: A is a bottleneck; D must wait for both B and C
- **Use when**: B and C both need output from A (e.g. a shared package or interface)
- **Waves**: A in 1; B and C in 2; D in 3. If A is only a package, the lead writes it as a contract in wave 0 and B and C start in wave 1

## Pattern 4: Fork-Join (Phased Parallelism)

```
Phase 1:  A1, A2, A3  (parallel)
          ────────────
Phase 2:  B1, B2      (parallel, after phase 1)
          ────────────
Phase 3:  C1          (after phase 2)
```

- **Parallelism**: Within each phase, tasks are parallel
- **Risk**: Phase boundaries add synchronization delays
- **Use when**: Natural phases with dependencies (RTL → testbench → regression)
- **Waves**: Phase N is wave N; this is the native shape of the wave model

## Pattern 5: Pipeline (Streaming)

```
Task A ──→ Task B ──→ Task C
  └──→ Task D ──→ Task E
```

- **Parallelism**: Two parallel chains
- **Risk**: Chains may diverge in approach
- **Use when**: Two independent feature branches from a common starting point
- **Waves**: A in 1; B and D in 2; C and E in 3. With waves, C also waits for D; if that costs too much, run each chain as its own sequence of chip-swarm fanout runs

## Anti-Patterns

### Circular Dependency (Deadlock)

```
Task A → Task B → Task C → Task A  ✗ DEADLOCK
```

**Fix**: Extract the shared dependency into a contract or a separate task that all three depend on.

### Unnecessary Dependencies

```
Task A → Task B → Task C
(where B doesn't actually need A's output)
```

**Fix**: Put B in the same wave as A; let it run independently against the contract.

### Star Pattern (Single Bottleneck)

```
     ┌→ B
A →  ├→ C  → F
     ├→ D
     └→ E
```

**Fix**: If A is slow, all downstream tasks are delayed. Try to parallelize A's work, or replace A's output with a frozen contract so B-E can start in wave 1.
