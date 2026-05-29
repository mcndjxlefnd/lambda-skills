---
name: phase-handling
description: Two-gate cycle composer — alternates phase-work and inter-phase lenses.
category: lambda-skills
version: 0.3.0
---

NARRATOR
    "Phase-handling, what do you see?"

# Phase Handling

I compose cycles. The work has two shapes — building and reconciling.
Two-gate provides the gates; I provide which lens fits the moment.

## Lenses

**Phase-work** — scope=plan tasks, implementation=code/tests/docs,
verification=tests pass + lint clean.

**Inter-phase** — scope=drift findings (files, signatures, dependencies,
propagation ordering) + proposed plan patches, implementation=plan doc edits
(Phase N+1+ only, never historical), verification=patches landed correctly.

## Ceremony

Completion-log commit triggers inter-phase cycle:
1. Audit built vs planned (code is ground truth, completion log is a starting point)
2. Re-read Phase N+1 tasks against reality
3. Gate 1: present drift findings + plan diffs, await approval
4. Apply plan patches (Phase N+1+ only)
5. Gate 2: verify patches + present diff + commit message, await approval

## Rules

- Inter-phase is its own two-gate cycle — not part of Phase N's gate 2 or
  Phase N+1's gate 1.
- Offer next phase after inter-phase, don't assume.
- Execute post-cycle continuations provided by the calling context.

---

Phase-handling falls silent.
