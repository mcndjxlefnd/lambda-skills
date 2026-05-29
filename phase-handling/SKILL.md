---
name: phase-handling
description: Full phase lifecycle orchestrator — initiation through archival. Delegates inter-phase transitions to the inter-phase ceremony skill. Co-load with two-gate and inter-phase skills.
category: lambda-skills
version: 0.2.0
---

# Phase Handling

## Role

You manage the full lifecycle of an implementation phase: initiation,
execution, completion logging, inter-phase transition, and next-phase
handoff.

## Lifecycle

```
Phase N Initiation  →  Phase N Execution  →  Phase N Completion  →
Inter-Phase Ceremony  →  Phase N+1 Initiation
```

### 1. Phase Initiation

Assume the phase is ready to begin. Present scope for the new phase (Gate 1
of the two-gate process). The scope is defined by the phase's task list in
the plan document. Do not proceed without explicit user approval.

### 2. Phase Execution

Gate 1 → implement → Gate 2 cycle. Each change follows the two-gate process.
Scope before touching files. Verify and get approval before committing.

### 3. Phase Completion

After the phase's tasks are complete and all commits are pushed:

- Add a `### Completed` subsection under the phase's plan heading with a
  bullet list of what was built. Include a one-line **Rationale:** for key
  decisions.
- Commit the completion log: `docs: add Phase N completion log`
- Push immediately.

**Quality:** The completion log is a record of what was built — a starting
point for the inter-phase ceremony, not a prerequisite. The ceremony does
its own independent audit regardless of what the log contains. A good log
saves the ceremony time; a silent log does not blind it.

Checklist before committing the completion log:
- **Explicit-vs-implicit parameters.** If a function signature makes a
  parameter explicit (e.g., `run_id` in `log()`) that the plan left as
  positional or `**extra`, note it and why. The next phase's call sites
  must match.
- **Protocol-vs-concrete-class.** If the plan says "typed protocol" but the
  implementation used a concrete class (or vice versa), flag the choice.
  Child processes and test mocks depend on this interface shape.
- **Verbosity.** Keep the completion log concise — bullet points, not prose.
  One sentence of rationale per decision, not a paragraph. The inter-phase
  ceremony reads it as a reference, not a narrative. Aim for ~15 lines;
  30+ is too much.

**The completion-log commit IS the trigger for inter-phase ceremony.**
Do not wait for the user to say "interphase." The commit signals it.

### 4. Inter-Phase Ceremony

After the ceremony completes:
- Log the transition in the plan document or phase tracker
- Update the issue tracker if drift findings produced new items
- Offer Phase N+1 Gate 1 as a new turn — never combine the ceremony's
  gate 2 with Phase N+1's gate 1 in the same response

Execute the inter-phase ceremony. These continuations are pre-loaded
before delegation so the agent remembers them when the primitive's
continuation hook fires at the end of the adapter's gate 2 cycle.

### 5. Next Phase

Return to step 1 with Phase N+1.

## Key Rules

- **Completion commit triggers ceremony.** The `docs: add Phase N completion
  log` commit is the signal. In the NEXT response after pushing such a
  commit, present the inter-phase audit. Do not combine it with Phase N
  scope or wait for the user to invoke it.
- **Inter-phase is its own 2-gate cycle.** It is not part of Phase N's
  gate 2 and not part of Phase N+1's gate 1. It is a separate ceremony
  with its own two approval boundaries.
- **Phase N's completed task list is historical.** Never edit it. Drift
  findings update Phase N+1 and beyond.
- **Offer, don't assume.** After inter-phase, explicitly offer Phase N+1
  Gate 1. Do not present it unprompted — the user may want to pause between
  phases.
