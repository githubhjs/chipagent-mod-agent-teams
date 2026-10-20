# Agent Profile Selection Guide

Decision matrix for choosing the right agent profile when dispatching teammates.

## Decision Matrix

```
Does the teammate need a specialized team role?
├── YES → Which role?
│         ├── Planning and synthesis → team-lead (or the main agent itself)
│         ├── Building modules → team-implementer
│         ├── Design review → team-reviewer
│         └── Failure investigation → team-debugger
└── NO → Does it need to modify files?
          ├── YES → default subagent (no profile) with an explicit file list
          └── NO → default subagent told not to edit, or team-reviewer with a research question
```

## Profile Comparison

| Profile                       | Can Read | Can Edit/Create | Can Bash/Git | Writes run files              | Specialized          |
| ----------------------------- | -------- | --------------- | ------------ | ----------------------------- | -------------------- |
| team-lead                     | Yes      | Yes             | Yes          | brief, tasks, contracts, report | Team orchestration |
| team-reviewer                 | Yes      | No              | Yes          | own findings/questions (Bash) | Design review        |
| team-debugger                 | Yes      | No              | Yes          | own findings/questions (Bash) | Failure investigation |
| team-implementer              | Yes      | Yes             | Yes          | own findings/questions        | Feature building     |
| default subagent (no profile) | Yes      | As main agent   | As main agent | as instructed                | No                   |

## Common Mistakes

| Mistake                                       | Why It Fails                         | Correct Choice                         |
| --------------------------------------------- | ------------------------------------ | -------------------------------------- |
| Using `team-reviewer` for implementation      | Cannot edit or create files          | `team-implementer`                     |
| Using `team-debugger` to apply the fix        | Read-only; mixes investigation and fix | `team-implementer` after arbitration |
| Using a default subagent for reviews          | No review structure/checklists       | `team-reviewer`                        |
| Using `team-implementer` for research         | Has edit tools but wrong focus       | Default subagent told not to edit      |

## When to Use Each

### team-lead

- Coordinating multiple teammates (normally the main agent takes this role)
- Decomposing work, writing contracts and `tasks.md`
- Synthesizing results from parallel work into `report.md`

### team-reviewer

- Focused review on one hardware dimension
- Producing structured findings with severity ratings
- Following dimension-specific checklists

### team-debugger

- Investigating one hypothesis about a failure
- Gathering evidence with file:line and signal@time citations
- Reporting confidence levels and causal chains

### team-implementer

- Building modules, packages or testbench components within ownership boundaries
- Following frozen interface contracts
- Reporting interface readiness in its findings file

### Default subagent (no profile)

- One-off tasks that don't fit specialized roles
- Research across code, specs and tool documentation
- Ad-hoc scripting or automation
