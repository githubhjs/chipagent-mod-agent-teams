# Run File Templates

Ready-to-use templates for the files teammates and the lead write in `.chipagents/team/<run-id>/`.

## Task Assignment (dispatch prompt)

```
You are {agent}, task {id} of run .chipagents/team/{run-id}/.

Owned files:
- {file1}
- {file2}

Key requirements:
- {requirement1}
- {requirement2}

Interface contract (frozen, do not edit):
- Import {package/types} from {contract file}
- Provide {ports/interface} as listed in contracts/{file}

Acceptance:
- {lint/compile/test criterion}

Write findings/{agent}.md and return the same report.
If blocked, write questions/{agent}.md, set "Status: blocked", and return.
```

## Integration Point Note (in the implementer's findings)

```
## Interfaces provided
{module} side of {interface/port list} is complete in {file}:
- {port group / modport 1}
- {port group / modport 2}
Matches contracts/{file} (checked by elaborating against {package}).
```

## Blocker (questions/<agent>.md)

```
# {agent} — {task id}: blocked

Blocker: {what is preventing progress}
Impact: {what can't be completed until this is resolved}

Options:
1. {option 1}
2. {option 2}

Answer: (lead fills in)
```

## Task Completion Report (findings/<agent>.md)

```
# {agent} — {task id}
Status: done
Summary: {one line}

Changes made:
- {file1}: {what changed}
- {file2}: {what changed} (new file — add to {filelist})

Checks run:
- {lint command}: {clean / N warnings, waived because ...}
- {sim command, seed}: {pass/fail}

Integration notes:
- {interface readiness, assumptions, requests for lead-owned files}
```

## Review Findings Summary (header of a reviewer's findings)

```
# {dimension}-reviewer — {task id}
Status: done
Summary: Review of {target} ({dimension}): Critical {n}, High {n}, Medium {n}, Low {n}

Top finding: {brief description of most important finding}

{findings in the team-reviewer format}
```

## Investigation Report Summary (header of an investigator's findings)

```
# investigator-{n} — {task id}
Status: done
Summary: Hypothesis {id}: {Confirmed | Falsified | Inconclusive}, confidence {High | Medium | Low}

Key evidence:
- {file:line}, {signal}={value} @ {time} (seed {seed}): {what was found}
- {file:line}: {what was found}

{If confirmed}: Recommended fix: {brief fix description}
{If falsified}: Contradicting evidence: {brief description}

{full evidence report}
```

## Change Log Entry (brief.md)

```
- {date time} {Answer to questions/{agent}.md | Contract change | Re-dispatch | chip-swarm run}: {what changed}; affected tasks: {ids}
```
