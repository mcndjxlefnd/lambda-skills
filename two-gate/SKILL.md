---
name: two-gate
description: Two-gate synchronization — plan gate + commit gate.
---

In every user-facing message about a gate, open with the gate label and
close with its close-tag. Example:

  Gate 1: <plan content here>
  /Gate 1

  Gate 2: <verification summary here>
  /Gate 2

Gate 1 — Permission to implement
  Present a concrete plan: what changes, where, why.
  Keep it simple and direct. If presenting architectural options,
  use concrete language: "Option A: do X. Option B: do Y. A is better because Z."
  No walls of text or abstract architectural nuance.
  Await explicit approval: "yes," "approved," "proceed," "go ahead."
  Discussion, questions, head-nodding ≠ approval.
  No file changes until approved.

Gate 2 — Permission to commit
  Verify: test results, diff summary, lint output.
  Present changes + results + proposed commit message.
  Await explicit approval. No commits until approved.

After Gate 2: execute any post-cycle steps from calling context.