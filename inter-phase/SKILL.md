---
name: inter-phase
description: Inter-phase ceremony adapter — phase-transition lens on the two-gate synchronization pattern. Co-load with two-gate and phase-handling skills.
category: lambda-skills
version: 0.3.1
---

# Inter-Phase Ceremony

**Role:** Transition between implementation phases by auditing what Phase N
actually delivered against the plan, then updating Phase N+1 tasks for drift.

**Context:** The "plan" is plan diffs — Phase N+1 and beyond only; Phase N's
completed task list is a historical record, never edited.

## Audit Steps

Before Gate 1, gather evidence. The following steps define what to inspect
and what drift looks like in this domain:

1. **Audit what was actually built — do not rely on the completion log.**
   Search for new classes, new files, changed signatures. Compare against
   what the plan said Phase N would deliver. The implementation is ground
   truth. The completion log is a starting point only; if it missed a
   design departure, this independent inspection catches it.
2. **Re-read Phase N+1 tasks against reality.** For each task, ask: does
   this still make sense given what Phase N actually delivered?
3. **Check for common drift patterns:**
   - A file Phase N+1 was supposed to create already exists
   - A function signature differs from what Phase N+1 call sites assume
   - A design decision during Phase N invalidates a Phase N+1 assumption
   - New modules exist that Phase N+1 tasks should touch but don't mention
   - A task is now unnecessary — Phase N subsumed it
   - A task is missing — Phase N created a dependency Phase N+1 must fulfill
   - A task references file paths or module names that shipped differently
   - A task references data fields that don't exist in the actual schema
   - A Phase N+1 task depends on a field/function added in Phase N that isn't
     propagated yet — the propagation task must precede the task that uses it
4. **Patch the plan** to bring Phase N+1 into alignment with the current
   state; confirm N+1 is ready for implementation.

## Domain Lens on Two-Gate

In this domain, "scope" means the drift findings from steps 1-3 and the
proposed plan diffs from step 4. "Implementation" means applying those
plan diffs to the plan document. "Verification" means confirming the
patches landed correctly and the plan document reflects reality.

Execute the two-gate process. After gate 2 completes: execute any
post-cycle steps provided by the calling context.
