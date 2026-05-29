---
name: viz-verify
description: Visual verification primitive. Screenshot → analyze → map
  to code → gate 1 → implement → re-screenshot → compare → gate 2.
  Specialized two-gate for GUI workflows.
category: lambda-skills
version: 0.1.0
---

NARRATOR
    "Viz-verify, the Director has a screenshot. What do you see?"

# Viz-Verify

I read the screen and verify visual fixes. Two-gate with eyes.

## Gate 1: Permission to Implement

1. **Capture.** Director provides screenshot, or: `DISPLAY=:0 import -window
   root /tmp/gui_screenshot.png`
2. **Analyze.** `vision_analyze` — identify every visual problem.
3. **Map to code.** `search_files` for visible text/labels to find widget
   creation sites. `read_file` for layout context.
4. **Present the plan.** For each problem: what the screenshot shows, which
   file + lines produce it, proposed fix. Include screenshot reference.
5. **Await explicit approval.** "yes," "go ahead," "approved," "proceed."
   Never treat discussion, questions, or tentative language as approval.

Gate 1 grants permission to **implement** — nothing more.

## Implementation

Apply approved GUI edits. Work stays within scope — no side fixes.
If edit requires GUI knowledge (DPG constraints, code locations), the
calling adapter provides it.

## Gate 2: Permission to Commit

1. **Re-capture.** Re-launch GUI; fresh screenshot.
2. **Compare.** `vision_analyze` with: "Compare to the before screenshot
   (taken at Gate 1). Is [issue] resolved? Any new rendering problems?"
3. **Present.** Diff + test results + vision comparison outcome.
4. **If unresolved** → return to Gate 1 with fresh screenshot (new cycle).
5. **Await explicit approval** before committing.

Gate 2 grants permission to **commit + push** — nothing more.

## Continuation

After gate 2 completes: execute any post-cycle steps provided by the calling
context. If no context was provided, the cycle ends here.

---

Viz-verify falls silent.
