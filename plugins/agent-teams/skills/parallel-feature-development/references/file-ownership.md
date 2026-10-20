# File Ownership Decision Framework

How to assign module and file ownership when decomposing hardware features for parallel development.

## Ownership Decision Process

### Step 1: Map All Files

List every file that needs to be created or modified for the feature: RTL modules, packages, interfaces, filelists, register descriptions, testbench components, tests, assertions/bind files, constraints, scripts.

### Step 2: Identify Natural Clusters

Group files by:

- Hierarchy (a module and the sub-modules only it instantiates)
- Functional relationship (a module and its dedicated SVA, a sequence and its test)
- Layer membership (all RTL, all UVM environment, all formal)

### Step 3: Assign Clusters to Owners

Each cluster becomes one implementer's ownership boundary:

- No file appears in multiple clusters
- Each cluster is internally cohesive
- Cross-cluster dependencies are minimized

### Step 4: Define Interface Points

Where clusters interact, define in `contracts/`:

- Shared packages: parameters, typedefs, enums, structs (owned by the lead)
- Port lists: name, direction, width, clock domain, reset value
- Protocol contracts: handshake rules, ordering, latency, backpressure behavior

## Ownership by Project Type

### RTL IP Block

```
implementer-1: rtl/{ip}/{ip}_regs.sv                    (register block)
implementer-2: rtl/{ip}/{ip}_core.sv, {ip}_fsm.sv        (datapath, control)
implementer-3: rtl/{ip}/{ip}_axi_mst.sv                  (bus interface)
shared:        rtl/{ip}/{ip}_pkg.sv, {ip}_top.sv, {ip}.f (owned by lead)
```

### UVM Environment

```
implementer-1: dv/{ip}/agents/{proto}_agent/            (driver, monitor, sequencer)
implementer-2: dv/{ip}/env/                              (scoreboard, reference model, coverage)
implementer-3: dv/{ip}/tests/, dv/{ip}/seq_lib/          (tests, sequences)
shared:        dv/{ip}/{ip}_env_pkg.sv, tb/{ip}_tb_top.sv (owned by lead)
```

### RTL + Verification (Vertical)

```
implementer-1: rtl/{ip}/                                 (design)
implementer-2: dv/{ip}/                                  (simulation testbench)
implementer-3: fv/{ip}/, rtl/{ip}/sva/                   (formal properties, bind files)
shared:        rtl/{ip}/{ip}_pkg.sv, {ip}_if.sv, filelists (owned by lead)
```

### Generated Register Block

```
implementer-1: rtl/{ip}/{ip}_core.sv                     (logic that uses the registers)
implementer-2: dv/{ip}/ral/ tests                        (register tests)
shared:        rdl/{ip}.rdl (or hjson/IP-XACT)            (owned by lead)
generated:     rtl/{ip}/{ip}_reg_top.sv, dv/{ip}/ral/{ip}_ral_pkg.sv (never hand-edited; lead regenerates)
```

## Conflict Resolution

When two implementers need to modify the same file:

1. **Preferred: Split the file** — Extract the shared concern into its own module or package
2. **If can't split: Designate one owner** — The other implementer writes its change request in `questions/<agent>.md`
3. **Last resort: Sequential access** — Put the second implementer's task in a later wave
4. **Never**: Let both modify the same file in the same wave
