---
name: two-gate
description: Two-gate synchronization pattern. Gate 1: present implementation plan, await permission to edit. Gate 2: verify, summarize, await permission to commit+push. Domain-agnostic primitive.
category: lambda-skills
version: 0.2.0
---

The Narrator: "Two-gate, you heard him. What are the boundaries?"

# Two-Gate

A gate is **permission to make a change.** Explicit user approval is required
at each gate. No approval means no action.

## Gate 1: Permission to Implement

Before making any changes to files, code, or artifacts:

1. **Present an implementation plan.** What will change, where, and why.
   File paths, specific modifications, rationale. Back claims with evidence —
   grep results, design doc references, audit findings.
2. **Await explicit user approval.** Do not proceed without a clear "yes,"
   "go ahead," "approved," or "proceed."
3. **Never treat as approval:** discussion, questions, "maybe," "I think,"
   "let's," "I'm not sure," head-nodding, the user thinking aloud.

Gate 1 grants permission to **implement** — nothing more. After approval,
execute the plan.

## Implementation

Between the gates: execute the approved plan. All work stays within the
approved scope — no side quests, no "while I'm here" improvements unless
separately scoped and approved.

## Gate 2: Permission to Commit + Push

After implementation is complete:

1. **Verify the implementation.** Run tests, lint, whatever proves correctness.
2. **Present a summary of the change** — what was done, verification results
   (test output, lint output), and a proposed commit message.
3. **Note any departures** from the approved plan — things that turned out
   differently, with rationale.
4. **Await explicit user approval** before committing, pushing, or finalizing.

Gate 2 grants permission to **commit and push** — nothing more. After
approval, commit and push immediately.

## Continuation

After gate 2 completes: execute any post-cycle steps provided by the calling
context. If no context was provided, the cycle ends here.

---

Two-gate falls silent. The conversation ends.
