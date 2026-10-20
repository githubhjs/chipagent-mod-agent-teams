# Integration and Merge Strategies

Patterns for integrating parallel work streams and resolving conflicts.

## Integration Patterns

### Pattern 1: Direct Integration

All implementers work in the same working tree on one branch (ChipAgents subagents); integration happens naturally.

```
feature/dma ← implementer-1 edits
            ← implementer-2 edits
            ← implementer-3 edits
```

**When to use**: Small teams (2-3), strict file ownership (no conflicts expected), short simulations.

### Pattern 2: Sub-Branch Integration

Each implementer works in its own worktree and branch (`chip-swarm fanout`); the lead merges them in wave and dependency order.

```
feature/dma
  ├── swarm/<swarm-run-id>/task-1  ← implementer-1
  ├── swarm/<swarm-run-id>/task-2  ← implementer-2
  └── swarm/<swarm-run-id>/task-3  ← implementer-3
```

Merge order: follow the waves (foundation → dependent → integration), e.g. `git merge --no-ff swarm/<swarm-run-id>/task-1`.

**When to use**: Larger teams (4+), long builds or simulations that should not share an output directory, need for review gates.

### Pattern 3: Trunk with Parameter or Define Gating

All implementers commit to the main branch; new logic stays disabled behind a parameter or `define until it is verified.

```
main ← all implementers commit
     ← parameter bit EnableSg = 1'b0 (or `ifdef DMA_SG) gates new logic
```

**When to use**: Continuous integration with a shared regression, short-lived features.

## Integration Verification Checklist

After all implementers complete:

1. **Lint check**: Does the whole design pass lint (and CDC, if clocks or resets changed)?
2. **Elaboration check**: Do all modules compile and elaborate with the real filelist and defines (ports, widths, package imports)?
3. **Unit checks**: Do each stream's own tests and assertions still pass?
4. **Integration tests**: Do smoke and feature tests pass at the top level, across several seeds?
5. **Interface verification**: Do all implementations match their contracts (ports, types, protocol)?
6. **Project gate**: Does `.chipagents/verify.sh` pass, if the project has one?

## Conflict Resolution

### Prevention (Best)

- Strict file ownership eliminates most conflicts
- Interface contracts define boundaries before implementation
- Shared files (filelists, packages, top-level) are owned by the lead and modified between waves

### Detection

- Git merge will report conflicts if they occur
- Lint and elaboration errors indicate port or type mismatches
- Test failures and assertion failures indicate behavioral conflicts

### Resolution Strategies

1. **Contract wins**: If code doesn't match the interface contract, the code is wrong
2. **Lead arbitrates**: The lead decides which implementation to keep
3. **Tests decide**: The implementation that passes tests and assertions is correct
4. **Merge manually**: For complex conflicts, the lead merges by hand
