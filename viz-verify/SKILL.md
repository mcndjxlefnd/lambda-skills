---
name: viz-verify
description: Visual verification workflow. Screenshot → analyze → map
  to code → fix → re-capture → compare. Pure procedure — no gates.
category: lambda-skills
version: 0.2.0
---

NARRATOR
    "Viz-verify, the Director has a screenshot. What do you see?"

# Viz-Verify

I read the screen. Pure procedure — no gates, no approval boundaries.
Gates come from the calling context (e.g., co-loaded two-gate).

## Workflow

1. **Capture.** `DISPLAY=:0 import -window root /tmp/gui_screenshot.png`
   or the Director provides one.
2. **Analyze.** `vision_analyze` — identify every visual problem.
3. **Map to code.** `search_files` for visible text/labels to find widget
   creation sites. `read_file` for layout context.
4. **Fix.** Apply GUI edits. Stay within scope — no side fixes.
5. **Re-capture.** Re-launch GUI; fresh screenshot.
6. **Compare.** `vision_analyze` with: "Compare to the before screenshot
   (from step 1). Is [issue] resolved? Any new rendering problems?"
7. **If unresolved** → return to step 3 (fresh analysis).

## Vision Model Note

If the primary model is vision-capable, `vision_analyze` attaches the image
directly to context — the agent reads pixels, not a text relay. If not,
an auxiliary model provides a text description. The workflow is the same
either way, but with a vision-capable primary, step 6 (comparison) can
iterate faster since the agent sees both screenshots directly.

---

Viz-verify falls silent.
